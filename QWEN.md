# Hexo Blog — youyoulyz

## Project Overview

This is a **Hexo static site generator** blog project, authored by **youyoulyz**. The blog is written primarily in **Chinese (zh-CN)** and covers technical topics including:

- AMD ROCm/GPU tooling (MI50, uProf, DeepEP H20)
- Kubernetes/GPU sharing (K3s, HAMi)
- OpenWrt and network setups (ESXi, Shanghaitech)
- AI/ML infrastructure (Triton, LLVM, AWS ECR)
- Various Linux and DevOps tutorials

**Tech stack:**
- **Hexo 7.3.0** — Static site generator
- **Node.js 22** — Runtime
- **Themes:** `hexo-theme-landscape` (active), `hexo-theme-yun` (installed)
- **Renderers:** Pug (templates), EJS, Markdown (marked), Stylus (CSS)
- **Deployment:** GitHub Pages (via GitHub Actions) + domestic Alibaba Cloud server

## Branch Strategy

| Branch | Deploy Target | Domain | Notes |
|--------|--------------|--------|-------|
| `main` | GitHub Pages | `youyoulyz.github.io` | International, no ICP filing required |
| `domestic` | Alibaba Cloud server | `youyou.moe` | Domestic China, ICP filing configured, HTTPS |

**Critical rules:**
- **Never merge an entire branch into `main`** — this would overwrite the GitHub Pages domain/title with domestic config.
- When syncing blog posts between branches, **only sync `source/_posts/` files** — never `_config.yml`, `themes/`, or `source/CNAME`.
- Configuration/theme changes should only be made on the relevant branch, never cross-synced.

See `AGENTS.md` for the detailed sync workflow.

## Key Commands

```bash
npm run build     # Generate static files (hexo generate)
npm run clean     # Clean cache (hexo clean)
npm run server    # Local preview (hexo server)
npm run deploy    # Deploy (hexo deploy, only configured on domestic branch)
```

### Creating a new post

```bash
npx hexo new post "标题"
# or manually: create a .md file in source/_posts/
```

Post scaffolds are in `scaffolds/post.md` (minimal frontmatter: title, date, tags).

## Deployment

### main (GitHub Pages) — Automatic
Pushing to `main` triggers the GitHub Actions workflow (`.github/workflows/pages.yml`), which:
1. Checks out the repo
2. Installs dependencies (with npm cache)
3. Runs `npm run build`
4. Uploads and deploys to GitHub Pages

### domestic (Alibaba Cloud) — Automatic + Manual
- **Auto-deploy:** Cron job runs daily at 00:00 (`0 0 * * * /home/hexo/deploy-hexo.sh`), pulling `domestic` branch and regenerating.
- **Manual deploy:** `ssh hexo "cd youyoulyz.github.io && hexo clean && hexo generate && hexo deploy"`
- Deploy script: `/home/hexo/deploy-hexo.sh`
- Logs: `/home/hexo/hexo-deploy.log`

## Project Structure

```
├── _config.yml           # Main Hexo configuration (branch-specific)
├── _config.landscape.yml # Landscape theme overrides
├── package.json          # Node.js dependencies and scripts
├── scaffolds/            # Post/page templates
│   ├── draft.md
│   ├── page.md
│   └── post.md
├── source/
│   ├── CNAME             # Custom domain (youyou.moe on domestic)
│   ├── _posts/           # Blog posts (Markdown)
│   └── images/           # Post images/assets
├── themes/               # Theme files
├── .github/workflows/    # GitHub Actions CI/CD
│   └── pages.yml         # Pages deployment workflow
├── AGENTS.md             # Agent collaboration guidelines (Chinese)
└── BRANCHES.md           # Branch documentation (Chinese)
```

## Pre-Commit Checklist

**Before every commit, all checks MUST pass:**

### 1. Fact Verification
- Verify all data, versions, commands, and file paths referenced in new/modified content using `git log`, `grep`, `read_file` — **never rely on memory or assumptions**
- For **every new or modified link (URL)**, verify it is reachable and accurate (use `web_fetch` or manual judgment)
- Technical descriptions must match the actual code/configuration

### 2. Privacy Redaction
- Redact the author's **real name** — use username or placeholder
- Redact **tokens, API keys, secrets, passwords** — never commit any visible credentials
- Redact **specific IP addresses** — use `<服务器IP>` placeholder
- Redact **internal network addresses** — `192.168.x.x`, `10.x.x.x` → use placeholders
- **ICP filing number** — only in domestic theme config, never in blog posts

### 3. Commit Style
- Conventional Commits: `type: description`
- Common types: `feat:` (new feature/post), `docs:` (documentation), `fix:` (bug fix)
- Description should focus on **why**, not restate what was done

## Writing Conventions

1. **Redact IP addresses** — use `<服务器IP>` placeholder
2. **Redact usernames and passwords** — never include credentials
3. **Redact internal network addresses** — `192.168.x.x`, `10.x.x.x` → use placeholders
4. **ICP filing number** — only in domestic theme config, never in blog posts

## Config Differences Between Branches

| Setting | `main` | `domestic` |
|---------|--------|------------|
| `title` | 代码与生活 | youyoulyz的博客 \| 上云就上悠悠云 |
| `url` | `http://youyoulyz.github.io` | `http://youyou.moe` |

## GitHub Actions

The `.github/workflows/pages.yml` workflow triggers on pushes to `main`:
- Uses `ubuntu-latest` runner
- Node.js 22
- npm dependency caching via `actions/cache`
- Deploys via `actions/deploy-pages@v4` with GitHub Pages write permissions
