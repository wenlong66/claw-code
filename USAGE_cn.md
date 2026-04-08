# Claw Code 使用指南

本指南覆盖 `rust/` 下的当前 Rust 工作区以及 `claw` CLI 二进制。如果你是第一次接触，建议把 doctor 健康检查作为第一次运行：先启动 `claw`，然后运行 `/doctor`。

## 快速健康检查

在发送 prompts、开启 sessions 或自动化之前先运行：

```bash
cd rust
cargo build --workspace
./target/debug/claw
# 在 REPL 里输入的第一条命令
/doctor
```

`/doctor` 是内置的初始化和预检诊断。保存过 session 之后，可以用 `./target/debug/claw --resume latest /doctor` 重新运行。

## 前置条件

- Rust 工具链和 `cargo`
- 以下两者之一：
  - `ANTHROPIC_API_KEY`，用于直接 API 访问
  - `claw login`，用于基于 OAuth 的认证
- 可选：当你要对接代理或本地服务时使用 `ANTHROPIC_BASE_URL`

## 安装 / 构建工作区

```bash
cd rust
cargo build --workspace
```

调试构建完成后，CLI 二进制位于 `rust/target/debug/claw`。请把上面的 doctor 检查作为构建后的第一步。

## 快速上手

### 首次运行的 doctor 检查

```bash
cd rust
./target/debug/claw
/doctor
```

### 交互式 REPL

```bash
cd rust
./target/debug/claw
```

### 一次性 prompt

```bash
cd rust
./target/debug/claw prompt "概括这个仓库"
```

### 简写 prompt 模式

```bash
cd rust
./target/debug/claw "解释 rust/crates/runtime/src/lib.rs"
```

### 用于脚本的 JSON 输出

```bash
cd rust
./target/debug/claw --output-format json prompt "status"
```

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
./target/debug/claw login
./target/debug/claw logout
```

## 本地模型

`claw` 可以通过 Anthropic-compatible 或 OpenAI-compatible endpoint 连接本地服务器和 provider gateway。对 Anthropic-compatible 服务使用 `ANTHROPIC_BASE_URL` 配合 `ANTHROPIC_AUTH_TOKEN`，对 OpenAI-compatible 服务使用 `OPENAI_BASE_URL` 配合 `OPENAI_API_KEY`。OAuth 只适用于 Anthropic，所以当设置了 `OPENAI_BASE_URL` 时，应使用 API key 风格的认证，而不是 `claw login`。

### Anthropic-compatible endpoint

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:8080"
export ANTHROPIC_AUTH_TOKEN="local-dev-token"

cd rust
./target/debug/claw --model "claude-sonnet-4-6" prompt "reply with the word ready"
```

### OpenAI-compatible endpoint

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

## 支持的 Providers 与模型

`claw` 内置了三个 provider backend。模型名称会自动决定使用哪个 provider，如果无法判断，则回退到环境中可用的凭据。

### Provider 矩阵

| Provider | 协议 | 认证环境变量 | Base URL 环境变量 | 默认 Base URL |
|---|---|---|---|---|
| **Anthropic**（直连） | Anthropic Messages API | `ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN` 或 OAuth（`claw login`） | `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` |
| **xAI** | OpenAI-compatible | `XAI_API_KEY` | `XAI_BASE_URL` | `https://api.x.ai/v1` |
| **OpenAI-compatible** | OpenAI Chat Completions | `OPENAI_API_KEY` | `OPENAI_BASE_URL` | `https://api.openai.com/v1` |

OpenAI-compatible backend 也可作为 **OpenRouter**、**Ollama** 以及其他支持 OpenAI `/v1/chat/completions` 协议的服务入口——只要把 `OPENAI_BASE_URL` 指向对应服务即可。

### 已测试的模型与别名

以下模型在内置别名表中注册，并带有已知的 token 上限：

| 别名 | 解析后的模型名 | Provider | 最大输出 token | Context window |
|---|---|---|---|---|
| `opus` | `claude-opus-4-6` | Anthropic | 32 000 | 200 000 |
| `sonnet` | `claude-sonnet-4-6` | Anthropic | 64 000 | 200 000 |
| `haiku` | `claude-haiku-4-5-20251213` | Anthropic | 64 000 | 200 000 |
| `grok` / `grok-3` | `grok-3` | xAI | 64 000 | 131 072 |
| `grok-mini` / `grok-3-mini` | `grok-3-mini` | xAI | 64 000 | 131 072 |
| `grok-2` | `grok-2` | xAI | — | — |

任何不匹配别名的模型名都会原样透传。这就是你使用 OpenRouter model slug（例如 `openai/gpt-4.1-mini`）、Ollama tag（例如 `llama3.2`）或完整 Anthropic model ID（例如 `claude-sonnet-4-20250514`）的方式。

### 用户自定义别名

你可以在任意 settings 文件中添加自定义别名（`~/.claw/settings.json`、`.claw/settings.json` 或 `.claw/settings.local.json`）：

```json
{
  "aliases": {
    "fast": "claude-haiku-4-5-20251213",
    "smart": "claude-opus-4-6",
    "cheap": "grok-3-mini"
  }
}
```

本地项目设置会覆盖用户级设置。别名解析会经过内置表，所以 `"fast": "haiku"` 也可以工作。

### provider 检测方式

1. 如果解析后的模型名以 `claude` 开头 → Anthropic。
2. 如果它以 `grok` 开头 → xAI。
3. 否则，`claw` 依次检查哪些凭据已设置：先 `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN`，再 `OPENAI_API_KEY`，最后 `XAI_API_KEY`。
4. 如果都不匹配，则默认 Anthropic。

## 常见问题

### 那 Codex 呢？

“codex” 这个名字出现在 Claw Code 生态中，但它**不是**指 OpenAI Codex（代码生成模型）。在本项目里它的含义如下：

- **`oh-my-codex`（OmX）** 是构建在 `claw` 之上的工作流和插件层。它提供 planning mode、并行多 agent 执行、通知路由以及其他自动化能力。请参见 [PHILOSOPHY.md](./PHILOSOPHY.md) 和 [oh-my-codex 仓库](https://github.com/Yeachan-Heo/oh-my-codex)。
- **`.codex/` 目录**（例如 `.codex/skills`、`.codex/agents`、`.codex/commands`）是 legacy lookup 路径，`claw` 仍会和主 `.claw/` 目录一起扫描它们。
- **`CODEX_HOME`** 是一个可选环境变量，用于指向用户级 skill 和 command 查找的自定义根目录。

`claw` **不**支持 OpenAI Codex sessions、Codex CLI 或 Codex session 的导入 / 导出。如果你需要使用 OpenAI 模型（例如 GPT-4.1），请按上面的 [OpenAI-compatible endpoint](#openai-compatible-endpoint) 和 [OpenRouter](#openrouter) 小节配置 OpenAI-compatible provider。

## HTTP 代理支持

`claw` 在发起到 Anthropic、OpenAI 和 xAI-compatible endpoint 的请求时，会遵守标准的 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 环境变量（同时接受大写和小写写法）。在启动 CLI 前设置这些变量，底层 `reqwest` 客户端会自动完成配置。

### 环境变量

```bash
export HTTPS_PROXY="http://proxy.corp.example:3128"
export HTTP_PROXY="http://proxy.corp.example:3128"
export NO_PROXY="localhost,127.0.0.1,.corp.example"

cd rust
./target/debug/claw prompt "hello via the corporate proxy"
```

### 程序化 `proxy_url` 配置项

作为按协议设置环境变量的替代方案，`ProxyConfig` 类型提供了一个 `proxy_url` 字段，作为 HTTP 和 HTTPS 流量的统一代理。设置 `proxy_url` 时，它会优先于单独的 `http_proxy` 和 `https_proxy` 字段。

```rust
use api::{build_http_client_with, ProxyConfig};

// 来自单一统一 URL（配置文件、CLI flag 等）
let config = ProxyConfig::from_proxy_url("http://proxy.corp.example:3128");
let client = build_http_client_with(&config).expect("proxy client");

// 或者与 NO_PROXY 一起直接设置字段
let config = ProxyConfig {
    proxy_url: Some("http://proxy.corp.example:3128".to_string()),
    no_proxy: Some("localhost,127.0.0.1".to_string()),
    ..ProxyConfig::default()
};
let client = build_http_client_with(&config).expect("proxy client");
```

### 备注

- 当同时设置 `HTTPS_PROXY` 和 `HTTP_PROXY` 时，安全代理会应用于 `https://` URL，普通代理会应用于 `http://` URL。
- `proxy_url` 是一个统一替代方案：一旦设置，就同时作用于 `http://` 和 `https://` 目标，并覆盖按协议设置的字段。
- `NO_PROXY` 接受以逗号分隔的主机后缀列表（例如 `.corp.example`）以及 IP literal。
- 空值会被视为未设置，所以在 shell 里留下 `HTTPS_PROXY=""` 不会启用代理。
- 如果代理 URL 无法解析，`claw` 会退回到直连（无代理）客户端，以保持现有工作流可用；如果你本来期望请求走代理，请检查 URL 是否正确。

## 常用运维命令

```bash
cd rust
./target/debug/claw status
./target/debug/claw sandbox
./target/debug/claw agents
./target/debug/claw mcp
./target/debug/claw skills
./target/debug/claw system-prompt --cwd .. --date 2026-04-04
```

## Session 管理

REPL 轮次会保存在当前工作区的 `.claw/sessions/` 下。

```bash
cd rust
./target/debug/claw --resume latest
./target/debug/claw --resume latest /status /diff
```

常用交互命令包括 `/help`、`/status`、`/cost`、`/config`、`/session`、`/model`、`/permissions` 和 `/export`。

## 配置文件加载顺序

运行时配置按以下顺序加载，后面的配置会覆盖前面的配置：

1. `~/.claw.json`
2. `~/.config/claw/settings.json`
3. `<repo>/.claw.json`
4. `<repo>/.claw/settings.json`
5. `<repo>/.claw/settings.local.json`

## Mock parity harness

工作区包含一个确定性的 Anthropic-compatible mock service 和 parity harness。

```bash
cd rust
./scripts/run_mock_parity_harness.sh
```

手动启动 mock service：

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
