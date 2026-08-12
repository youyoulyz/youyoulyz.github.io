---
title: "OpenCode Go Relay：让 Claude Code 和 Codex 共用 DeepSeek V4 Flash 的超低价模型"
date: 2026-08-12 21:00:00
tags:
  - OpenCode Go
  - DeepSeek V4 Flash
  - Claude Code
  - Codex
  - 协议转换
  - AI Agent
  - 开源项目
categories:
  - AI 开发
  - 开源项目
---

> 一个零依赖、单文件的协议中转站：让 Claude Code（Anthropic Messages 协议）和 Codex（OpenAI Responses 协议）都能用上 OpenCode Go 订阅里的 DeepSeek V4 Flash——把超低价的模型，接到最贵的客户端上。

<!--more-->

## 背景：模型很便宜，客户端很挑剔

OpenCode Go 的订阅里带了一批 OpenAI 兼容模型，其中默认的 `deepseek-v4-flash` 价格低得离谱，日常写代码、改 bug、跑长任务的性价比极高。但它只暴露一个 OpenAI 兼容端点（`https://opencode.ai/zen/go/v1`），背后的模型只讲 `chat/completions` 这一种协议。

问题出在客户端上：

- **Claude Code** 只讲 Anthropic Messages 协议，直接指过去根本没法用。而 Claude Code 本身是很多人最顺手的 Agent 客户端，配上 Anthropic 官方模型的价格又让人肉疼。
- **Codex** 其实可以直接指向上游（`wire_api = "chat"`，不需要中转），但直接连意味着每个客户端都要单独配 key 和模型名，部署点也散。

于是就有了这个项目：一座把"贵客户端"和"便宜模型"接起来的桥。

## 项目简介

[opencode-go-relay](https://github.com/youyoulyz/opencode-go-relay) 是一个零依赖的协议中转站：

- **纯 Python 标准库**，整个项目就是 `relay.py` 一个文件，没有任何第三方依赖，Python 3.9+ 就能跑
- 在服务器上放一个进程，Claude Code / Codex 全都指向它，统一鉴权、统一模型别名、统一部署点
- 自带完整的自动化测试（mock 上游、零网络），改完代码跑一下就知道有没有坏
- MIT 协议开源

核心思路一句话：**OpenCode Go 只讲 OpenAI 协议，我们负责把其他协议的请求翻译成 OpenAI 协议，再原样转发上游。**

## 功能亮点

### 模型透传：不绑定 deepseek-v4-flash

先说一个容易忽略的点：**relay 不把模型写死**。客户端请求里带了什么模型，relay 就原样转发什么模型到 OpenCode Go 的 OpenAI 兼容 API——Anthropic 请求体里的 `model`、Responses 请求体里的 `model`、chat/completions 的 `model` 全部透传。OpenCode Go 那边挂了不止一个模型（比如 `glm-5.2` 等），客户端都能直接点名使用。

`deepseek-v4-flash` 只是**兜底**：只有当客户端没带 `model` 时才用它，`/v1/models` 广告的也是它。所以这个 relay 本质上是"协议翻译器"而不是"模型代理"——协议归协议，模型归模型，你客户端配了哪个模型，就用哪个模型。

### `/v1/messages`：Anthropic Messages ⇄ chat/completions

这是 Claude Code 接入的关键路径，翻译覆盖得比较完整：

- `system` 块、工具调用（tool use / tool results）、图片（URL 和 base64）、thinking 块（上游的 `reasoning_content` 会翻译成 thinking blocks）
- `tool_choice` 映射（`any` → `required`、指定工具等）
- 完整的 SSE 流式翻译：`message_start`、`content_block_delta`、`input_json_delta`、`message_delta`、`message_stop`

也就是说 Claude Code 的工具调用、流式输出、思考过程这些核心体验都不会丢。

### `/v1/responses`：OpenAI Responses ⇄ chat/completions

Codex 走这条路径（或者走下面的透传路径）：

- `instructions` → `system`，`function_call` / `function_call_output` 等 items 的翻译
- `max_output_tokens` → `max_tokens`
- SSE 翻译：`response.created`、`output_text.delta`、`function_call_arguments.delta`、`response.completed`

### 其他端点

| 路径 | 作用 |
|------|------|
| `POST /v1/chat/completions` | OpenAI 原生透传（仅鉴权，客户端指定的模型原样转发） |
| `GET /v1/models` | 模型列表（默认广告 `deepseek-v4-flash`） |
| `GET /healthz` | 健康检查（无需鉴权） |

### 两种鉴权模式

- **服务器单 key 模式**：设置 `OPENCODE_GO_API_KEY`，所有请求共用这一个 key，key 只存在服务器上。
- **按请求取 key 模式（推荐给 Claude Code）**：不设置 `OPENCODE_GO_API_KEY`，relay 从每个请求的 `Authorization: Bearer <key>` 或 `x-api-key: <key>` 里取 key 并原样转发，**服务器不保存任何 key**。Claude Code 的 `ANTHROPIC_AUTH_TOKEN` 本来就是要发出去的凭证，直接复用它就行，客户端几乎零改动。

另外还可以设置 `RELAY_TOKEN` 作为额外闸门：客户端必须同时带上自己的访问 token，防止 relay 被白嫖。启动时如果没设置，relay 会大声警告。

## 快速开始

```bash
# 在服务器上
export OPENCODE_GO_API_KEY="..."     # 你的 OpenCode Go key（只在服务器上）
export RELAY_TOKEN="change-me"       # 客户端访问 token，强烈建议设置
python3 relay.py                     # 监听 0.0.0.0:8787
```

本地验证（mock 上游，不需要网络也不需要真实 key）：

```bash
python3 test_relay.py
```

环境变量一览（常用部分）：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OPENCODE_GO_API_KEY` | 空 | 服务器单 key 模式；为空则走按请求取 key |
| `RELAY_TOKEN` | 空 | 客户端访问 token |
| `DEFAULT_MODEL` | `deepseek-v4-flash` | 客户端不指定模型时的兜底 |
| `UPSTREAM_BASE` | `https://opencode.ai/zen/go/v1` | 上游地址 |
| `HOST` / `PORT` | `0.0.0.0` / `8787` | 监听地址 |
| `STREAM_OPTIONS` | `1` | 是否向上游发送 `stream_options.include_usage` |
| `REQUEST_TIMEOUT` | `600` | 上游请求超时（秒） |

## Claude Code 接入

```bash
export ANTHROPIC_BASE_URL="http://<服务器IP>:8787"
export ANTHROPIC_AUTH_TOKEN="<RELAY_TOKEN>"
export ANTHROPIC_MODEL="deepseek-v4-flash"
claude
```

`ANTHROPIC_MODEL` 换成 OpenCode Go 上任意模型都行（比如 `glm-5.2`），relay 会原样转发，不限定只用 v4 flash。

或者写进 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://<服务器IP>:8787",
    "ANTHROPIC_AUTH_TOKEN": "<RELAY_TOKEN>",
    "ANTHROPIC_MODEL": "deepseek-v4-flash"
  }
}
```

如果走按请求取 key 模式，`ANTHROPIC_AUTH_TOKEN` 填你自己的 OpenCode Go key 就行，服务器端什么都不要存。

## Codex 接入

不想折腾的话，Codex 可以直接指上游，**完全不需要 relay**：

```toml
# ~/.codex/config.toml
[model_providers.opencode]
name = "OpenCode Go"
base_url = "https://opencode.ai/zen/go/v1"
env_key = "OPENCODE_GO_API_KEY"
wire_api = "chat"
```

```bash
export OPENCODE_GO_API_KEY="..."
codex --provider opencode --model deepseek-v4-flash
```

同样，`--model` 也可以指定其他模型。

但如果想统一鉴权、统一模型别名，也可以走 relay（`wire_api = "chat"` 和 `wire_api = "responses"` 都支持）：

```toml
[model_providers.opencode-relay]
name = "OpenCode Go (relay)"
base_url = "http://<服务器IP>:8787/v1"
env_key = "RELAY_TOKEN"
wire_api = "chat"
```

## 部署与安全

仓库里带了 systemd 示例（`opencode-go-relay.service`，用 `EnvironmentFile=` 把 key 放在单元文件之外），还有一篇完整的阿里云 ECS 部署文档：用户级 Python（uv 安装 3.12）、nginx HTTPS + 自定义端口 + 路径前缀反向代理、按请求取 key 模式、安全组放行，以及国内服务器部署的备案合规提示。

安全上几个要点：

- **`RELAY_TOKEN` 一定要设**（或严格限制来源 IP），否则任何知道地址的人都能借用你的 relay
- relay 本身是纯 HTTP，暴露到不可信网络前必须在前面套 TLS（Caddy / nginx / Traefik）
- 按请求取 key 模式下服务器不存任何 key，客户端 key 走 HTTPS 传输
- 内置了 64MB 请求体上限、最大并发控制（超出排队，背压）、连接读超时（防慢连接拖死进程）
- 上游 User-Agent 做了处理，默认用浏览器 UA，避免被边缘节点（比如 Cloudflare）拦掉 urllib 的默认 UA

## 测试

`test_relay.py` 会启动一个 mock 上游，在完全不联网的情况下验证六条路径：鉴权、模型列表、Anthropic 非流式/流式、Responses 非流式/流式、chat 透传。改完协议翻译逻辑跑一遍，心里就踏实了。

## 结语

DeepSeek V4 Flash 这类模型的低价让"AI 编程"这件事的成本结构彻底变了——但前提是你能把它接到自己顺手的客户端上。这个项目就是干这件事的：协议翻译的部分我做完了，剩下的就是把它部署到你的服务器，然后把环境变量指过去。

项目地址：[github.com/youyoulyz/opencode-go-relay](https://github.com/youyoulyz/opencode-go-relay)，欢迎 star、提 issue、顺手改 bug。
