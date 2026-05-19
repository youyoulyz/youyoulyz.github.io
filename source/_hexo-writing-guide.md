---
title: Hexo 博客编写注意事项
date: 2026-05-19 22:00:00
draft: true
---

## 分支说明

仓库有两个分支，各自有不同的部署目标：

| 分支 | 部署目标 | 域名 | 说明 |
|------|---------|------|------|
| `main` | GitHub Pages | youyoulyz.github.io | 面向国际，无 ICP 备案要求 |
| `domestic` | 国内服务器 | youyou.moe | 面向国内，已配置 ICP 备案号、HTTPS |

## 配置文件差异

两个分支的 `_config.yml` 有不同之处：

- **title**: main 用"代码与生活"，domestic 用"youyoulyz的博客 | 上云就上悠悠云"
- **url**: main 是 `http://youyoulyz.github.io`，domestic 是 `http://youyou.moe`

⚠️ **不要合并整个分支到 main**，否则 domestic 的域名和标题配置会覆盖到 GitHub Pages 上。

## 正确的同步流程

### 新博客文章

只同步 `source/_posts/` 下的 md 文件，不要动 `_config.yml`、`themes/`、`source/CNAME` 等：

```bash
# 在 domestic 分支写完文章后
git add source/_posts/新文章.md
git commit -m "feat: add blog post xxx"
git push origin domestic

# 同步到 main（只同步文章文件）
git checkout main
git checkout domestic -- source/_posts/新文章.md
git add source/_posts/新文章.md
git commit -m "feat: add blog post xxx"
git push origin main
git checkout domestic
```

### 其他配置修改

如果修改了 `_config.yml`、主题等，**只在对应的分支上操作**，不要跨分支同步。

## 写作规范

1. **隐去具体 IP 地址**：用 `<服务器IP>` 代替实际 IP
2. **隐去用户名和密码**：不要写"用户名: xxx，密码: xxx"
3. **隐去内网地址**：`192.168.x.x`、`10.x.x.x` 等用占位符代替
4. **ICP 备案号**：只出现在 domestic 分支的主题配置中，不出现在文章内容里

## 部署

### domestic 分支
```bash
cd ~/youyoulyz.github.io
git checkout domestic
hexo generate && hexo deploy
```

### main 分支（GitHub Pages）
```bash
cd ~/youyoulyz.github.io
git checkout main
hexo generate
# GitHub Pages 自动部署，push 到 main 即可
```

## 常用命令

```bash
# 新建文章
hexo new post "文章标题"

# 预览
hexo server

# 生成静态文件
hexo generate   # 或 hexo g

# 清理缓存
hexo clean

# 部署（仅 domestic 分支配置了 deploy）
hexo deploy     # 或 hexo d
```
