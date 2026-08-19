---
title: 纸上得来终觉浅，绝知此事要躬行——llama.cpp 部署调优实录
date: 2026-08-19 23:30:00
tags: [llama.cpp, LLM, 部署, 调优, KV-Cache, Flash-Attention]
categories: 技术实践
---

> 纸上得来终觉浅，绝知此事要躬行。
> ——陆游《冬夜读书示子聿》

> 没有调查，就没有发言权。
> ——毛泽东《反对本本主义》

---

## 一、背景

我们用 2 张 RTX 3080 20G 跑 Qwen3.8-27B-GGUF（Q4_K_M 量化），Tensor Parallel 模式（`-sm tensor`），给 agent 提供推理服务。

网上关于 llama.cpp 的调优文章汗牛充栋，KV Cache 量化、Flash Attention、多 slot 并发——每个参数都被人写过"最佳实践"。但**别人的最佳实践，放到你的硬件、你的模型、你的 workload 上，是不是还是最佳？** 不试一下，谁也不知道。

于是我们决定：**逐个参数实测，用数据说话。**

## 二、初始配置：裸跑

最开始的启动脚本很简单：

```bash
llama-server \
  -m Qwen3.8-27B-Q4_K_M.gguf \
  -ngl 99 -sm tensor \
  -c 262144 --parallel 1 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  --host 0.0.0.0 --port 8080
```

- 没开 Flash Attention（`-fa` 默认 auto）
- 没开 KV Cache 量化（`-ctk/-ctv` 默认 f16）
- 单 slot（`--parallel 1`）

**实测数据：**
- Prefill：~960 tok/s（大 prompt）
- Generation：~60-70 t/s
- 显存：17.3G / 20G（GPU 0），15.8G / 20G（GPU 1）

看起来还行？但问题出在多 agent 并发时——单 slot 意味着所有请求排队，一个 agent 在思考，其他 agent 只能干等。

## 三、第一刀：开 Flash Attention

网上都说 Flash Attention "无脑开就好"。真的吗？

```bash
llama-server ... -fa on
```

**实测数据（单 slot）：**
- Prefill：~967 tok/s（基本持平）
- Generation：~60 t/s（基本持平）

**结论：Flash Attention 在 RTX 3080 上对 prefill 和 generation 速度几乎没有提升。**

为什么？因为 Flash Attention 的核心优势是**降低 peak memory**，而不是加速计算。在 20G 显存的卡上跑 Q4 量化模型，KV Cache 本身就不大（f16 下 262K context 约 2-4G），FA 省下来的那点内存，并没有转化为速度提升。

**真正的价值：** 如果你要跑更长的 context（比如512K+），FA 能防止 OOM。但在我们的场景下，它更像是一个"保险"，而不是"加速"。

## 四、第二刀：KV Cache 量化

这是最核心的调优。f16 的 KV Cache 每个元素 2 字节，Q8 量化后只要 1 字节，理论上 KV Cache 显存占用减半。

### 4.1 尝试 `-ctk q8_0 -ctv q8_0`

```bash
llama-server ... -fa on -ctk q8_0 -ctv q8_0
```

**成功启动。** 实测：
- 显存占用变化不大（因为262K context 的 KV Cache 本身就不大）
- Cache 命中率正常（90%+）
- 无异常换入换出

### 4.2 尝试 `-ctv q4_0`（更激进的量化）

理论上 q4 比 q8 再省一半，我们试了：

```bash
llama-server ... -ctk q8_0 -ctv q4_0
```

**崩了。** 报错：

```
GGML_ASSERT(ret.axis != GGML_BACKEND_SPLIT_AXIS_UNKNOWN) failed
```

原因：**q4_0 量化格式不支持 Tensor Parallel 的跨 GPU 分片。** GGML 后端无法将 q4 的 KV tensor 拆分到两张卡上。

### 4.3 尝试 `-ctv q4_1`

```bash
llama-server ... -ctk q8_0 -ctv q4_1
```

**也崩了。** 同样的错误。q4_1 也不支持 TP 分片。

**结论：在 `-sm tensor` 模式下，KV Cache 量化的最低安全选项是 `q8_0`。q4_0/q4_1 虽然省内存，但和 TP 不兼容。**

这是一个典型的"纸上得来"的坑——很多文章推荐 q4 量化 KV Cache，但它们跑的是单卡模式。**多卡 TP 下，量化选项受限。**

## 五、第三刀：多 Slot 并发

单 slot 排队太慢，我们加了 `--parallel 4`：

```bash
llama-server ... --parallel 4 --kv-unified
```

**问题出现了：** 两个 agent 同时发大 prompt（109K + 76K tokens），4 个 slot 的统一 KV Cache 开始频繁淘汰：

```
W srv alloc: making room for prompt cache entry, removing oldest entry (size = 599 MiB)
W srv alloc: making room for prompt cache entry, removing oldest entry (size = 2119 MiB)
```

Cache 命中率暴跌到 **0%**——每次请求都要重新 prefill 全部 tokens，因为之前的缓存被挤掉了。

**这是"没有调查就没有发言权"的经典案例：** 文档说 `--parallel` 是"总 context 除以 slot 数"，但没人告诉你：如果两个 agent 同时发大 prompt，总 KV Cache 不够用时会发生什么。答案是：**互相挤兑，全部重算。**

### 解决方案

降为 `--parallel 2`，减少并发 slot 数，给每个 agent 留足 KV Cache 空间：

```bash
llama-server ... --parallel 2 --kv-unified \
  -ctk q8_0 -ctv q8_0 -fa on
```

**实测数据：**
- Cache 命中率：**98-99%** ✅
- Cache 淘汰：**0 次** ✅
- Generation：~61 t/s（单 slot 独占时）
- 两个 slot 同时活跃时：~35 t/s（合理，显存带宽被瓜分）

## 六、最终配置与经验总结

### 最终启动脚本

```bash
llama-server \
  -m Qwen3.8-27B-Q4_K_M.gguf \
  --mmproj mmproj-BF16.gguf \
  -ngl 99 -sm tensor \
  -c 262144 --parallel 2 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  -fa on \
  -ctk q8_0 -ctv q8_0 \
  --image-min-tokens 1024 \
  --kv-unified \
  --host 0.0.0.0 --port 8080
```

### 参数调优对照表

| 参数 | 我们的尝试 | 结论 |
|------|-----------|------|
| `-fa on` | ✅ 开启 | 对速度无明显提升，但防止长 context OOM，建议开着 |
| `-ctk q8_0` | ✅ 开启 | KV Cache K 用 Q8，省 50% 显存，推荐 |
| `-ctv q8_0` | ✅ 开启 | KV Cache V 用 Q8，和 TP 兼容的最低选项 |
| `-ctv q4_0/q4_1` | ❌ 崩溃 | 与 `-sm tensor` 不兼容，多卡勿用 |
| `--parallel 4` | ⚠️ 挤兑 | 4 slot 在大 prompt 下互相淘汰，命中率暴跌 |
| `--parallel 2` | ✅ 稳定 | 2 slot 是我们的最佳平衡点 |
| `--kv-unified` | ✅ 开启 | 统一 KV Cache 池，比 per-slot 更灵活 |

### 踩坑记录

1. **`-fa` 不需要显式 `on`？** 错。`-fa` 单独用会把下一个参数当值解析（`-fa -ctk` 会把 `-ctk` 当成 flash-attn 的值），必须写 `-fa on`。

2. **`--prompt-cache-all` 不存在？** 对。这个参数在某些文章里出现过，但我们的 build 版本没有。实际的 prompt caching 是 `--cache-prompt`，默认就是开启的。

3. **q4 量化 KV Cache 能省一半显存？** 在单卡上可以。在多卡 TP 模式下不行，GGML 后端不支持对 q4 tensor 做跨 GPU 分片。

4. **多 slot 就是好？** 不一定。slot 越多，KV Cache 池被瓜分越厉害。大 prompt 场景下，少 slot 反而更稳。

## 七、方法论：没有调查就没有发言权

这次调优最大的收获不是参数配置，而是方法论：

**1. 不要迷信"最佳实践"**

每篇文章都说"开 Flash Attention"、"用 Q4 量化 KV Cache"。但在我们的硬件（2x RTX 3080）和模式（TP）下，这些"最佳实践"要么无效，要么直接崩溃。**别人的结论，要在你自己的环境里验证。**

**2. 用数据说话，不要靠感觉**

"感觉"开了 FA 会快一点，"感觉"q4 比 q8 好。但实测数据告诉你：FA 对速度几乎没影响，q4 在 TP 下根本跑不了。**没有数据支撑的优化，就是自欺欺人。**

**3. 监控比调参更重要**

调完参数不是结束，而是开始。我们用 `/slots` API 监控 cache 命中率、用 `nvidia-smi` 看显存、用 log 看淘汰事件。**不监控的优化，等于盲人摸象。**

**4. 逐步调整，不要一次改太多**

我们先加 FA，测一轮；再加 Q8，测一轮；再改 parallel，测一轮。如果一次改三个参数，出了问题你都不知道是哪个参数的锅。**控制变量，逐步推进。**

## 八、结语

> 纸上得来终觉浅，绝知此事要躬行。

写代码如此，调参数亦如此。llama.cpp 的每个参数背后都有复杂的行为，文档和文章只能给你一个起点，**真正的理解必须来自你自己的实测。**

这次调优从"裸跑"到"FA + Q8 + 2 slot"，每一步都踩过坑、看过数据、做过判断。最终的配置不一定是最优的，但它是我们**调查过、验证过、理解过**的。

**这，才是"躬行"的意义。**

---

*本文记录了 2026 年 8 月在 2x RTX 3080 20G 上部署 Qwen3.8-27B 的 llama.cpp 调优过程。硬件、模型、workload 不同，结论可能不同。请以实测为准。*
