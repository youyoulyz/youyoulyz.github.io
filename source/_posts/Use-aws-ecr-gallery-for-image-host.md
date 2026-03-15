---
title: 白嫖 AWS ECR 镜像源托管公开镜像
date: 2026-03-10 22:58:30
tags:
  - AWS
  - Docker
  - ECR
  ---

AWS 提供了一个免费的公有容器注册表服务（ECR Public Gallery），可以用来托管公开镜像。相比其他私有源，这个方案完全免费且支持全球加速。

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

## 注意事项

- ECR Public Registry 的镜像是公开的，任何人都可以拉取，不适合存放敏感信息
- 需要确保 AWS IAM 用户有足够的权限操作 ECR Public Registry
- 推荐的区域是 `us-east-1`，这是 ECR Public 的统一入口区域
- Docker 登录是一次性的，重新登录后才能继续 push

## 参考链接

- [AWS CLI Getting Started](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html)
- [ECR Public Registry Authentication](https://docs.aws.amazon.com/AmazonECR/latest/public/public-registry-auth.html)
- [ECR Public Console](https://ap-southeast-2.console.aws.amazon.com/ecr/public-registry/repositories?region=ap-southeast-2)