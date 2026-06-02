# Hexo 博客项目 — Agent 工作指南

## 分支说明

| 分支 | 部署目标 | 域名 | 说明 |
|------|---------|------|------|
| `main` | GitHub Pages | youyoulyz.github.io | 面向国际，无 ICP 备案要求 |
| `domestic` | 国内服务器 | youyou.moe | 面向国内，已配置 ICP 备案号、HTTPS |

## 配置差异

两个分支的 `_config.yml` 不同：
- **title**: main = "代码与生活"，domestic = "youyoulyz的博客 | 上云就上悠悠云"
- **url**: main = `http://youyoulyz.github.io`，domestic = `http://youyou.moe`

⚠️ **永远不要合并整个分支到 main**，否则 domestic 的域名和标题会覆盖到 GitHub Pages 上。

## 正确的同步流程

### 新博客文章
只同步 `source/_posts/` 下的 md 文件，不要碰 `_config.yml`、`themes/`、`source/CNAME`：

```bash
# 在 domestic 分支写完文章后
git checkout domestic
git add source/_posts/文章.md && git commit -m "feat: add blog post xxx" && git push origin domestic

# 同步到 main（只同步文章文件）
git checkout main
git checkout domestic -- source/_posts/文章.md
git add source/_posts/文章.md && git commit -m "feat: add blog post xxx" && git push origin main
git checkout domestic
```

### 配置/主题修改
只在对应分支操作，不要跨分支同步。

## Commit 前检查清单

每次 commit 之前，**必须**执行以下检查：

### 1. 事实确认
- 对新增或修改内容中引用的数据、版本、命令、文件路径等关键事实，用 `git log`、`grep`、`read_file` 等工具实时验证，**不依赖记忆或假设**
- 对新增或修改的**每一个链接（URL）**，逐个确认其可用性和准确性（可通过 `web_fetch` 或人工判断是否合理）
- 涉及技术方案的描述，必须与代码/配置中的实际实现一致

### 2. 隐私提示与脱敏
- 隐去作者的**真实姓名**，用占位符或用户名代替
- 隐去 **token、API key、secret、密码** 等敏感凭据，绝不写入任何可见内容
- 隐去**具体 IP 地址**：用 `<服务器IP>` 代替
- 隐去**内网地址**：`192.168.x.x`、`10.x.x.x` 等用占位符
- **ICP 备案号**：只出现在 domestic 主题配置中，不写在文章里

### 3. Commit 风格
- 使用 Conventional Commits：`type: description`
- 常用 type：`feat:`（新功能/文章）、`docs:`（文档）、`fix:`（修复）
- 描述聚焦于 **为什么** 做这个改动，而非复述做了什么

## 写作规范

1. **隐去具体 IP 地址**：用 `<服务器IP>` 代替
2. **隐去用户名和密码**：不要写具体凭据
3. **隐去内网地址**：`192.168.x.x`、`10.x.x.x` 等用占位符
4. **ICP 备案号**：只出现在 domestic 主题配置中，不写在文章里

## 部署

### domestic 分支（自动）
国内服务器已配置定时任务，每天 00:00 自动拉取 `domestic` 分支并重新部署。本地 push 到 `origin/domestic` 后，无需手动操作，次日凌晨自动生效。

- **定时任务**: `0 0 * * * /home/hexo/deploy-hexo.sh`
- **脚本路径**: `/home/hexo/deploy-hexo.sh`
- **日志**: `/home/hexo/hexo-deploy.log`
- **SSH 连接**: `ssh hexo`

手动立即部署：
```bash
ssh hexo "cd youyoulyz.github.io && hexo clean && hexo generate && hexo deploy"
```

### main 分支（GitHub Pages）
push 到 main 即自动部署。

## 常用命令

```bash
hexo new post "标题"    # 新建文章
hexo server             # 本地预览
hexo generate           # 生成静态文件 (hexo g)
hexo clean              # 清理缓存
hexo deploy             # 部署 (hexo d, 仅 domestic 分支配置了 deploy)
```
