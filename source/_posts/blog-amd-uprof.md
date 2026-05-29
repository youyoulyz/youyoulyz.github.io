---
title: AMD uProf 性能分析：矩阵乘法优化之旅
date: 2026-05-15 12:00:00
tags:
  - 性能分析
  - AMD
  - Profiling
  - 缓存优化
categories:
  - 系统优化
---

## 背景

CS210 课程的第二次作业要求使用性能分析工具对矩阵乘法程序进行 profiling。由于使用的是 AMD CPU，我选择了 AMD uProf 的 `data_access` 模式来分析 L1/DTLB 性能变化。

## 环境

- **CPU**: AMD (Family 0x17, Model 0x71, 24 核)
- **OS**: Linux Ubuntu 25.04, Kernel 6.14.0-37-generic
- **编译器**: GCC 14.2.0, `-g -O3 -march=native`
- **分析工具**: AMD uProf 5.3.518 (`data_access` config)
- **矩阵大小**: 2048×2048

## 测试用例

使用 Intel oneAPI 的 `matrix_multiply_c` 示例，包含 5 个优化版本：

| 版本 | 函数 | 优化策略 |
|------|------|----------|
| 0 | multiply0 | 基础串行 (i-j-k 循环顺序) |
| 1 | multiply1 | 多线程 (pthreads, 同 i-j-k 顺序) |
| 2 | multiply2 | 循环交换 (i-k-j 顺序) |
| 3 | multiply3 | 循环交换 + 向量化提示 (`#pragma ivdep`) |
| 4 | multiply4 | Cache blocking (64 元素块) + 循环展开 |

## 编译与分析

每个版本独立编译并使用 AMDuProfCLI 进行数据访问分析：

```bash
# 编译
gcc -g -O3 -march=native -DUSE_THR -c src/multiply.c -D_LINUX
gcc -g -O3 -march=native -DUSE_THR src/*.c -o matrix -lpthread -lm

# 使用 data_access 模式进行 profiling
AMDuProfCLI collect --config data_access -o profile_multiply0/multiply0-da ./matrix

# 生成 CSV 报告
AMDuProfCLI report -i profile_multiply0/multiply0-da \
  --report-output profile_multiply0/multiply0-da-report.csv
```

## 结果

### 性能对比

| 版本 | 执行时间 (s) | MFLOPS | 加速比 |
|------|-------------|--------|--------|
| multiply0 | 127.78 | 134 | 1× |
| multiply1 | 6.89 | 2,495 | 18.5× |
| multiply2 | 0.77 | 22,320 | 166× |
| multiply3 | 0.73 | 23,605 | 175× |
| multiply4 | 0.15 | 112,524 | **836×** |

### L1 数据缓存分析

| 版本 | CPI | L1 DC Miss Ratio | L1 DTLB Miss Rate |
|------|-----|------------------|-------------------|
| multiply0 | 6.26 | 16.9% | 0.106 |
| multiply1 | 5.32 | 26.5% | 0.112 |
| multiply2 | 3.13 | 26.5% | 0.001 |
| multiply3 | 3.08 | 24.6% | 0.001 |
| multiply4 | **0.82** | 23.3% | 0.019 |

### 关键发现

**1. 循环交换是最关键的优化**

原始代码中，内层循环访问 `b[k][j]`，其中 `k` 变化时会跨越整行内存（步长为 2048 个 double = 16KB）。这导致：
- 大量 L1 缓存未命中
- 高 DTLB 未命中率（跨页访问）

循环交换为 i-k-j 顺序后，`b[k][j]` 变为连续内存访问，DTLB 未命中率从 0.106 降至 0.001。

**2. Cache Blocking 在大矩阵下效果显著**

当矩阵大小为 2048×2048 时（每个矩阵 32MB，共 96MB），工作集远超 L1 缓存但可放入 L2。Cache blocking 将计算限制在 64×64 的块内，使工作集完全驻留在 L2 中：
- CPI 从 6.26 降至 0.82
- DRAM refill 从 6.21% 降至 0.01%

**3. 多线程不能解决缓存问题**

multiply1 虽然通过 16 线程获得了 18.5× 加速，但 L1 DC miss ratio 反而从 16.9% 恶化到 26.5%，因为多线程间的缓存竞争加剧了问题。

## AMD Zen3 架构启示

在 AMD Zen3 架构上，矩阵乘法的性能瓶颈主要是：
1. **L1 数据缓存未命中** - 由非连续内存访问模式引起
2. **DTLB 未命中** - 跨页访问导致 TLB 压力
3. **CPI 过高** - 内存停顿导致流水线空闲

最有效的优化策略是 **Cache blocking + 循环交换**，将工作集限制在 L2 缓存内，同时保证块内访问的连续性。

