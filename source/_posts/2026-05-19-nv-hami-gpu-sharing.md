---
title: 实验室单卡GPU虚拟化：K3s + Z2JH + HAMi 实现多用户GPU共享
date: 2026-05-19 21:30:00
tags:
  - Kubernetes
  - GPU
  - HAMi
  - JupyterHub
  - K3s
categories:
  - Infrastructure
---

## 背景

实验室有一台 GPU 服务器（RTX 3080 + RTX 5060 Ti，共两张卡），之前多个人要用 GPU 只能排队——一个人独占整张卡，其他人等着。这种粗放模式显然效率太低，尤其是跑 Jupyter Notebook 做实验的时候，大部分时间 GPU 根本跑不满。

目标很明确：
- 一台物理机，两张消费级 GPU
- 多个用户同时使用 JupyterHub 做 PyTorch 开发
- 每个用户分到 **4GB 显存 + 25% GPU 算力**
- 一张卡最多切 5 个 vGPU，总共支撑 10 个用户并发
- 能可视化监控 GPU 使用情况

最终效果：一个 Helm values 文件 + 一套自动化安装脚本，在 K3s 上把 Z2JH、HAMi、HAMi-WebUI 全栈跑起来。

---

## 架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                     Ubuntu 24.04 LTS                        │
│  K3s (轻量 Kubernetes)                                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HAMi — GPU 虚拟化中间件                              │   │
│  │  ├─ hami-scheduler     # 自定义 GPU 感知调度器       │   │
│  │  ├─ hami-device-plugin # 注册 vGPU 资源到 K8s        │   │
│  │  └─ webhook            # 自动注入 GPU 环境变量       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Z2JH (JupyterHub)                                   │   │
│  │  ├─ Hub          # 用户认证 & 管理                   │   │
│  │  ├─ Proxy        # NodePort                          │   │
│  │  └─ User Pods    # PyTorch CUDA Notebook             │   │
│  │                  # 每个 Pod: 4GB显存 / 25%算力       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HAMi-WebUI (魔改版)                                 │   │
│  │  ├─ 前端: Vue.js / NodePort                          │   │
│  │  ├─ 后端: Go (Kratos)                                │   │
│  │  └─ Prometheus + DCGM-Exporter 监控                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  GPU: RTX 3080 + RTX 5060 Ti                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 核心技术选型

### K3s 替代标准 K8s

标准 K8s 太重了——kube-apiserver、etcd、controller-manager、scheduler 一堆组件，单机跑起来吃掉好几 GB 内存。我们是单节点场景，用 K3s 完美替代：
- 二进制只有 **~70MB**，内存占用不到 500MB
- 自带 containerd、Flannel CNI
- 完全兼容标准 K8s API，Helm 直接可用
- 禁用了 traefik ingress，用 NodePort 直出端口更简单

### HAMi 做 GPU 虚拟化

HAMi（异构算力管理中间件，原 vGPU-device-plugin）是开源的 GPU 共享调度方案，核心能力：

| 功能 | 说明 |
|------|------|
| **显存切分** | 一张物理 GPU 按 MB 级别切分，`deviceSplitCount: 10` 最多切 10 份 |
| **算力共享** | `deviceCoreSharing: 1`，按比例分配 GPU SM 算力 |
| **调度器** | 自定义 scheduler + admission webhook，binpack 策略优先塞满同一张卡 |
| **多厂商** | 除了 NVIDIA，还支持华为 Ascend、海光 DCU、寒武纪 MLU 等 |

用户 Pod 只需声明三个资源：
```yaml
resources:
  limits:
    nvidia.com/gpu: "1"          # 1 个 vGPU
    nvidia.com/gpumem: "4000"    # 4 GB 显存
    nvidia.com/gpucores: "25"    # 25% 算力
```

HAMi webhook 会自动注入 NVIDIA 可见设备环境变量，scheduler 把 Pod 调度到有足够剩余资源的 GPU 切片上。

### Z2JH 做用户界面

JupyterHub 是学术界最流行的多用户 Notebook 方案，Z2JH 是它的 Helm Chart。关键配置：

```yaml
singleuser:
  extraResource:
    limits:
      nvidia.com/gpumem: "4000"    # 4 GB 显存
      nvidia.com/gpucores: "25"    # 25% 算力
  extraPodConfig:
    runtimeClassName: nvidia        # 指定 NVIDIA runtime
scheduling:
  userScheduler:
    enabled: false                  # 用 HAMi 调度器
```

认证插件使用 Dummy Authenticator，适合实验室内网环境。

---

## 魔改 HAMi-WebUI

原生的 HAMi-WebUI 有个 bug：GPU 使用率永远显示为 0。

### Bug 根因

HAMi scheduler 在 Pod annotation 里编码 GPU 分配信息时，是按**所有容器**（包括 init containers）的顺序写入的。但 WebUI 解码时只遍历了 `pod.Spec.Containers`，忽略了 init containers，导致索引错位——应用容器拿到的永远是 init container（`block-cloud-metadata`）的空设备记录。

```
Scheduler 写入 annotation:
  [init-container设备(空)] ; [主容器设备(GPU-xxx,NVIDIA,4000,25)]

WebUI 解码 (修复前):
  i=0 → pod.Spec.Containers[0] → 错误地取了 init-container 的空记录 ❌
```

### 修复方案

修了三个文件：

**1. `util.go` — 解码时考虑 init containers**
```go
// 修复前：只算 Containers
totalContainers := len(pod.Spec.Containers)

// 修复后：InitContainers + Containers
totalContainers := len(pod.Spec.InitContainers) + len(pod.Spec.Containers)
```

**2. `pod.go` — 分配 device 索引时跳过 init containers**
```go
// 修复前：直接用 i 作为索引，错位
c.ContainerDevices = bizContainerDevices[i]

// 修复后：加上 initContainer 的偏移量
initContainerCount := len(pod.Spec.InitContainers)
deviceIdx := initContainerCount + i
if deviceIdx < len(bizContainerDevices) {
    c.ContainerDevices = bizContainerDevices[deviceIdx]
}
```

**3. `node.go` — 添加 nil 防护**

当 POST 请求不带 filters 时 `req.Filters` 为 nil，会触发 panic。加上判空：
```go
if filters != nil {
    if filters.Ip != "" && filters.Ip != nodeReply.Ip { continue }
    // ...
}
```

修复后 WebUI 正确显示各 GPU 使用率、显存和算力占用。

---

## 部署流程

整套部署压缩到三个脚本里：

```bash
# 1. 安装 K3s + Helm + 部署 Z2JH
./install-z2jh.sh

# 2. 查看状态
./status-z2jh.sh

# 3. 卸载（含数据清理）
./uninstall-z2jh.sh
```

HAMi 和 HAMi-WebUI 通过 Helm 单独部署：
```bash
helm install hami HAMi/charts/hami -n kube-system
helm install hami-webui HAMi-WebUI/charts/hami-webui -n kube-system
```

## 访问方式

| 服务 | 地址 | 说明 |
|------|------|------|
| JupyterHub | `http://<服务器IP>:31080` | 按配置认证登录 |
| HAMi-WebUI | `http://<服务器IP>:32361` | GPU 资源监控面板 |

---

## 资源分配示意

```
RTX 3080 (10GB)          RTX 5060 Ti (16GB)
┌─────────────────┐      ┌─────────────────┐
│ vGPU 1: 4GB 25% │      │ vGPU 6: 4GB 25% │
│ vGPU 2: 4GB 25% │      │ vGPU 7: 4GB 25% │
│ vGPU 3: 2GB 25% │      │ vGPU 8: 4GB 25% │
│                 │      │ vGPU 9: 4GB 25% │
└─────────────────┘      └─────────────────┘
   5 个 vGPU 切片             5 个 vGPU 切片
```

每个 JupyterHub 用户 Pod 自动分配到一个 vGPU 切片，跑 PyTorch 和独占一张卡体验完全一致——`torch.cuda.is_available()` 返回 true，`nvidia-smi` 能看到 GPU，只不过显存上限是 4GB。

---

## 总结

这套方案的核心价值：
1. **消费级 GPU 也能虚拟化**——不需要 Tesla/V100 这类数据中心卡
2. **资源利用率大幅提升**——从一人占整卡到十人共享
3. **开箱即用**——K3s 单机一条脚本拉起全栈
4. **可观测**——WebUI 实时看 GPU 占用，Prometheus 留存历史数据
5. **易维护**——Helm 统一管理，卸载一条命令清理干净

适合实验室、小团队、教学机房等预算有限但需要多用户 GPU 环境的场景。
