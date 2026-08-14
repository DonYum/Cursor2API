# cursor2api

<p align="center">
  <strong>面向 Cursor Composer 的 OpenAI / Anthropic 兼容 API 网关</strong><br>
  同时支持 <code>Responses</code> · <code>Messages</code> · <code>Chat Completions</code> 三种协议
</p>

<p align="center">
  <a href="https://github.com/NGLSG/cursor2api">GitHub</a>
  ·
  <a href="#快速部署">快速部署</a>
  ·
  <a href="#api">API 文档</a>
  ·
  <a href="#客户端接入">客户端接入</a>
</p>

> [!NOTE]
> 本项目仅供技术研究与学习交流。使用时请务必遵循 Cursor 官方使用条款及当地法律法规。

## 项目简介

**cursor2api** 是一个轻量级 API 网关，把你的 **Cursor API Key**（`crsr_…`）转成标准 HTTP 接口。客户端只需配置 `baseUrl` + `apiKey`，即可接入 Cursor 上的 Composer、Claude、GPT 等模型，无需为每个工具单独写适配层。

与仅支持单一 OpenAI Chat 格式的早期版本不同，**当前版本完整支持三种主流协议**：

| 协议 | 端点 | 典型客户端 |
| :-- | :-- | :-- |
| **Responses** | `POST /v1/responses` | **Codex**、Cherry Studio |
| **Messages** | `POST /v1/messages` | **Claude Code**（Anthropic 兼容） |
| **Chat Completions** | `POST /v1/chat/completions` | OpenCode、Cline、Continue、VS Code 插件等 |

另有 `GET /v1/models`（动态模型列表）和 `GET /health`（健康检查）。Codex / Claude Code 的工具调用经 `@cursor/sdk` 本地 Bridge 转发。

支持 **Windows / Linux / macOS** 本地部署；可选 [Windows 托盘应用](desktop/README.md)（仅 Windows）；也保留 Cloudflare Worker 远程部署路径。

### 目录结构

```
cursor2api/
├── sidecar/          # 跨平台 API 网关（Responses / Messages / Chat）— Linux 部署核心
├── scripts/          # SDK Bridge、server.mjs 等
├── worker/           # 协议转换逻辑 + Cloudflare Worker
├── server.mjs        # 本地启动 CLI（start / stop / claude / codex）
├── desktop/          # 可选：Windows Tauri 托盘应用（Linux 不需要）
└── ...
```

| 目录 | 平台 | Linux 服务器需要？ |
| :-- | :-- | :-- |
| `sidecar/` + `server.mjs` | 全平台 | ✅ |
| `scripts/cursor-sdk-local-agent-bridge.mjs` | 全平台 | ✅ |
| `desktop/` | 仅 Windows | ❌ |
| `worker/`（Cloudflare） | 云端 | 可选 |

### 项目架构

```mermaid
flowchart LR
    classDef access fill:#e1f5fe,stroke:#01579b
    classDef core fill:#fff3e0,stroke:#e65100
    classDef bridge fill:#f3e5f5,stroke:#4a148c
    classDef upstream fill:#fce4ec,stroke:#880e4f

    subgraph Access["接入域"]
        direction LR
        Codex["Codex<br/>Responses"]
        Claude["Claude Code<br/>Messages"]
        ChatClients["OpenCode / Cline<br/>Chat Completions"]
    end

    subgraph Core["网关核心域"]
        direction LR
        Sidecar["Sidecar 服务<br/>Bun · 协议转换"]
        OpenAI["worker/openai.ts<br/>Responses · Chat"]
        Anthropic["anthropic.ts<br/>Messages"]
        OpenAI --- Anthropic
        Sidecar --> OpenAI
        Sidecar --> Anthropic
    end

    subgraph Bridge["SDK Bridge 域"]
        NodeBridge["Node Bridge<br/>@cursor/sdk"]
    end

    Upstream["🌐 Cursor 后端"]

    Codex --> Sidecar
    Claude --> Sidecar
    ChatClients --> Sidecar
    Sidecar --> NodeBridge
    NodeBridge --> Upstream

    class Codex,Claude,ChatClients access
    class Sidecar,OpenAI,Anthropic core
    class NodeBridge bridge
    class Upstream upstream
```

Sidecar 负责三种协议的入站解析与出站整形；SDK Bridge 用官方 `@cursor/sdk` 与 Cursor 后端通信（gRPC / HTTP2）。**只需你的 Cursor Key，无需额外后端密钥。**

### 核心能力

| 模块 | 能力 |
| :-- | :-- |
| **接口** | Responses、Chat Completions、Anthropic Messages；流式 SSE + 非流式 JSON |
| **客户端** | Codex、Claude Code、Cherry Studio，以及 OpenAI / Anthropic 兼容 SDK |
| **模型** | `GET /v1/models` 动态拉取 Cursor 账号可用模型（Composer、Claude、GPT 等） |
| **工具** | Codex `exec` 等 Responses 工具、Claude Code 读写文件等 Messages 工具，经 SDK Bridge 转发 |
| **上下文** | Claude Code 1M 上下文（`claude-opus-5[1m]` 或 `anthropic-beta` 头） |
| **可靠性** | SDK 瞬时断连自动重试（最多 3 次）；Bridge 凭据定期刷新 |
| **部署** | 本地 sidecar + bridge；可选 Windows Tauri 托盘；Cloudflare Worker |

### 协议边界

三种协议独立入口、统一后端，客户端按自身能力选择对应端点：

| 协议 | 鉴权方式 | 流式 | 工具调用 | 适用场景 |
| :-- | :-- | :-- | :-- | :-- |
| **Responses** | `Authorization: Bearer crsr_…` | SSE | ✅ Codex 工具链 | Codex、Cherry Studio（Responses 模式） |
| **Messages** | `x-api-key: crsr_…` 或 Bearer | SSE | ✅ Claude Code 工具 | Claude Code CLI |
| **Chat Completions** | `Authorization: Bearer crsr_…` | SSE / JSON | 取决于客户端 | 通用 OpenAI 兼容 Agent |

## 快速部署

### 环境要求

| 依赖 | 版本 |
| :-- | :-- |
| Node.js | 20+（SDK Bridge **必须**用 Node） |
| Bun | 1.3+（Sidecar 服务） |
| Cursor 账号 | 已开通 API / Composer 权限 |

### 本地部署（推荐）

```bash
git clone https://github.com/NGLSG/cursor2api.git
cd cursor2api
npm ci   # 或 bun install

# 启动 Sidecar + SDK Bridge（后台）
node server.mjs start
# 输出 JSON：baseUrl / anthropicBaseUrl / pid

# 前台运行（调试）
node server.mjs start --foreground

# 停止
node server.mjs stop

# 查看状态
node server.mjs status
```

也可使用 npm 脚本：

```bash
npm run start:local
npm run stop:local
```

验证服务：

```bash
curl http://127.0.0.1:6718/health
export CURSOR_API_KEY="crsr_YOUR_KEY"
node server.mjs models
curl -H "Authorization: Bearer $CURSOR_API_KEY" http://127.0.0.1:6718/v1/models
```

<details>
<summary>手动分进程启动（高级）</summary>

```bash
# 终端 1 — SDK Bridge
export CURSOR_SDK_BRIDGE_HOST=127.0.0.1
export CURSOR_SDK_BRIDGE_PORT=6719
export CURSOR_SDK_BRIDGE_TOKEN=$(openssl rand -hex 16)
node scripts/cursor-sdk-local-agent-bridge.mjs

# 终端 2 — Sidecar API
export PORT=6718
export CURSOR_SDK_BRIDGE_URL=http://127.0.0.1:6719/sdk
bun run sidecar/server.ts
```

</details>

### 默认地址

| 用途 | 地址 |
| :-- | :-- |
| OpenAI 兼容（Codex / Chat） | `http://127.0.0.1:6718/v1` |
| Anthropic 兼容（Claude Code） | `http://127.0.0.1:6718` |
| 健康检查 | `http://127.0.0.1:6718/health` |

建议客户端使用 **`127.0.0.1`** 而非 `localhost`，避免 IPv6 解析导致连不上。

### 远程部署

**Cloudflare Worker**（仓库自带 `worker/` + D1）：

```bash
npm run deploy
```

**自建 VPS / Linux**：与本地相同，启动 sidecar + bridge 后用 systemd 或 Docker 守护进程即可。

## 客户端接入

### Codex（Responses 协议）

`~/.codex/config.toml`：

```toml
model = "composer-2.5"   # 或 /v1/models 返回的任意 id
model_provider = "cursorapi"

[model_providers.cursorapi]
name = "Cursor API"
base_url = "http://127.0.0.1:6718/v1"
wire_api = "responses"
env_key = "CODEX_API_KEY"
```

```bash
export CODEX_API_KEY="crsr_YOUR_KEY"
codex

# 或一键启动（需已配置 codex profile，默认 cursor6718）
node server.mjs codex
node server.mjs codex --profile cursor6718 -- "你的提示词"
```

**实机效果** — Codex 经 cursor2api 调用终端 `exec`、扫描代码库、输出结构化分析：

![Codex 经 cursor2api 调用 exec 工具并生成项目概览](docs/screenshots/codex-demo.png)

### Claude Code（Messages 协议）

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:6718"
export ANTHROPIC_API_KEY="crsr_YOUR_KEY"
claude

# 或一键启动（自动注入上述环境变量）
export CURSOR_API_KEY="crsr_YOUR_KEY"
node server.mjs claude
node server.mjs claude -- "你的提示词"
```

### OpenCode / Cline 等（Chat 协议）

| 配置项 | 值 |
| :-- | :-- |
| Base URL | `http://127.0.0.1:6718/v1` |
| API Key | `crsr_…` |
| Model | 从 `GET /v1/models` 选择 |

### Cherry Studio

| 配置项 | 值 |
| :-- | :-- |
| Base URL | `http://127.0.0.1:6718/v1` |
| API 类型 | OpenAI 兼容 / Responses API |
| API Key | `crsr_…` |

### 客户端对照表

| 客户端 | Base URL | 协议 |
| :-- | :-- | :-- |
| Codex | `http://127.0.0.1:6718/v1` | Responses |
| Claude Code | `http://127.0.0.1:6718` | Messages |
| Cherry Studio | `http://127.0.0.1:6718/v1` | Responses / Chat |
| OpenCode / Cline | `http://127.0.0.1:6718/v1` | Chat |

## 模型与路由

cursor2api **不使用固定模型清单**。`GET /v1/models` 会实时读取当前 Cursor 账号可用模型，不同账号、订阅等级可能返回不同结果。

常见模型示例（以实际 `/v1/models` 返回为准）：

| 模型 | 类型 | 网关接口能力 |
| :-- | :-- | :-- |
| `composer-2.5` / `composer-2.5-fast` | 对话 | Responses、Chat、Messages |
| `claude-opus-5` / `claude-sonnet-5` 等 | 对话 | Responses、Chat、Messages |
| `gpt-5.6-sol-max` 等 | 对话 | Responses、Chat、Messages |

客户端应以 **`GET /v1/models` 返回的当前可服务模型** 为准。

## API

推理接口使用 Cursor API Key：

```http
Authorization: Bearer crsr_YOUR_KEY
```

Anthropic 客户端也可使用：

```http
x-api-key: crsr_YOUR_KEY
```

| 方法 | 路径 | 用途 |
| :-- | :-- | :-- |
| `GET` | `/health` | 健康检查 |
| `GET` | `/v1/models` | 当前可服务模型列表 |
| `POST` | `/v1/responses` | Responses JSON / SSE（Codex） |
| `POST` | `/v1/chat/completions` | Chat Completions JSON / SSE |
| `POST` | `/v1/messages` | Anthropic Messages JSON / SSE（Claude Code） |
| `POST` | `/v1/messages/count_tokens` | Claude Code 预发送 token 估算 |

### 请求示例

**Responses（Codex）：**

```bash
curl http://127.0.0.1:6718/v1/responses \
  -H "Authorization: Bearer crsr_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "composer-2.5",
    "input": "分析这个项目结构",
    "stream": true
  }'
```

**Messages（Claude Code）：**

```bash
curl http://127.0.0.1:6718/v1/messages \
  -H "x-api-key: crsr_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-5",
    "max_tokens": 4096,
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

**Chat Completions（通用）：**

```bash
curl http://127.0.0.1:6718/v1/chat/completions \
  -H "Authorization: Bearer crsr_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "composer-2.5",
    "messages": [{"role": "user", "content": "你好"}],
    "stream": true
  }'
```

## 配置说明

本地进程管理 CLI（[`server.mjs`](server.mjs)）：

| 命令 | 说明 |
| :-- | :-- |
| `node server.mjs start [--port 6718] [--foreground]` | 启动 Sidecar + Bridge |
| `node server.mjs stop` | 停止后台进程 |
| `node server.mjs status` | 查看运行状态 |
| `node server.mjs models [--json]` | 列出可用模型 |
| `node server.mjs claude [-- ...]` | 注入 Anthropic 环境变量并启动 Claude Code |
| `node server.mjs codex [--profile NAME] [-- ...]` | 注入 CODEX_API_KEY 并启动 Codex |

Sidecar 与 Bridge 通过环境变量配置，参考 [`.env.example`](.env.example)：

| 变量 | 说明 | 默认值 |
| :-- | :-- | :-- |
| `PORT` | Sidecar 监听端口 | `8787`（脚本默认 `6718`） |
| `HOST` | 绑定地址 | `127.0.0.1` |
| `CURSOR_API_KEY` | 默认 Cursor Key（可选，客户端 Bearer 优先） | — |
| `CURSOR_SDK_BRIDGE_URL` | Bridge 地址 | — |
| `CURSOR_SDK_BRIDGE_TOKEN` | Bridge 鉴权 Token | — |
| `CURSOR_SDK_BRIDGE_HOST` | Bridge 绑定地址 | `127.0.0.1` |
| `CURSOR_SDK_BRIDGE_PORT` | Bridge 端口 | 随机 |

> [!TIP]
> SDK Bridge **必须用 Node 运行**（不能换 Bun）：`@cursor/sdk` 依赖 sqlite3 原生模块和 gRPC over HTTP/2。

运行时日志与进程状态：`~/.cursor2api/`（Windows 为 `%USERPROFILE%\.cursor2api\`）

## 常见问题

| 现象 | 处理 |
| :-- | :-- |
| 刚启动第一次请求失败 | SDK Bridge 冷启动约 10–15 秒，等一会重试 |
| 客户端连不上但 curl 正常 | 改用 `127.0.0.1`，不要用 `localhost` |
| `401 unauthorized` | Key 无效 / 已撤销，或启动后才设置 Key — 重启服务 |
| Codex 工具不执行 | 确认 `wire_api = "responses"` 且设置了 `CODEX_API_KEY` |
| Claude Code 404 | Base URL **不要**带 `/v1` |
| 偶发 `socket connection closed` | SDK 瞬时故障，服务自动重试最多 3 次 |
| Cherry Studio 校验报错 | 确保 SSE 错误事件格式正确（本版本已修复） |

## 生产检查

- Sidecar 默认绑定 `127.0.0.1`，仅本机可访问；公网暴露请前置反向代理 + HTTPS + 访问控制。
- **不要**将 `crsr_…` Key、`.dev.vars`、签名证书提交到仓库。
- Bridge Token 每次启动随机生成，仅 loopback 内共享。
- 远程 VPS 部署时限制防火墙，仅允许可信 IP 访问 API 端口。

## 开发验证

```bash
npm test              # vitest（worker + bridge）
npm run typecheck
npm run test:sidecar   # 或 cd sidecar && bun test
```

## 可选组件

| 组件 | 说明 |
| :-- | :-- |
| [Windows 托盘应用](desktop/README.md) | Tauri 2 系统托盘，默认端口 8787，Credential Manager 存 Key，一键配置 Agent |
| [Cloudflare Worker](worker/) | 远程多用户网关（需自行部署到 CF 账号） |

## 相关文档

- [Windows 托盘应用](desktop/README.md)
- [构建契约](desktop/BUILD_CONTRACT.md)
- [变更日志](CHANGELOG.md)
- [上游项目 composer-api](https://github.com/standardagents/composer-api)

## 致谢

基于 **[standardagents/composer-api](https://github.com/standardagents/composer-api)**（MIT）fork 并扩展。

**[NGLSG/cursor2api](https://github.com/NGLSG/cursor2api)** 新增：跨平台 sidecar 部署、**Responses / Messages / Chat 三协议**、Codex 工具转发、Claude Code Anthropic 适配、动态模型列表、Cherry Studio SSE 修复、可选 Windows 托盘应用。

Powered by [`@cursor/sdk`](https://www.npmjs.com/package/@cursor/sdk) 与 Cursor Composer 模型。

[LINUX DO](https://linux.do/)提供的交流社区
## License

[MIT](LICENSE)
