---
title: DeepEP H20 GPU Replay Benchmark 教程
date: 2026-05-06 10:00:00
tags:
  - DeepEP
  - GPU
  - H20
  - NCCL
  - Benchmark
  - DeepSeek
categories:
  - Deep Learning
  - System
---

本教程详细说明如何在 NVIDIA H20 GPU 上运行 DeepEP replay benchmark。

<!-- more -->

## 目录

1. [环境配置](#环境配置)
2. [NCCL 兼容性问题](#nccl兼容性问题)
3. [运行 Benchmark](#运行-benchmark)
4. [结果分析](#结果分析)
5. [故障排除](#故障排除)

## 环境配置

### 硬件要求

- NVIDIA H20-3e GPU (8卡)
- CUDA 13.0+
- NVLink 连接

### 软件要求

- Python 3.12
- PyTorch 2.x with CUDA 13.0
- NCCL 2.28.9+ (PyTorch 内置) 或 NCCL 2.30.4+ (系统)
- NVSHMEM

### 容器信息

- 镜像名称: `deepep-shape-sim:latest`
- 容器名称: `deepep_exp`
- 工作目录: `/workspace`

## NCCL 兼容性问题

### 问题描述

DeepEP V2 需要使用 NCCL Gin API (如 `ncclCommQueryProperties`)，这些 API 仅在 NCCL 2.19+ 中可用。但 PyTorch 内置的 NCCL 版本可能不包含这些 API，导致符号未定义错误：

```
ImportError: undefined symbol: ncclCommQueryProperties
```

### 解决方案

我们提供了两种解决方案：

#### 方案一：使用系统 NCCL (推荐)

通过 `LD_PRELOAD` 强制加载系统 NCCL 库：

```bash
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libnccl.so.2
export EP_SUPPRESS_NCCL_CHECK=1
```

**优点**：
- 简单快速
- 无需重新编译
- 使用更新的 NCCL 版本

**缺点**：
- 需要每次运行时设置环境变量
- 系统和 PyTorch NCCL 版本冲突检查

#### 方案二：修改源码重新编译 (推荐)

修改 DeepEP 源码以兼容旧版 NCCL，详见 [Dockerfile 修改部分](#dockerfile-修改)。

**优点**：
- 自动适配 NCCL 版本
- 无需手动设置环境变量
- 更加稳定可靠

**缺点**：
- 需要重新编译 DeepEP

## 运行 Benchmark

### 方式一：使用脚本 (推荐)

```bash
cd /workspace
./run_replay_bench.sh <warmup> <iterations> <num_processes>
```

示例：
```bash
# 快速测试
./run_replay_bench.sh 3 10 8

# 正式测试
./run_replay_bench.sh 5 20 8

# 完整测试
./run_replay_bench.sh 10 50 8
```

### 方式二：直接运行 Python

```bash
cd /workspace

# 如果使用方案一，需要先设置环境变量
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libnccl.so.2
export EP_SUPPRESS_NCCL_CHECK=1

# 运行 benchmark
python3 replay_bench_v1.py \
  --dispatch-input-csv dispatch_input.csv \
  --combine-input-csv combine_input.csv \
  --output-csv h20_replay_results.csv \
  --warmup 5 \
  --iters 20 \
  --num-processes 8
```

### 参数说明

- `--dispatch-input-csv`: Dispatch 操作配置文件
- `--combine-input-csv`: Combine 操作配置文件
- `--output-csv`: 结果输出文件
- `--warmup`: 预热迭代次数（默认：5）
- `--iters`: 正式测试迭代次数（默认：20）
- `--num-processes`: GPU 进程数（默认：8）

### 输入文件格式

#### dispatch_input.csv

```csv
run_id,hidden,num_experts,num_topk,num_local_experts,num_tokens,tokens_per_rank,tokens_per_expert,x_dtype,timing_us
0,7168,128,4,16,16013,2045;2034;1991;1972;1964;2136;1936;1935,507;518;...,bfloat16,539.761
```

字段说明：
- `tokens_per_rank`: 分号分隔的每个 rank 的 token 数量
- `timing_us`: 原始 benchmark 的 dispatch 时间（微秒）

#### combine_input.csv

```csv
run_id,hidden,num_experts,num_topk,num_local_experts,num_tokens,tokens_per_rank,tokens_per_expert,has_bias,timing_us
0,7168,128,4,16,16013,2045;2034;1991;1972;1964;2136;1936;1935,507;518;...,0,247.884
```

## 结果分析

### 输出文件格式

`h20_replay_results.csv`:

```csv
run_id,num_tokens,hidden,num_experts,num_topk,dispatch_avg_us,dispatch_min_us,dispatch_max_us,combine_avg_us,combine_min_us,combine_max_us,csv_dispatch_timing_us,csv_combine_timing_us
0,16013,7168,128,4,2275.1,2241.2,2319.2,6457.4,6350.6,6830.4,539.761,247.884
```

### 关键指标

- **dispatch_avg_us**: Dispatch 平均延迟（微秒）
- **dispatch_min_us**: Dispatch 最小延迟
- **dispatch_max_us**: Dispatch 最大延迟
- **combine_avg_us**: Combine 平均延迟（微秒）
- **csv_dispatch_timing_us**: 原始 H20 benchmark 的 dispatch 时间
- **csv_combine_timing_us**: 原始 H20 benchmark 的 combine 时间

### 性能分析示例

```python
import pandas as pd
import matplotlib.pyplot as plt

# 加载结果
df = pd.read_csv('h20_replay_results.csv')

# 计算 replay vs 原始 benchmark 的性能比值
df['dispatch_ratio'] = df['dispatch_avg_us'] / df['csv_dispatch_timing_us']
df['combine_ratio'] = df['combine_avg_us'] / df['csv_combine_timing_us']

# 打印统计信息
print(f"Dispatch 性能比值: {df['dispatch_ratio'].mean():.2f}x")
print(f"Combine 性能比值: {df['combine_ratio'].mean():.2f}x")

# 绘制性能对比图
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

ax1.scatter(df['num_tokens'], df['dispatch_avg_us'], label='Replay', alpha=0.6)
ax1.scatter(df['num_tokens'], df['csv_dispatch_timing_us'], label='Original', alpha=0.6)
ax1.set_xlabel('Num Tokens')
ax1.set_ylabel('Dispatch Time (us)')
ax1.set_title('Dispatch Performance')
ax1.legend()
ax1.grid(True, alpha=0.3)

ax2.scatter(df['num_tokens'], df['combine_avg_us'], label='Replay', alpha=0.6)
ax2.scatter(df['num_tokens'], df['csv_combine_timing_us'], label='Original', alpha=0.6)
ax2.set_xlabel('Num Tokens')
ax2.set_ylabel('Combine Time (us)')
ax2.set_title('Combine Performance')
ax2.legend()
ax2.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('performance_comparison.png', dpi=300)
plt.show()
```

## 故障排除

### 问题 1: NCCL 符号未定义

**错误信息**:
```
ImportError: undefined symbol: ncclCommQueryProperties
```

**解决方案**:

方案一：设置环境变量
```bash
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libnccl.so.2
export EP_SUPPRESS_NCCL_CHECK=1
```

方案二：重新编译 DeepEP（推荐）
```bash
cd /workspace/DeepEP
pip3 uninstall deep-ep -y --break-system-packages
python3 setup.py bdist_wheel
pip3 install dist/*.whl --break-system-packages
```

### 问题 2: NCCL 版本冲突

**错误信息**:
```
AssertionError: Duplicate NCCL runtime found in the current system
```

**解决方案**:
```bash
export EP_SUPPRESS_NCCL_CHECK=1
```

### 问题 3: CUDA Out of Memory

**错误信息**:
```
RuntimeError: CUDA out of memory
```

**解决方案**:
- 减少进程数: `--num-processes 4`
- 减少 token 数量
- 增加缓冲区大小

### 问题 4: NCCL 初始化失败

**错误信息**:
```
NCCL error: unhandled system error
```

**解决方案**:
```bash
# 检查 NCCL 版本
python3 -c "import torch; print(torch.cuda.nccl.version())"

# 检查 GPU 连接
nvidia-smi topo -m

# 设置 NCCL 调试
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=ALL
```

### 问题 5: 进程卡死

**可能原因**:
- NCCL 通信死锁
- GPU 之间的同步问题

**解决方案**:
```bash
# 设置 NCCL 超时
export NCCL_BLOCKING_WAIT=1
export NCCL_TIMEOUT=1800  # 30 分钟

# 或使用更小的拓扑
export NCCL_P2P_DISABLE=1  # 禁用 P2P
```

## 高级配置

### 自定义输入数据

创建新的输入 CSV 文件：

```python
import pandas as pd

# 生成自定义配置
configs = []
for hidden in [4096, 5120, 7168]:
    for experts in [128, 256]:
        for topk in [4, 6, 8]:
            for tokens in [4096, 8192, 16384]:
                configs.append({
                    'hidden': hidden,
                    'num_experts': experts,
                    'num_topk': topk,
                    'num_tokens': tokens,
                    # ... 其他字段
                })

df = pd.DataFrame(configs)
df.to_csv('custom_dispatch_input.csv', index=False)
```

### 启用 RDMA

```bash
python3 replay_bench_v1.py \
  --dispatch-input-csv dispatch_input.csv \
  --combine-input-csv combine_input.csv \
  --output-csv h20_rdma_results.csv \
  --enable-rdma \
  --num-processes 8
```

### 调整 Buffer 大小

修改 `replay_bench_v1.py` 中的 buffer 计算逻辑：

```python
# 原始计算
num_nvl_bytes = num_max_tokens_per_rank * hidden * dtype_size * num_ranks * 8

# 增大 buffer (2x)
num_nvl_bytes = num_max_tokens_per_rank * hidden * dtype_size * num_ranks * 8 * 2
```

## 参考链接

- [DeepEP GitHub](https://github.com/deepseek-ai/DeepEP)
- [NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/)
- [PyTorch Distributed](https://pytorch.org/docs/stable/distributed.html)

## 联系方式

如有问题，请联系：
- DeepEP 问题: [GitHub Issues](https://github.com/deepseek-ai/DeepEP/issues)
- 本实验问题: 查看 `/workspace/README_H20_REPLAY.md`
