---
title: "当 AI Agent 碰上 LLVM 后端：一场关于 GPU 指令调度的诚实实验"
date: 2026-05-29 20:00:00
tags:
  - GPU
  - LLVM
  - 编译器优化
  - AMD RDNA3
  - AI for Code
  - 指令调度
categories:
  - 系统研究
---

> 我们用 7 种 AI 调度 Agent 操控了 LLVM-AMDGPU 后端的 64 个指令调度决策点，产生了数百行不同的汇编代码——然后发现性能几乎没有变化。这是一个关于"编译器到底有多强"的诚实故事。

<!--more-->

## 背景：一个诱人的想法

现代 LLM 能写代码、能推理、能做数学。那它能不能**替代编译器的硬编码启发式规则**？

Google 在 2022 年的 MLGO 论文中证明：用 RL 替换 LLVM 的 x86 inlining 和 regalloc 启发式是可行的。但 **AMD GPU 后端的 ML-guided 优化是一片完全空白的领域**。

我们的目标：用 AI Agent 替代 LLVM-AMDGPU 后端中 `GCNSchedStrategy::pickNode()` 的启发式规则，让 Agent 决定每条指令的调度顺序。

## 实验装置

### 工具链

```
HIP 源码 → clang (device IR) → llc -misched=agent-sched → Python Agent → llc -agent-sched-decisions → GPU 二进制
```

我们在 LLVM 的 AMDGPU 后端中找到了一个已经存在的 `AgentSchedStrategy` pass（在 `llvm-project-rocm/` fork 中），它完美地暴露了我们需要的接口：

- `-agent-sched-state-log <file>`：在每个 `pickNode()` 决策点导出候选指令列表
- `-agent-sched-decisions <file>`：从文件读取预计算的调度决策

这让我们可以在 **完全不修改 LLVM 源码** 的情况下，在 Python 侧闭环控制编译器的指令调度。

### Agent 设计

| Agent | 策略 | 实现 |
|-------|------|------|
| **default** | LLVM 默认启发式（GCNMaxOccupancy） | 对照组 |
| **random** | 随机选择候选指令 | 验证管道正确性 |
| **min_pressure** | 优先选 VGPR writes 最少的指令 | 降低寄存器压力 |
| **max_latency** | 优先选延迟最高的指令 | 隐藏延迟 |
| **pref_load** | 优先选内存加载指令 | 隐藏访存延迟 |
| **pref_salu** | 优先选标量指令 | 释放 VGPR |
| **LLM (DeepSeek)** | 用大模型分析候选列表做决策 | 核心验证点 |

## 实验一：Fused MLP（计算密集型）

### Kernel

手写 RDNA3 优化的 Fused MLP：`GEMM + Bias + SiLU`，64×64 tile，4×4 register tiling。

### 结果

| Agent | 决策变更率 | VGPR | 时间 (us) | vs Default |
|-------|-----------|------|-----------|------------|
| default | 0/64 (0%) | 74 | 311.2 | — |
| random | 62.5% | 74 | 311.4 | +0.06% |
| min_pressure | 70.3% | 74 | 312.9 | +0.55% |
| max_latency | 90.6% | 74 | 313.8 | +0.84% |
| pref_load | 84.4% | 74 | 311.9 | +0.22% |
| pref_salu | 73.4% | 74 | 310.9 | -0.10% |
| LLM | 60.9% | 74 | 312.4 | +0.39% |

**关键观察**：所有 Agent 产生了不同的汇编代码（最多 512 行 diff），但 VGPR 分配完全相同（74），性能差异 <1%。

### 为什么？

1. **74 VGPRs → 16 waves/SIMD → 100% occupancy**。已经到达硬件上限，没有优化空间。
2. **Compute-bound**：瓶颈是 FMA 吞吐量，不是指令调度。
3. **Greedy Allocator 太强**：不同指令顺序产生相同的寄存器分配。

## 实验二：FlashAttention Decode（访存密集型）

### 动机

fused_mlp 的 74 VGPRs 太"干净"了。我们需要更复杂的 kernel。

### Kernel 设计

手写 HIP 版 FlashAttention decode，使用 **4 个独立 attention head** 强制编译器保留 4× 的累加器状态：

```cpp
// 4 个独立累加器 —— 编译器不能重用寄存器
float a0[32]={}, a1[32]={}, a2[32]={}, a3[32]={};
float m0=-1e30f, m1=-1e30f, m2=-1e30f, m3=-1e30f;
float l0=0, l1=0, l2=0, l3=0;

// 每个 head 独立计算 softmax
for (int sn = 0; sn < N; sn += BLOCK_N) {
    // ... 加载 K, V tiles ...
    float s0[BLOCK_N], s1[BLOCK_N], s2[BLOCK_N], s3[BLOCK_N];
    // ... 4 组独立的 score → softmax → accumulate ...
}
```

### 结果（原始版本，97 VGPRs）

| Agent | 决策变更率 | VGPR | 时间 (us) | vs Default |
|-------|-----------|------|-----------|------------|
| default | 0/26 (0%) | 97 | 92,249 | — |
| random | 53.8% | 97 | 95,300 | **+3.3%** |
| min_pressure | 30.8% | 97 | 91,161 | **-1.2%** |
| max_latency | 46.2% | 97 | 94,787 | +2.8% |
| pref_load | 46.2% | 97 | 92,216 | 0.0% |
| pref_salu | 46.2% | 97 | 94,911 | +2.9% |

**这次有差异了！** min_pressure 比默认快 1.2%，random 慢 3.3%，总差异 ~4.5%。

### 为什么 FA decode 有效而 fused_mlp 无效？

**因为 FA decode 是 memory-bound 的。**

fused_mlp 的瓶颈是 FMA 吞吐量——不管怎么重排指令，FMA 单元都在满负荷运转。

FA decode 的瓶颈是 Global Memory 延迟——每轮循环都要等 K/V 从显存加载回来。此时指令调度可以影响"在等待期间发射哪些 ALU 指令"，从而影响 pipeline 利用率。

## 实验三：waves_per_eu(1,1)——释放编译器的束缚

### 方法

```cpp
__attribute__((amdgpu_waves_per_eu(1, 1)))  // 目标：最低 occupancy
__global__ void fa_decode_max_pressure(...)
```

这个属性告诉编译器："不需要为了高 occupancy 而压缩寄存器"。

### 结果

| 配置 | VGPR | Occupancy | 调度影响 |
|------|------|-----------|---------|
| 原始 FA decode | 97 | 100% (16 waves) | ~4.5% |
| waves_per_eu(1,1) | 144 | 50% (8 waves) | ~3.4% |
| 目标 | >256 | <37.5% | — |

**VGPR 从 97 → 144（+48%），但调度影响反而从 4.5% 降到了 3.4%。**

### 为什么 occupancy 降低反而调度影响变小？

这是一个精彩的硬件行为：

当 Occupancy 从 16 waves 降到 8 waves 时，硬件通过多 Wave 隐藏延迟的能力下降了。理论上这应该给指令调度留出更大空间，但实际上：

**FA decode 是 memory-bound 的。** Occupancy 减半 → 活跃的访存请求并发度腰斩 → 整个流水线从"跟 VALU 抢延迟的复杂交错状态"退化成了"单纯死等 Global Memory 回传"的饥饿状态。

此时不管 Agent 在 Ready Queue 里怎么精妙地重排那几条 VALU 指令，都无法拯救因为总带宽吞吐下滑带来的大片气泡。

## 尝试引爆"寄存器核弹"——全部失败

我们尝试了多种方式将 VGPR 推到 256+，观察 occupancy 崩溃：

| 方法 | 结果 | 原因 |
|------|------|------|
| BLOCK_D=256 的 FlashAttention | 97 VGPRs | 编译器循环内重用寄存器 |
| 4 个独立 attention head | 144 VGPRs | 编译器仍然能优化 |
| 8×8 register tiling GEMM | 87 VGPRs | 内层循环寄存器可复用 |
| `volatile` 阻止重用 | 97 VGPRs | volatile 帮助编译器优化 |
| `-amdgpu-fixed-num-vgpr=256` | 不存在 | LLVM 没有这个 flag |
| MIR dependency injection | 编译失败 | MIR 格式验证太严格 |
| **waves_per_eu(1,1)** | **144 VGPRs** | **唯一有效方法** |

### MIR 注入踩坑：RDNA3 的 Wave32 红线

我们尝试在 MIR 中注入 COPY 指令扩展寄存器 live range，但遇到了：

```
error: missing implicit register operand 'implicit $vcc'
```

这是 RDNA3 Wave32 架构的底层约束：在 Wave64（CDNA）上，条件寄存器 VCC 是 64 位的；在 Wave32（RDNA）上，VCC 被物理切分为 `$vcc_lo` 和 `$vcc_hi`。LLVM 的 Machine Verifier 严格检查虚拟寄存器类的合法性——手工注入的指令一旦破坏了硬件指令的隐式依赖，LLVM 宁可 crash 也绝不生成错误机器码。

## 核心结论

### 结论一：Pre-RA 指令调度对 GPU 性能影响有限

在我们的实验矩阵中：
- Compute-bound kernel（fused_mlp）：调度影响 <1%
- Memory-bound kernel（FA decode）：调度影响 ~4.5%
- 降低 occupancy 后：调度影响 ~3.4%

**LLVM 的 GCNSchedStrategy 已经做得足够好。** AI Agent 在 Pre-RA 阶段的搜索空间里，找不到显著超越默认启发式的解。

### 结论二：VGPR 由算法决定，不由调度决定

编译器的 Greedy Allocator 通过 spilling 和 live range splitting，将 VGPR 数量控制在算法数据依赖的最小值。即使 `waves_per_eu(1,1)` 放松了约束，编译器也只用 144 VGPRs——因为它不需要更多。

### 结论三：真正的战场不在 Pre-RA

通过这组实验，我们用诚实的工程闭环探明了 AMD GPU 优化的深水区。**不要在 Pre-RA 的搜索空间里浪费算力。** 真正的 Agent 干预战场在别处。

## 下一步：两个真正的突破口

### 方向一：算法层面的超级算子融合

既然编译器不会主动制造 256+ VGPR，那就**从算法层面逼迫它**：

把整个 Transformer Layer（Attention + FFN + LayerNorm + Top-K 路由）全部写进同一个 Triton kernel。当算法在数学上需要数千个同时活跃的变量时，编译器别无选择——VGPR 必然爆炸，Occupancy 必然崩溃。

此时 Agent 的价值是：**搜索最优的超级融合策略**——哪些操作融合在一起、以什么顺序执行、中间结果如何在寄存器和 LDS 之间流转。

### 方向二：Post-RA 机器码重排

彻底放弃 Pre-RA 阶段。既然寄存器已经由 Greedy 完美分好了（无论是 74 还是 144），就接受这个结果。

让 Agent 在 **Post-RA 阶段**，利用 RDNA3 独有的：
- **VOPD（双发射）**：一条指令同时执行两个不同的 VALU 操作
- **s_delay_alu**：精确控制 ALU 流水线的延迟
- **Waitcnt 优化**：精确控制 memory/ALU/barrier 的等待点

在物理寄存器确定的绝对安全世界里，极限对齐指令、压榨硬件的单核吞吐量。

## 复现

所有代码和数据在 `neuro-compiler-agent/mvp/`：

```bash
cd neuro-compiler-agent/mvp

# 提取调度状态
llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx1100 \
  -misched=agent-sched -agent-sched-state-log=sched_states.log \
  fused_mlp_v3_device.ll -o /dev/null

# 运行 Agent
python3 agents.py sched_states.log -a min_pressure -o decisions.txt

# 用 Agent 决策重编译
llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx1100 \
  -misched=agent-sched -agent-sched-decisions=decisions.txt \
  -filetype=obj fused_mlp_v3_device.ll -o device.o

# 链接并运行
ld.lld -shared device.o -o device.co
./fa_runner device.co 32 128 2048
```

## 致谢

- LLVM AMDGPU 团队的 `AgentSchedStrategy` 框架
- DeepSeek API 提供 LLM 调度决策
- AMD ROCm 7.2 工具链

---


