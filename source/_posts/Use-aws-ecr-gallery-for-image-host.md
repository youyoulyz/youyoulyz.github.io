---
title: 白嫖 AWS ECR / Aliyun 镜像源托管公开镜像
date: 2026-03-10 22:58:30
tags:
  - AWS
  - Docker
  - ECR
  - Aliyun
  - ACR
---

AWS 提供了一个免费的公有容器注册表服务（ECR Public Gallery），可以用来托管公开镜像。相比其他私有源，这个方案完全免费且支持全球加速。阿里云同样提供了免费的容器镜像服务（ACR）个人版，国内访问速度更快。两者结合使用，可以覆盖国内和海外用户的需求。

## Prerequisites

- AWS 账户
- AWS CLI 安装并配置好 credentials

### 安装 AWS CLI

参考官方文档：
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html

### 配置 AWS CLI

```bash
sudo aws configure
```

输入 Access Key ID、Secret Access Key、默认区域等信息。

## 创建 ECR Public Repository

1. 登录 AWS Console
2. 访问 ECR Public Registry：https://ap-southeast-2.console.aws.amazon.com/ecr/public-registry/repositories?region=ap-southeast-2
3. 点击 "Create repository" 创建一个新的公共仓库
4. 填写仓库名称（例如 `myproject/myimage`）
5. 完成创建后，记下完整的仓库 URL 格式：`public.ecr.aws/<owner-id>/<repo-name>`

## Docker 登录 ECR Public Registry

```bash
# 获取当前用户并切换到 root（如果需要 sudo 权限）
sudo su

# 使用 AWS CLI 获取登录凭证并自动登录到 ECR Public Registry
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
```

参考文档：https://docs.aws.amazon.com/AmazonECR/latest/public/public-registry-auth.html

## 构建并推送镜像

### 准备 Dockerfile

确保你的项目目录中有 `Dockerfile`。

### 构建镜像

```bash
# 构建镜像
docker build -t myimage:latest .
```

### Tag 镜像

将镜像 tag 为你创建仓库的完整 URL：

```bash
# 假设你的仓库是 public.ecr.aws/abcd1234/myrepo
docker tag myimage:latest public.ecr.aws/abcd1234/myrepo:latest
```

### 推送镜像

```bash
docker push public.ecr.aws/abcd1234/myrepo:latest
```

## 拉取使用镜像

其他人可以使用以下命令拉取你的公开镜像：

```bash
# 先登录（可选，只读访问不需要，登录每月2T FREE，匿名500G）
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws

# 拉取镜像
docker pull public.ecr.aws/abcd1234/myrepo:latest
```

## 完整命令流程示例

```bash
# 1. 切换到 root（如果需要）
sudo su

# 2. 登录 ECR Public Registry
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws

# 3. 构建镜像
docker build -t myapp:1.0 .

# 4. Tag 镜像（替换为实际的仓库地址）
docker tag myapp:1.0 public.ecr.aws/your-owner-id/your-repo-name:1.0

# 5. 推送镜像
docker push public.ecr.aws/your-owner-id/your-repo-name:1.0

# 6. 验证推送成功
docker pull public.ecr.aws/your-owner-id/your-repo-name:1.0
```

---

## 阿里云 ACR 个人版（免费）

除了 AWS ECR Public，阿里云的容器镜像服务（ACR）个人版也是一个免费的镜像托管方案，适合国内用户使用，访问速度更快。

### Prerequisites

- 阿里云账号
- Docker 已安装

### 开通 ACR 个人版

1. 登录阿里云控制台
2. 访问容器镜像服务：https://cr.console.aliyun.com
3. 首次使用会自动开通个人版实例，免费使用

### 获取登录凭证

访问凭证页面：https://cr.console.aliyun.com/cn-shanghai/instance/credentials

在「访问凭证」页面设置 Docker 登录密码，然后获取固定密码或临时凭证。

### Docker 登录 ACR

```bash
# 登录阿里云 ACR（替换为你的区域和用户名）
docker login --username=<你的阿里云用户名> registry.cn-shanghai.aliyuncs.com
```

输入密码后即可登录成功。

### 创建命名空间和仓库

1. 在 ACR 控制台创建命名空间（Namespace）
2. 在命名空间下创建镜像仓库
3. 仓库地址格式：`registry.cn-shanghai.aliyuncs.com/<namespace>/<repo-name>`

### 构建并推送镜像

```bash
# 构建镜像
docker build -t myimage:latest .

# Tag 镜像（替换为实际地址）
docker tag myimage:latest registry.cn-shanghai.aliyuncs.com/my-namespace/myimage:latest

# 推送镜像
docker push registry.cn-shanghai.aliyuncs.com/my-namespace/myimage:latest
```

### 拉取使用镜像

```bash
# 登录（可选，公开仓库不需要）
docker login --username=<阿里云用户名> registry.cn-shanghai.aliyuncs.com

# 拉取镜像
docker pull registry.cn-shanghai.aliyuncs.com/my-namespace/myimage:latest
```

### 完整命令流程示例

```bash
# 1. 登录 ACR
docker login --username=myuser registry.cn-shanghai.aliyuncs.com

# 2. 构建镜像
docker build -t myapp:1.0 .

# 3. Tag 镜像
docker tag myapp:1.0 registry.cn-shanghai.aliyuncs.com/my-namespace/myapp:1.0

# 4. 推送镜像
docker push registry.cn-shanghai.aliyuncs.com/my-namespace/myapp:1.0

# 5. 验证拉取
docker pull registry.cn-shanghai.aliyuncs.com/my-namespace/myapp:1.0
```

### 实际示例

```bash
# 登录阿里云 ACR 个人版实例
sudo docker login --username=<your-username> crpi-<your-instance-id>.cn-shanghai.personal.cr.aliyuncs.com

# 构建镜像并自动推送（以 auplc-installer 为例）
sudo MIRROR_PREFIX="m.daocloud.io" \
     MIRROR_PIP="https://pypi.tuna.tsinghua.edu.cn/simple" \
     GPU_TYPE=strix-halo \
     ./auplc-installer img build sht-llm
```

> 注意：个人版实例的 registry 地址格式为 `crpi-<id>.cn-shanghai.personal.cr.aliyuncs.com`，与标准版略有不同。

### 可用区域

阿里云 ACR 在多个区域提供服务，常用的包括：

| 区域 | Endpoint |
|------|----------|
| 上海 | `registry.cn-shanghai.aliyuncs.com` |
| 杭州 | `registry.cn-hangzhou.aliyuncs.com` |
| 北京 | `registry.cn-beijing.aliyuncs.com` |
| 深圳 | `registry.cn-shenzhen.aliyuncs.com` |
| 香港 | `registry.cn-hongkong.aliyuncs.com` |

## AWS ECR vs 阿里云 ACR 对比

| 特性 | AWS ECR Public | 阿里云 ACR 个人版 |
|------|---------------|------------------|
| 免费额度 | 匿名 500G/月，登录 2T/月 | 免费，无明确流量限制 |
| 国内访问 | 较慢（需翻墙或走国际线路） | 快（国内节点） |
| 适用场景 | 面向全球用户 | 面向国内用户 |
| 命名空间 | `public.ecr.aws/<owner>/<repo>` | `registry.cn-<region>.aliyuncs.com/<ns>/<repo>` |
| 登录方式 | AWS CLI 获取 token | 用户名 + 固定密码 |

**建议**：如果你的用户主要在国内，优先使用阿里云 ACR；如果面向全球，使用 AWS ECR Public。

## 注意事项

- 公有镜像仓库的镜像任何人都可以拉取，不适合存放敏感信息
- **AWS ECR Public**：推荐区域是 `us-east-1`，这是 ECR Public 的统一入口区域
- **阿里云 ACR**：选择离目标用户最近的区域以获得最佳速度
- Docker 登录是一次性的，重新登录后才能继续 push
- 如果需要存储私有镜像，两个平台都支持设置为私有仓库

## 参考链接

- [AWS CLI Getting Started](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html)
- [ECR Public Registry Authentication](https://docs.aws.amazon.com/AmazonECR/latest/public/public-registry-auth.html)
- [ECR Public Console](https://ap-southeast-2.console.aws.amazon.com/ecr/public-registry/repositories?region=ap-southeast-2)
- [阿里云容器镜像服务 ACR](https://www.aliyun.com/product/acr)
- [阿里云 ACR 控制台](https://cr.console.aliyun.com)
- [阿里云 ACR 访问凭证](https://cr.console.aliyun.com/cn-shanghai/instance/credentials)