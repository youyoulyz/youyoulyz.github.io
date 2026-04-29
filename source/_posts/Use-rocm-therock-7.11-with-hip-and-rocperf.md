---
title: 使用 TheRock + HIP 和 rocperf 的一些情报
author: 
date: 2026-03-10 22:58:30
tags:
  - ROCm
  - AMD
  - Ryzen AI
layout: post
---

## 简介

本文记录在 Ryzen AI Max PRO 395 平台上安装和使用 TheRock ROCm 7.11.0 的过程，包括 HIP 开发环境和 rocperf 性能测试工具的配置。

## 安装 TheRock ROCm 7.11.0

### 环境信息

- **CPU**: Ryzen AI Max PRO 395 (gfx1151)
- **OS**: Ubuntu 24.04
- **ROCm 版本**: 7.11.0-preview

### 前置条件

#### 1. 安装 OEM 内核

Ryzen AI APU 需要 Ubuntu 24.04 的 OEM 内核 6.14：

```bash
sudo apt update && sudo apt install linux-image-6.14.0-1018-oem
```

安装完成后**重启系统**。

#### 2. 安装依赖包

```bash
# 安装基础依赖
sudo apt install sudo wget libatomic1 python3.12 python3.12-venv
```

#### 3. 配置 GPU 访问权限

有两种方式配置 GPU 访问权限：

**方式一：添加用户到组（推荐）**

```bash
sudo usermod -a -G render,video $LOGNAME
```

**方式二：使用 udev 规则**

```bash
sudo tee /etc/udev/rules.d/70-amdgpu.rules << EOF
KERNEL=="kfd", GROUP="render", MODE="0666"
SUBSYSTEM=="drm", KERNEL=="renderD*", GROUP="render", MODE="0666"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

配置完成后需要**重启系统**以应用所有设置。

### 安装 ROCm Core SDK

#### 添加 ROCm 软件源

```bash
# 下载并安装 GPG 密钥
sudo mkdir --parents --mode=0755 /etc/apt/keyrings
wget https://repo.amd.com/rocm/packages/gpg/rocm.gpg -O - | gpg --dearmor | sudo tee /etc/apt/keyrings/amdrocm.gpg > /dev/null

# 添加 ROCm 仓库配置（Ubuntu 24.04）
sudo tee /etc/apt/sources.list.d/rocm.list << EOF
deb [arch=amd64 signed-by=/etc/apt/keyrings/amdrocm.gpg] https://repo.amd.com/rocm/packages/ubuntu2404 stable main
EOF

# 更新包列表
sudo apt update
```

#### 安装 ROCm 核心包（针对 gfx1151）

```bash
# 安装 TheRock Core Runtime (用于运行应用)
sudo apt install amdrocm7.11-gfx1151

# 或安装完整的 SDK (包含编译器、开发工具等)
sudo apt install amdrocm-core-sdk-gfx1151
```

#### 可选：使用 Python pip 安装 ROCm 库

对于 Python 项目，可以使用 pip 方式安装 ROCm 库：

```bash
# 创建并激活 Python 虚拟环境
python3.12 -m venv .venv
source .venv/bin/activate

# 使用 pip 安装 ROSM 7.11 (适用于 Ryzen AI Max PRO 395)
python -m pip install --index-url https://repo.amd.com/rocm/whl/gfx1151/ "rocm[libraries,devel]"
```

### 验证安装

#### 检查 ROCm 版本

```bash
rocminfo | grep "Driver Version"
```

期望输出应显示类似 `7.11.0`。

#### 检查 GPU 信息

```bash
rocminfo --show-hw
```

