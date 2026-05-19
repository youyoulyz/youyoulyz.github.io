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

## 写作规范

1. **隐去具体 IP 地址**：用 `<服务器IP>` 代替
2. **隐去用户名和密码**：不要写具体凭据
3. **隐去内网地址**：`192.168.x.x`、`10.x.x.x` 等用占位符
4. **ICP 备案号**：只出现在 domestic 主题配置中，不写在文章里

## 部署

### domestic 分支
```bash
git checkout domestic
hexo generate && hexo deploy
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
