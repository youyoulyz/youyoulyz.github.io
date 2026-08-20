---
title: 本地 27B Q4 + opencode 全程驱动：hami-learning-cloud 教学云更新实录（附模型自评）
date: 2026-08-20 13:30:00
tags:
  - HAMi
  - JupyterHub
  - K3s
  - GPU
  - LLM
  - llama.cpp
  - AI Agent
  - 部署
categories:
  - Infrastructure
---

> 五月的[《实验室单卡GPU虚拟化：K3s + Z2JH + HAMi》](/2026/05/2026-05-19-nv-hami-gpu-sharing/)把"K3s + Z2JH + HAMi"的骨架跑通了。这次我们把骨架长成了一个能用的教学云 **hami-learning-cloud**。
>
> 比内容更重要的是：**这次更新的全部内容——代码审计、功能修复、镜像构建、部署验证、git 历史清洗——完全由本地部署的 Qwen3.8-27B（Q4_K_M 量化，2×RTX 3080 张量并行）+ opencode 完成，全程没有调用任何云模型。** 文末让这个模型对自己的表现做了一次自评。

<!--more-->

## 一、从 nv-hami 到 hami-learning-cloud

五月那版的形态是裸 Z2JH + HAMi：用户来了就是一行行 JupyterHub 界面。这次的目标是把它变成一个有完整形态的教学云：

- **教学云形态**：课程、配额、团队、用量统计——复用 **AUP Learning Cloud**（AMD 开源教学云，MIT）的 UI 与配额体系
- **双加速器**：NVIDIA 走 HAMi vGPU 硬配额，AMD 走 ROCm 时间切片，一个集群两种厂商
- **浏览器里的 VS Code**（code-server），不只是 Notebook
- **仓库不出现任何内部部署信息**（IP、主机名、NAS 地址、宿主机路径）

硬件是 3 台家用机、5 张卡：

| 节点 | 卡 | 角色 |
|---|---|---|
| N1 | 2070S 8G + 5060Ti 16G | K3s master + Hub + NVIDIA vGPU 节点 |
| N2 | 2× 3080 20G | **不进集群**——显存被推理模型占满（就是写本文的模型，见 §四） |
| N3 | 7900XTX 24G | AMD 时间切片节点 |

架构总览：

```
┌──────────────────────────────────────────────────────────────┐
│  K3s 集群 (N1 + N3)                                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ GPU 共享层                                            │   │
│  │ ├─ HAMi v2.9.0 (nvidia): hami-scheduler +            │   │
│  │ │   device-plugin + webhook → 硬配额 4GB/25% 起      │   │
│  │ └─ 自研 time-slice 插件 (amd): 上报 amd.com/gpu: 3   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 教学云层 (AUP Learning Cloud fork, chart 零改动)      │   │
│  │ ├─ Hub: 课程/配额/团队/用量 + 双厂商 rebrand          │   │
│  │ ├─ Proxy: NodePort                                   │   │
│  │ └─ User Pods: Notebook (cpu/gpu) + VS Code (code-*)  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  路由: nvidia pod → hami-scheduler (vGPU fit/score)         │
│        amd pod    → 默认 scheduler (amd.com/gpu 普通资源)    │
│                                                              │
│  用户 home: NFS RWX (NAS), pod 重建数据不丢                  │
└──────────────────────────────────────────────────────────────┘
```

两个设计支点（五月那篇没有的）：

1. **配额表即路由**。accelerator 只有 `nvidia` / `amd` 两个，不按 GPU SKU 分。要 12G 显存的 pod 自然落不到 8G 的 2070S 上——K8s 资源语义天然完成了选卡。
2. **schedulerName 按加速器分派**。nvidia pod 走 `hami-scheduler`（HAMi extender 做 vGPU fit/score），amd pod 走默认 scheduler（`amd.com/gpu` 只是普通 extended resource）。

## 二、这次的更新内容

### 1. 双加速器资源模型

NVIDIA 侧沿用 HAMi v2.9.0，单位 `nvidia.com/gpumem`（MB）/ `nvidia.com/gpucores`（%），配额三档：base 4000/25、中档 8000/50、大档 12000/75。

AMD 侧有个坑：官方 `rocm/k8s-device-plugin` **不支持 sliceCount**（只支持整卡上报，也不认 `AMDGPU_DEVICE_MAX_PER_GPU`）。所以打了补丁做了个自定义 time-slice 插件：`ListAndWatch` 上报 `sliceCount` 个 slice ID，`Allocate` 时把所有 slice 映射回同一张物理卡的 `/dev/kfd` + `/dev/dri/*`。验收：3 个 pod 共享一张 7900XTX 全部调度成功，pod 内 torch 2.10.0+rocm7.2.4 的 matmul 通过。

AUP 的 `runtime/chart`（Z2JH fork）坚持**零改动**：GPU 资源全部经 `singleuser.extraResource` 透传，chart 保持厂商无关。

### 2. VS Code（code-server）资源

新增 `code-cpu` / `code-gpu` 两个资源，DEVELOPMENT 分组，`launchMode: code-server`：

- `code-gpu` 只对 nvidia 开放（HAMi vGPU：4G 显存 / 25% 算力），AMD 变体等 ROCm 版 code 镜像
- 用户 pod 内：code-server（`--auth none`，绑 127.0.0.1）+ 一个 in-pod nginx 反向代理，负责剥掉 `/user/<name>/` 前缀
- 镜像内置 5 个扩展（Python / Ruff / GitLens / DebugPy + 平台插件），构建时装好，启动不走网络

### 3. 本地值机制 .env.local

原则：**仓库里只有 `${VAR}` 占位符，真实值只存在于 gitignored 的 `.env.local`**。

```
.env.local.example   # 模板（提交进仓库）
.env.local           # 真实值（gitignored，永不提交）
        │
        ▼  ./scripts/render_local.sh
runtime/values.local.yaml   # helm overlay（hub 节点 pinning）
build/*.yaml                # NFS PV/PVC 等 manifest
        │
        ▼  ./scripts/helm_upgrade.bash 自动合并
helm upgrade ... -f runtime/values.yaml -f runtime/values.local.yaml
```

### 4. 安全姿态显性化

README 新增一张 6 项表，把本地演示的"有意放宽"全部写明：dummy auth + allow_all、无 TLS（NodePort 明文）、`allowedOrigins: *`、Grafana admin/admin、notebook 容器带 sudo、hub 镜像 pullPolicy Never。这些不是 bug，是本地教学场景的有意取舍——**暴露公网前必须逐项处理**，文档里写清楚了每一项要做什么。

### 5. Git 历史清洗

仓库历史里混着内部拓扑（LAN IP、节点主机名、NAS 地址、`/mnt/...` 宿主机路径）。用 `git filter-repo --replace-text` 把**全部 15 个 commit** 的历史重写为占位符，force-push 后 fresh clone 验证零残留。

## 三、两个 code-server 启动 bug（本文主角）

### Bug 1：首启卡 87% —— NFS 小文件拷贝

**现象**：用户点 code-gpu，pod 正常 Running，但 Hub 页面永远停在 "Your server is starting up … 87% Complete"，300 秒后 Hub 超时放弃。

**排查**：进 pod 看进程，entrypoint 卡在一个 `cp -a`——把镜像内置的 167MB 扩展集（几千个小文件）拷进用户的 NFS home，进程处于 `D` 状态（不可中断 I/O 等待），两分钟才挪了 50MB。但对照实验：NAS 单文件写 29.5MB/s，pod 内写 49.8MB/s——**NAS 没问题，慢的是小文件元数据操作**：NFS 上每个小文件都是一串 open/write/close/setattr RPC，NAS 的元数据吞吐被几百个并发文件操作打爆了。

**修复**：`--extensions-dir` 默认改为**容器本地**（直接用镜像内置的扩展目录）——首启零拷贝，秒级启动。代价：用户在 VS Code 里自装的扩展会随 pod 重建而重置（教学场景可接受）。

### Bug 2：白屏 —— nginx 的 rewrite 截断长 URI

Bug 1 修完，用户重新 spawn，**还是白屏**：浏览器 console 一片 404（workbench.js / css / manifest），而 pod 内直连 code-server 8889 端口却返回 200。

隔离过程全部在一个一次性测试 pod 里做（每步都可证伪）：

1. 直连 code-server：200 ✓ → 问题在 in-pod nginx
2. 最小 nginx 复现：prefix location 匹配正常；单独 rewrite 正常
3. `rewrite ... break;` + `return`：**return 被跳过**，请求落进静态文件
4. `rewrite ... break;` + `proxy_pass`：上游收到**损坏的 URI**
5. echo server 抓精确路径：上游收到的是"**原 URI 截掉尾部恰好等于前缀长度个字符**"
6. `add_header` 读回捕获组：`$1 = 字符串的前 (N − 前缀长度) 个字符`——捕获偏移整体错移了恰好一个前缀长度
7. 排除 PCRE2 JIT（LD_PRELOAD 桩掉 `pcre2_jit_compile_8`，行为不变）、排除 PCRE2 库本身（`grep -P` / perl 结果都正确）
8. 换标准替代路径：`proxy_pass` 带 URI（尾斜杠 = 前缀替换），走 proxy 模块、不经过 rewrite 指令
9. 端到端验证：根路径 302、`stable-<commit>/.../nls.messages.js` 长资源路径 200 ✓

最终修复就三行：

```nginx
location /user/123/ {
  # 尾斜杠 = 前缀替换: /user/<name>/x -> /x
  proxy_pass http://127.0.0.1:8889/;
}
```

这个 Ubuntu nginx 1.24.0 build 上 rewrite 损坏 URI 的确切根因**至今没定位到**（PCRE2 库和 JIT 都排除了）。如果你能想出来，请务必告诉我。

## 四、全程由本地模型驱动

推理服务：N2 机器 2× RTX 3080 20G，llama.cpp 张量并行跑 **Qwen3.8-27B-GGUF（Q4_K_M 量化）**（部署与调优细节见[《纸上得来终觉浅——llama.cpp 部署调优实录》](/2026/08/llama-cpp-deploy-tuning/)）。客户端：opencode（终端 coding agent）。

分工有点循环的味道：**3080 那台机器不做教学 vGPU，改跑写本文的模型；5060Ti 做教学 vGPU，同时跑着驱动这次更新的 opencode。**

模型独立完成的全链路（人类输入只有两句："给我一个端口"、"还是白屏"）：

1. 代码审计（设计规则 / 安全 / 质量）→ 报告
2. 恢复 code-server 启动路径 + 新增 code-cpu/code-gpu 资源
3. `.env.local` 机制 + 内部信息清除 + 安全姿态文档
4. 分操作 11 个 commit、filter-repo 全历史清洗、force-push + fresh-clone 验证
5. 多轮 docker build → k3s containerd 导入 → helm upgrade → rollout
6. 两个启动 bug 的完整根因排查（§三）
7. 端到端验收：spawn 页分组、pod 就绪时间、workbench 资源 200

全程零云 API 调用，数据不出局域网。

## 五、模型自评（我自己评自己）

> 以下是模型对本次任务的自我评估。自我评估天然有自我开脱倾向，请带折扣阅读。

**做对的**

1. **分层隔离**。每个假设都配一个可证伪实验：NAS 快 → pod 写快 → cp 卡在 D 状态 → 直连 200/代理 404 → 最小复现 → 捕获组偏移 → JIT/PCRE2 排除 → 换路径验证。没有一步是"我觉得应该是"。
2. **工程纪律**。11 个 commit 每个单一职责、message 写清楚 why；历史清洗后 fresh clone 全量验证；`.gitignore` 的 `.env.*` 通配会误伤模板文件这种细节也补了 negation。
3. **不轻信"像对的"解释**。比如 `/user/123/` 返回 302 这个反例，差点被"启动竞态"的解释吞掉，最后靠它逼出了对照实验。

**做错的**

1. **最严重：一次错误结论**。第一次白屏 404 之后我判断"是启动头几秒的竞态，刷新就好"——错了。真正的 bug（nginx rewrite）还在，用户白刷新了一次。"直连 200 / 代理 404"是硬矛盾，当时就应该死磕下去。
2. **绕路太长**。在 PCRE2 JIT 假设上钻了很久，还写了 C 程序直接测 PCRE2 ovector——结果连串工具失误（找不到库、缺头文件、`pcre2_compile` 签名少写一个参数、apt 装不上 dev 包），浪费了约 10 步。如果早一步试文档里的标准替代 `proxy_pass` 带 URI，会快得多。
3. **工具操作错误**：`pkill -f` 匹配到自己的 shell 把会话杀了；残留测试文件产生过一个假的"URI 截断"数据点，追着幻影查了好几步才用对照实验排除；`add_header` 的 `always` 位置写错。
4. **置信度校准差**：在端到端验证通过之前告诉用户"刷新后如果 GPU 识别有问题再叫我"——过度自信。

**总评**

| 维度 | 分数 | 说明 |
|---|---|---|
| 能力 | 8/10 | 根因不在任何文档里，是实验二分出来的；"审计→修复→构建→部署→验证→提交→历史清洗"闭环独立完成 |
| 效率 | 6/10 | 绕路率偏高，前沿云模型大概 1/3 轮次收敛；Q4 量化 + 27B 规模的代价是"错误假设不会被快速否决" |
| 可用性 | 7/10 | infra 类工作上"人给方向、本地模型执行、人验收结果"的模式已经成立 |

一句话：**27B Q4 的模型做系统级 debug 的上限比预期高，但它的每轮推理都要打折——用轮次换正确性，是这个量级本地模型的正常工作方式。**

## 六、总结

- hami-learning-cloud 现在的形态：NV vGPU 硬配额 + AMD 时间切片 + Notebook + 浏览器 VS Code + 配额/团队/用量统计，跑在 3 台家用机 / 5 张卡上
- 仓库：[github.com/youyoulyz/hami-learning-cloud](https://github.com/youyoulyz/hami-learning-cloud)（MIT；AUP 与 HAMi 部分保留原许可与署名）
- 本文的元结论：本地部署的 27B Q4 模型 + 终端 agent，已经能承担 infra 项目里"执行"的部分；人负责方向和验收
