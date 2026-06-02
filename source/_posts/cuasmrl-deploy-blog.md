---
title: "cuasmrl 部署实录：用 RL 优化 CUDA Kernel 指令调度"
date: 2026-05-28 18:00:00
tags:
  - Triton
  - CUDA
  - 强化学习
  - 编译器
  - GPU
categories:
  - 编译器
  - HPC
---

# cuasmrl 部署实录：用 RL 优化 CUDA Kernel 指令调度

## 背景

[cuasmrl](https://github.com/hgl71964/cuasmrl) 是一个基于 [Triton](https://github.com/triton-lang/triton) 和 [CuAssembler](https://github.com/hgl71964/CuAssembler) 的研究项目，核心思路是：

> 用强化学习（PPO）对 Triton 编译生成的 SASS 指令进行**指令级重排序**，通过调整 memory instruction 的相对位置来隐藏 latency，提升 kernel 的实际吞吐。

整个 pipeline 大致是：

```
Triton JIT 编译 → cubin → CuAssembler 反汇编为 SASS
    → 静态分析 (stall count / 依赖图) → RL agent 重排指令
    → CuAssembler 重新汇编为 cubin → 加载执行 → 测量 TFLOPS → reward
```

最近在一台双卡服务器上成功复现了这套流程，记录一下踩过的坑。

<!-- more -->

## 环境

| 组件 | 版本 |
|------|------|
| Python | 3.12.3 |
| PyTorch | 2.11.0+cu130 |
| Triton | 2.1.0 (从源码编译) |
| LLVM | 49af6502 (Triton 预编译 binary) |
| CUDA Driver | 545.23.08 |
| ptxas | 12.3.52 |

### 硬件

| GPU | Compute Capability | 状态 |
|-----|-------------------|------|
| NVIDIA GeForce RTX 3080 (Ampere) | 8.6 | ✅ 完全支持 |
| NVIDIA GeForce RTX 5060 Ti (Blackwell) | 12.0 | ❌ ptxas 12.3 不支持 sm_120 |

Blackwell 架构太新，Triton 2.1 自带的 ptxas 12.3 不认识 `sm_120`，所以后续所有实验都在 RTX 3080 上跑。

## 部署步骤

### 1. 激活环境 & 安装构建依赖

```bash
source ~/venv/torch/bin/activate
uv pip install ninja cmake wheel pyelftools tensorboard
```

### 2. 克隆 CuAssembler

```bash
git clone https://github.com/hgl71964/CuAssembler.git
export PYTHONPATH=/path/to/CuAssembler:/path/to/CuAssembler/bin:/path/to/CuAssembler/CuAsm:$PYTHONPATH
```

### 3. 安装 cuasmrl

```bash
uv pip install -e cuasmrl/cuasmrl
```

### 4. 从源码编译 Triton

```bash
cd cuasmrl/python
uv pip install --no-build-isolation -e .
```

编译过程会自动下载 LLVM 预编译 binary（约几百 MB）和 pybind11，用 Ninja 构建。整个过程大约 3 分钟。

### 5. 安装 RL 依赖

```bash
uv pip install gymnasium==0.29.1 numpy sympy
```

## 踩坑记录

### 坑 1：googletest 网络问题

```
fatal: unable to access 'https://github.com/google/googletest.git/'
gnutls_handshake() failed: The TLS connection was non-properly terminated.
```

CMake 在 build 阶段通过 `FetchContent` 拉取 googletest，但机器网络连 GitHub 不稳定。解法：直接注释掉主 `CMakeLists.txt` 中的 unittest 子目录：

```cmake
# add_subdirectory(unittest)
```

### 坑 2：Compute Capability 8.6 部分缺失

RTX 3080 是 CC 8.6（Ampere），cuasmrl 的 `gpu_utils.py` 中某些函数支持了 8.6，但 `get_st_database()` 和 `has_hazard()` 漏掉了。补上即可：

```python
elif cc == (8, 6):
    # RTX 3080 uses same database as A100 (8, 0)
    return {
        'IADD3': 4, 'IMAD.IADD': 4, 'IADD3.X': 4, ...
    }
```

### 坑 3：PyTorch 2.11 与 Triton 2.1 的接口不兼容

PyTorch 2.11 的 inductor 期望从 `triton.backends.compiler` 导入 `AttrsDescriptor` 和 `GPUTarget`，但这是 Triton 3.x 才有的模块结构。Triton 2.1 的编译器模块在 `triton.compiler.compiler`。

解法：创建一个 compat shim：

```
site-packages/triton/backends/
├── __init__.py      # 暴露 backends = {"cuda": nvidia}
├── compiler.py      # 提供 GPUTarget dataclass 和 AttrsDescriptor
└── nvidia.py        # 空模块，占位
```

### 坑 4：CuAssembler 的 `eval()` 缺少符号

CuAssembler 用 `eval()` 加载预编译的指令知识库（`.txt` 文件），里面包含 `Matrix`（sympy）、`CuInsAssemblerRepos`、`CuSMVersion` 等符号。如果这些不在 eval 的 globals 里，就会报错：

```
Assemble failed: name 'Matrix' is not defined
```

修复 `CuInsAssemblerRepos.py` 中的 `initFromFile`：

```python
def initFromFile(self, fname):
    with open(fname, 'r') as fin:
        fconts = fin.read()
        g = {
            '__builtins__': __builtins__,
            'Matrix': Matrix,
            'CuInsAssemblerRepos': CuInsAssemblerRepos,
            'CuInsAssembler': CuInsAssembler,
            'CuSMVersion': CuSMVersion,
        }
        asm_repos = eval(fconts, g)
```

### 坑 5：gymnasium 版本

项目 requirement 里指定的是 `gymnasium==0.29.1`。用最新版（1.3.0）会导致 wrapper 链不兼容。

## 验证结果

### 基础 Triton kernel

```
$ python 01-vector-add.py
The maximum difference between torch and triton is 0.0
vector-add-performance:
           size      Triton       Torch
15  134217728.0  691.064996  690.761495
```

Triton 和 PyTorch 结果完全一致。

### RL 优化 Matrix Multiplication

运行 `03-matrix-multiplication.py`（512×512×2048, leaky_relu fused）：

```
[INIT] dims: 29; total kernel lineno: 624; mem_loc: 185; max_src_len: 6;
[ENV_LOOP] WorkDir: data/NVIDIA_GeForce_RTX_3080/...mm_leakyRelu/512_512_2048

[RESET] init perf: 36.51 TFLOPS
global_step=32,  episodic_return=[0.16]
global_step=64,  episodic_return=[0.05]
global_step=96,  episodic_return=[1.06]  ← 找到正向收益配置
global_step=128, episodic_return=[0.86]
global_step=160, episodic_return=[0.81]
global_step=224, episodic_return=[1.01]
```

关键指标：

- **Baseline**: ~36.5 TFLOPS（Triton 默认编译结果）
- **RL Agent**: 通过 PPO 探索指令重排策略，约 100 step 后找到 ~1% 收益的配置
- **可重装配**: CuAssembler 成功将变异后的 SASS 重新组装为 cubin 并加载执行，整个闭环跑得通

## 工作流程示意

```
┌─────────────────────────────────────────────────────┐
│  Triton JIT Compiler                                │
│  @triton.jit → PTX → ptxas → cubin                 │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  CuAssembler                                        │
│  cubin → nvdisasm → SASS text                       │
│  解析 ctrl code, predicate, opcode, dst, src         │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Static Analysis                                    │
│  构建依赖图, 分析 stall count                        │
│  识别可移动的 memory ops (LDGSTS, LDG, LDS, STG)    │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  PPO Agent                                          │
│  Observation: SASS embedding (ctrl+opcode+regs)     │
│  Action: 交换相邻指令的 swap up/down                 │
│  Reward: ΔTFLOPS (性能提升量)                       │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  CuAssembler (re-assemble)                          │
│  修改后 SASS → cubin → CUDA Driver 加载 → 执行      │
│  测量新的 TFLOPS → 返回 reward                      │
└─────────────────────────────────────────────────────┘
```

## 复盘：为什么 RL 在这个问题上能 work

跑通流程之后回头看，这个项目之所以能让 RL 有效优化，核心不是"RL 很强"，而是**问题的结构恰好适合 RL**。拆解一下：

### 条件 1：动作空间是离散且局部的

```python
# backend.py:84
self.action_space = Discrete(n=dims * 2 + 1)  # 每条 mem op: swap up / swap down / noop
```

只做相邻 swap，不做跨行重排。这把搜索空间从 `O(n!)` 压到 `O(n×2)`。如果允许任意重排，RL 根本不可能收敛。

### 条件 2：Action Mask 把非法空间剪枝到几乎为零

```python
# sample.py:229-429 — _generate_mask() 检查 5 层约束
mask = [1, 1]
# 1. 标签边界 — 不能跨越 label
# 2. 数据依赖 (RAW/WAR) — 有依赖的指令对不能交换
# 3. 相邻指令约束 — check_adj_opcodes() 编码了数百行 GPU 微架构经验规则
# 4. Scoreboard 冲突 — 读/写 scoreboard 冲突时禁止交换
# 5. Stall count 窗口 — 在 10 条指令窗口内检查 stall 是否足够
```

没有 mask，RL 会不断产生 segfault 或错误结果，根本学不到东西。**mask 是整个项目能 work 的前提。** 实测中，每一步的有效 action 通常只有 3~5 个（总 action space 可能有几百），mask 剪枝了 95%+ 的搜索空间。

### 条件 3：正确性可以高效验证

```python
# verify.py — 生成随机输入，对比输出
test_ok = self.eng.test_fn(cubin, n_tests, n_tests, False)
```

每一步变异后，跑一次 kernel 对比结果就能判定对错。这是免费的 correctness oracle。segfault 直接给 reward=-1，测试失败给 reward=-5，RL 可以快速学到"哪些操作是危险的"。

### 条件 4：奖励信号是密集且直接的

```python
# backend.py:207
reward = (perf - self.last_perf) / self.init_perf * 100
```

每一步 swap 都能立即测量 ΔTFLOPS。不需要等整个 episode 结束。这比 sparse reward 的 RL 问题容易太多——agent 每走一步都能得到反馈，知道"这个 swap 是好是坏"。

### 条件 5：状态表示是结构化的

SASS 指令天然有固定的结构：ctrl code（barrier/stall/yield）、predicate、opcode、dst、src。可以直接编码为固定长度的特征向量堆叠成矩阵，不需要从非结构化数据中提取特征。这让一个简单的 Conv2D 网络就能处理。

### 条件 6：优化空间有局部性

好的指令布局通常是"当前布局的微调"，而不是"完全不同的排列"。相邻 swap 恰好利用了这个局部性——每一步只改变一点点，RL 的 incremental exploration 天然适合这种 landscape。从实验看，~100 步就能收敛到 ~1% 的提升，说明 reward landscape 是相对平滑的。

### Checklist 总结

| 条件 | 本项目 | 为什么重要 |
|------|--------|-----------|
| 离散动作空间 | swap up/down | 连续空间需要完全不同的算法 |
| 可行空间远小于总空间 | action mask 剪枝 95%+ | 无 mask 则探索效率极低，大部分时间浪费在 segfault 上 |
| 正确性可高效验证 | 跑 kernel 对比输出 | 无法验证则无法学习，agent 不知道什么是对的 |
| 密集奖励 | 每步 ΔTFLOPS | sparse reward 很难收敛，需要很多步才能得到第一个正反馈 |
| 结构化状态 | SASS 指令天然结构化 | 非结构化数据需要额外的表示学习，增加复杂度 |
| 局部最优有梯度 | 相邻 swap 的性能变化平滑 | 如果 landscape 是随机的，RL 没有探索方向 |

这 6 个条件缺一不可。缺了任何一个，这个项目的 RL 优化都不太可能 work。反过来说，如果要在 GPU 编译优化的其他方向复制这个成功，也应该先检查这些条件是否满足。

## 总结

cuasmrl 展示了一条有趣的优化路径：在编译器生成的机器码层面，用 RL 做局部搜索。虽然单 kernel 的提升幅度不大（~1%），但这种方法不需要手写 heuristic，而是让 agent 自己发现对特定 GPU 微架构最优的指令布局。

对于 Ampere（8.6）架构，整套流程已经跑通。Blackwell（12.0）等更新架构需要等待 CUDA Toolkit 和 Triton 的更新支持。
