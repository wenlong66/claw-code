# Claw Code 使用指南

本指南覆盖 `rust/` 下当前的 Rust 工作区以及 `claw` CLI 二进制。如果你是新手，请先把 doctor 健康检查作为第一次运行：启动 `claw`，然后运行 `/doctor`。

## 快速健康检查

在运行 prompt、会话或自动化之前，请先执行：

```bash
cd rust
cargo build --workspace
./target/debug/claw
# REPL 内的第一条命令
/doctor
```

`/doctor` 是内置的设置与预检诊断。保存会话后，你可以用 `./target/debug/claw --resume latest /doctor` 重新运行它。

## 前置条件

- Rust 工具链和 `cargo`
- 以下二者之一：
  - `ANTHROPIC_API_KEY`，用于直接 API 访问
  - `ANTHROPIC_AUTH_TOKEN`，用于 bearer-token 认证
- 可选：当目标是代理或本地服务时使用 `ANTHROPIC_BASE_URL`

## 安装 / 构建工作区

```bash
cd rust
cargo build --workspace
```

调试构建后，CLI 二进制位于 `rust/target/debug/claw`。请把上面的 doctor 检查作为构建后的第一步。

## 快速开始

### 首次运行的 doctor 检查

```bash
cd rust
./target/debug/claw
/doctor
```

也可以直接运行 doctor，并用 JSON 输出方便脚本处理：

```bash
cd rust
./target/debug/claw doctor --output-format json
```

**注意：** 诊断动词（`doctor`、`status`、`sandbox`、`version`）都支持 `--output-format json`，用于机器可读输出。像 `--json` 这样的无效后缀参数现在会在解析阶段被拒绝，而不会继续流向 prompt 分发。

### 初始化仓库

使用 `.claw` 配置、`.claw.json`、`.gitignore` 条目以及 `CLAUDE.md` 指南文件来设置一个新仓库：

```bash
cd /path/to/your/repo
./target/debug/claw init
```

文本模式（人类可读）会显示工件创建摘要、项目路径和后续步骤。它是幂等的——在同一仓库中多次运行时，已经创建的文件会标记为“skipped”。

用于脚本的 JSON 模式：
```bash
./target/debug/claw init --output-format json
```

返回结构化输出，包含 `project_path`、`created[]`、`updated[]`、`skipped[]` 数组（每个工件各一个），以及带有每个文件 `name` 和机器稳定 `status` 标签的 `artifacts[]`。旧的 `message` 字段保留向后兼容。

**为什么结构化字段重要：** claws 可以在不靠字符串匹配自然语言的情况下检测每个工件的状态（`created`、`updated`、`skipped`）。请使用 `created[]`、`updated[]` 和 `skipped[]` 数组来做条件后续逻辑（例如：只有在确实新建了文件时才提交，而不仅仅是更新了文件）。

### 交互式 REPL

```bash
cd rust
./target/debug/claw
```

### 一次性 prompt

```bash
cd rust
./target/debug/claw prompt "summarize this repository"
```

### 简写 prompt 模式

```bash
cd rust
./target/debug/claw "explain rust/crates/runtime/src/lib.rs"
```

### 用于脚本的 JSON 输出

```bash
cd rust
./target/debug/claw --output-format json prompt "status"
```

### 查看 worker 状态

`claw state` 命令会读取 `.claw/worker-state.json`，该文件由交互式 REPL 或一次性 prompt 在 worker 执行任务时写入。文件包含 worker ID、会话引用、模型和权限模式。

前提：你必须先在仓库中至少运行一次 `claw`（交互式 REPL）或 `claw prompt <text>`，才能生成 worker state 文件。

```bash
cd rust
./target/debug/claw state
```

JSON 模式：
```bash
./target/debug/claw state --output-format json
```

如果你在任何 worker 还没执行前就运行 `claw state`，会看到一条有帮助的错误：
```
error: no worker state file found at .claw/worker-state.json
  Hint: worker state is written by the interactive REPL or a non-interactive prompt.
  Run:   claw               # start the REPL (writes state on first turn)
  Or:    claw prompt <text> # run one non-interactive turn
  Then rerun: claw state [--output-format json]
```

## 高级斜杠命令（仅交互式 REPL）

这些命令只能在交互式 REPL（无参数启动的 `claw`）中使用。它们为助手扩展了工作区分析、规划和导航能力。

### `/ultraplan` — 通过多步推理做深度规划

**用途：** 用扩展推理把复杂任务拆成多个步骤。

```bash
# 启动 REPL
claw

# 在 REPL 中
/ultraplan refactor the auth module to use async/await
/ultraplan design a caching layer for database queries
/ultraplan analyze this module for performance bottlenecks
```

输出：一个包含编号步骤、每一步理由以及预期结果的结构化计划。当你希望助手在编码前先详细思考问题时，用这个命令。

### `/teleport` — 跳转到文件或符号

**用途：** 按名称快速跳转到文件、函数、类或结构体。

```bash
# 跳转到符号
/teleport UserService
/teleport authenticate_user
/teleport RequestHandler

# 跳转到文件
/teleport src/auth.rs
/teleport crates/runtime/lib.rs
/teleport ./ARCHITECTURE.md
```

输出：文件内容，并高亮请求的符号，或直接完整加载文件。适合在不手动浏览目录的情况下探索代码库。如果存在多个匹配项，助手会显示最相关的候选项。

### `/bughunter` — 扫描潜在 bug 和问题

**用途：** 分析代码中的常见陷阱、反模式和潜在 bug。

```bash
# 扫描整个工作区
/bughunter

# 扫描指定目录或文件
/bughunter src/handlers
/bughunter rust/crates/runtime
/bughunter src/auth.rs
```

输出：带解释的可疑模式列表（例如“未检查的 unwrap()”、“潜在竞态条件”、“缺少错误处理”）。每个发现都包含文件、行号和建议修复方式。可作为完整代码审查前的第一步。

## 模型与权限控制

```bash
cd rust
./target/debug/claw --model sonnet prompt "review this diff"
./target/debug/claw --permission-mode read-only prompt "summarize Cargo.toml"
./target/debug/claw --permission-mode workspace-write prompt "update README.md"
./target/debug/claw --allowedTools read,glob "inspect the runtime crate"
```

支持的权限模式：

- `read-only`
- `workspace-write`
- `danger-full-access`

CLI 当前支持的模型别名：

- `opus` → `claude-opus-4-6`
- `sonnet` → `claude-sonnet-4-6`
- `haiku` → `claude-haiku-4-5-20251213`

## 认证

### API key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

### OAuth

```bash
cd rust
export ANTHROPIC_AUTH_TOKEN="anthropic-oauth-or-proxy-bearer-token"
```

### 各环境变量该放什么

`claw` 接受两个 Anthropic 凭证环境变量，而且它们**不可互换**——Anthropic 期望的 HTTP header 会因凭证形态不同而不同。把错误的值放进错误的槽位，是我们最常见的 401 来源。

| 凭证形态 | 环境变量 | HTTP header | 常见来源 |
|---|---|---|---|
| `sk-ant-*` API key | `ANTHROPIC_API_KEY` | `x-api-key: sk-ant-...` | [console.anthropic.com](https://console.anthropic.com) |
| OAuth access token（不透明） | `ANTHROPIC_AUTH_TOKEN` | `Authorization: Bearer ...` | 生成 bearer token 的 Anthropic 兼容代理或 OAuth 流程 |
| OpenRouter key (`sk-or-v1-*`) | `OPENAI_API_KEY` + `OPENAI_BASE_URL=https://openrouter.ai/api/v1` | `Authorization: Bearer ...` | [openrouter.ai/keys](https://openrouter.ai/keys) |

**为什么这很重要：** 如果你把 `sk-ant-*` key 粘到 `ANTHROPIC_AUTH_TOKEN` 里，Anthropic 的 API 会返回 `401 Invalid bearer token`，因为 `sk-ant-*` key 不能通过 Bearer header 认证。修复方法就是一行环境变量交换——把 key 移到 `ANTHROPIC_API_KEY`。最近的 `claw` 构建会检测这种特定形态（401 + Bearer 槽位里出现 `sk-ant-*`）并在错误信息里追加一个指向修复方法的提示。

**如果你指的是别的提供方：** 如果 `claw` 报告缺少 Anthropic 凭证，但你已经导出了 `OPENAI_API_KEY`、`XAI_API_KEY` 或 `DASHSCOPE_API_KEY`，那你大概率只是忘了给模型名加上提供方路由前缀。使用 `--model openai/gpt-4.1-mini`（OpenAI 兼容 / OpenRouter / Ollama）、`--model grok`（xAI）或 `--model qwen-plus`（DashScope），前缀路由器就会不管环境凭证如何，选对后端。错误信息现在也会包含一个提示，指出检测到的 env var。

## 本地模型

`claw` 可以通过 Anthropic 兼容或 OpenAI 兼容端点连接本地服务和提供方网关。对 Anthropic 兼容服务使用 `ANTHROPIC_BASE_URL` 配合 `ANTHROPIC_AUTH_TOKEN`，对 OpenAI 兼容服务使用 `OPENAI_BASE_URL` 配合 `OPENAI_API_KEY`。OAuth 只适用于 Anthropic，所以当设置了 `OPENAI_BASE_URL` 时，应使用 API key 风格的认证，而不是 `claw login`。

### Anthropic 兼容端点

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:8080"
export ANTHROPIC_AUTH_TOKEN="local-dev-token"

cd rust
./target/debug/claw --model "claude-sonnet-4-6" prompt "reply with the word ready"
```

### OpenAI 兼容端点

```bash
export OPENAI_BASE_URL="http://127.0.0.1:8000/v1"
export OPENAI_API_KEY="local-dev-token"

cd rust
./target/debug/claw --model "qwen2.5-coder" prompt "reply with the word ready"
```

### Ollama

```bash
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
unset OPENAI_API_KEY

cd rust
./target/debug/claw --model "llama3.2" prompt "summarize this repository in one sentence"
```

### OpenRouter

```bash
export OPENAI_BASE_URL="https://openrouter.ai/api/v1"
export OPENAI_API_KEY="sk-or-v1-..."

cd rust
./target/debug/claw --model "openai/gpt-4.1-mini" prompt "summarize this repository in one sentence"
```

### Alibaba DashScope（Qwen）

对于通过 Alibaba 原生 DashScope API 使用的 Qwen 模型（比 OpenRouter 限流更宽松）：

```bash
export DASHSCOPE_API_KEY="sk-..."

cd rust
./target/debug/claw --model "qwen/qwen-max" prompt "hello"
# 或直接写：
./target/debug/claw --model "qwen-plus" prompt "hello"
```

以 `qwen/` 或 `qwen-` 开头的模型名会自动路由到 DashScope 兼容模式端点（`https://dashscope.aliyuncs.com/compatible-mode/v1`）。你**不需要**设置 `OPENAI_BASE_URL` 或取消设置 `ANTHROPIC_API_KEY` —— 模型前缀会优先于环境凭证嗅探。

推理变体（`qwen-qwq-*`、`qwq-*`、`*-thinking`）会在请求发出前自动去掉 `temperature`/`top_p`/`frequency_penalty`/`presence_penalty`（这些参数会被推理模型拒绝）。

## 支持的提供方与模型

`claw` 内置了三个提供方后端。提供方会先按模型名自动选择，如果模型名没有明确路由，再回退到环境中的凭证。

### 提供方矩阵

| 提供方 | 协议 | 认证 env var | Base URL env var | 默认 base URL |
|---|---|---|---|---|
| **Anthropic**（直连） | Anthropic Messages API | `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN` | `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` |
| **xAI** | OpenAI 兼容 | `XAI_API_KEY` | `XAI_BASE_URL` | `https://api.x.ai/v1` |
| **OpenAI-compatible** | OpenAI Chat Completions | `OPENAI_API_KEY` | `OPENAI_BASE_URL` | `https://api.openai.com/v1` |
| **DashScope**（Alibaba） | OpenAI 兼容 | `DASHSCOPE_API_KEY` | `DASHSCOPE_BASE_URL` | `https://dashscope.aliyuncs.com/compatible-mode/v1` |

OpenAI 兼容后端也充当 **OpenRouter**、**Ollama** 以及任何说 OpenAI `/v1/chat/completions` 协议的服务的网关——只需把 `OPENAI_BASE_URL` 指向该服务即可。

**模型名前缀路由：** 如果模型名以 `openai/`、`gpt-`、`qwen/` 或 `qwen-` 开头，就会按前缀选择提供方，而不管设置了哪些 env var。这样可以避免在环境里同时存在多个凭证时误路由到 Anthropic。

### 已测试的模型与别名

这些是内置别名表中已登记、且已知 token 上限的模型：

| 别名 | 解析后的模型名 | 提供方 | 最大输出 token | 上下文窗口 |
|---|---|---|---|---|
| `opus` | `claude-opus-4-6` | Anthropic | 32 000 | 200 000 |
| `sonnet` | `claude-sonnet-4-6` | Anthropic | 64 000 | 200 000 |
| `haiku` | `claude-haiku-4-5-20251213` | Anthropic | 64 000 | 200 000 |
| `grok` / `grok-3` | `grok-3` | xAI | 64 000 | 131 072 |
| `grok-mini` / `grok-3-mini` | `grok-3-mini` | xAI | 64 000 | 131 072 |
| `grok-2` | `grok-2` | xAI | — | — |

任何不匹配别名的模型名都会原样透传。你可以据此使用 OpenRouter 的模型 slug（`openai/gpt-4.1-mini`）、Ollama 标签（`llama3.2`），或完整的 Anthropic 模型 ID（`claude-sonnet-4-20250514`）。

### 用户自定义别名

你可以在任何设置文件中添加自定义别名（`~/.claw/settings.json`、`.claw/settings.json` 或 `.claw/settings.local.json`）：

```json
{
  "aliases": {
    "fast": "claude-haiku-4-5-20251213",
    "smart": "claude-opus-4-6",
    "cheap": "grok-3-mini"
  }
}
```

本地项目设置会覆盖用户级设置。别名还会经过内置表解析，所以 `"fast": "haiku"` 也可以。

### 提供方检测如何工作

1. 如果解析后的模型名以 `claude` 开头 → Anthropic。
2. 如果以 `grok` 开头 → xAI。
3. 否则，`claw` 会检查当前设置了哪个凭证：先看 `ANTHROPIC_API_KEY`/`ANTHROPIC_AUTH_TOKEN`，再看 `OPENAI_API_KEY`，最后看 `XAI_API_KEY`。
4. 如果都不匹配，就默认 Anthropic。

## FAQ

### Codex 是什么？

“codex” 这个名字会出现在 Claw Code 生态里，但它**并不是**指 OpenAI Codex（代码生成模型）。在本项目里，它的含义如下：

- **`oh-my-codex`（OmX）** 是建立在 `claw` 之上的工作流和插件层。它提供规划模式、并行多智能体执行、通知路由以及其他自动化功能。参见 [PHILOSOPHY.md](./PHILOSOPHY.md) 和 [oh-my-codex 仓库](https://github.com/Yeachan-Heo/oh-my-codex)。
- **`.codex/` 目录**（例如 `.codex/skills`、`.codex/agents`、`.codex/commands`）是旧的查找路径，`claw` 仍会在主要的 `.claw/` 目录之外一并扫描它们。
- **`CODEX_HOME`** 是一个可选环境变量，用来指定用户级 skill 和 command 查找的自定义根目录。

`claw` **不**支持 OpenAI Codex 会话、Codex CLI，或 Codex 会话导入/导出。如果你需要使用 OpenAI 模型（如 GPT-4.1），请按上面的 [OpenAI 兼容端点](#openai-compatible-endpoint) 和 [OpenRouter](#openrouter) 部分配置 OpenAI-compatible 提供方。

## HTTP 代理支持

`claw` 在向 Anthropic、OpenAI 和 xAI 兼容端点发出外部请求时，会遵守标准的 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 环境变量（大小写形式都接受）。在启动 CLI 之前设置好它们，底层的 `reqwest` 客户端会自动配置。

### 环境变量

```bash
export HTTPS_PROXY="http://proxy.corp.example:3128"
export HTTP_PROXY="http://proxy.corp.example:3128"
export NO_PROXY="localhost,127.0.0.1,.corp.example"

cd rust
./target/debug/claw prompt "hello via the corporate proxy"
```

### 程序化 `proxy_url` 配置选项

作为按协议环境变量的替代，`ProxyConfig` 类型提供一个 `proxy_url` 字段，可作为 HTTP 与 HTTPS 流量共用的统一代理。设置 `proxy_url` 后，它会优先于单独的 `http_proxy` 和 `https_proxy` 字段。

```rust
use api::{build_http_client_with, ProxyConfig};

// 来自单个统一 URL（配置文件、CLI 参数等）
let config = ProxyConfig::from_proxy_url("http://proxy.corp.example:3128");
let client = build_http_client_with(&config).expect("proxy client");

// 或者直接设置字段，同时配合 NO_PROXY
let config = ProxyConfig {
    proxy_url: Some("http://proxy.corp.example:3128".to_string()),
    no_proxy: Some("localhost,127.0.0.1".to_string()),
    ..ProxyConfig::default()
};
let client = build_http_client_with(&config).expect("proxy client");
```

### 说明

- 当同时设置了 `HTTPS_PROXY` 和 `HTTP_PROXY` 时，安全代理会用于 `https://` URL，普通代理会用于 `http://` URL。
- `proxy_url` 是一个统一替代方案：设置后，它会同时作用于 `http://` 和 `https://` 目标，并覆盖按协议字段。
- `NO_PROXY` 接受以逗号分隔的主机后缀列表（例如 `.corp.example`）以及 IP 字面量。
- 空值会被视为未设置，所以在 shell 里留着 `HTTPS_PROXY=""` 并不会启用代理。
- 如果代理 URL 无法解析，`claw` 会回退到直连（无代理）客户端，这样现有工作流仍可继续；如果你本来期望请求走隧道，请再检查一下 URL。

## 常用操作命令

```bash
cd rust
./target/debug/claw status
./target/debug/claw sandbox
./target/debug/claw agents
./target/debug/claw mcp
./target/debug/claw skills
./target/debug/claw system-prompt --cwd .. --date 2026-04-04
```

## 会话管理

REPL 的回合会持久化到当前工作区下的 `.claw/sessions/`。

```bash
cd rust
./target/debug/claw --resume latest
./target/debug/claw --resume latest /status /diff
```

交互式命令还包括 `/help`、`/status`、`/cost`、`/config`、`/session`、`/model`、`/permissions` 和 `/export`。

## 配置文件解析顺序

运行时配置按以下顺序加载，后面的条目会覆盖前面的条目：

1. `~/.claw.json`
2. `~/.config/claw/settings.json`
3. `<repo>/.claw.json`
4. `<repo>/.claw/settings.json`
5. `<repo>/.claw/settings.local.json`

## mock parity harness

该工作区包含一个确定性的 Anthropic-compatible mock service 和 parity harness。

```bash
cd rust
./scripts/run_mock_parity_harness.sh
```

手动启动 mock 服务：

```bash
cd rust
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

## 验证

```bash
cd rust
cargo test --workspace
```

## 工作区概览

当前 Rust crates：

- `api`
- `commands`
- `compat-harness`
- `mock-anthropic-service`
- `plugins`
- `runtime`
- `rusty-claude-cli`
- `telemetry`
- `tools`
