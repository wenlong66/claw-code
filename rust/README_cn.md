# 🦞 Claw Code — Rust 实现

Claw Code CLI 代理执行环境的高性能 Rust 重写版本。为速度、安全性和原生工具执行而构建。

有关可复制粘贴示例的任务导向指南，请参见 [`../USAGE.md`](../USAGE.md)。

## 快速开始

```bash
# 查看可用命令
cd rust/
cargo run -p rusty-claude-cli -- --help

# 构建工作区
cargo build --workspace

# 运行交互式 REPL
cargo run -p rusty-claude-cli -- --model claude-opus-4-6

# 一次性提示
cargo run -p rusty-claude-cli -- prompt "explain this codebase"

# 用于自动化的 JSON 输出
cargo run -p rusty-claude-cli -- --output-format json prompt "summarize src/main.rs"
```

## 配置

设置你的 API 凭证：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# 或使用代理
export ANTHROPIC_BASE_URL="https://your-proxy.com"
```

或者直接提供 OAuth bearer token：

```bash
export ANTHROPIC_AUTH_TOKEN="anthropic-oauth-or-proxy-bearer-token"
```

## Mock parity harness

该工作区现在包含一个确定性的 Anthropic 兼容 mock 服务，以及一个用于端到端 parity 检查的干净环境 CLI harness。

```bash
cd rust/

# 运行脚本化的干净环境 harness
./scripts/run_mock_parity_harness.sh

# 或手动启动 mock 服务以便临时 CLI 运行
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

Harness 覆盖：

- `streaming_text`
- `read_file_roundtrip`
- `grep_chunk_assembly`
- `write_file_allowed`
- `write_file_denied`
- `multi_tool_turn_roundtrip`
- `bash_stdout_roundtrip`
- `bash_permission_prompt_approved`
- `bash_permission_prompt_denied`
- `plugin_tool_roundtrip`

主要产物：

- `crates/mock-anthropic-service/` — 可复用的 mock Anthropic 兼容服务
- `crates/rusty-claude-cli/tests/mock_parity_harness.rs` — 干净环境 CLI harness
- `scripts/run_mock_parity_harness.sh` — 可复现的包装脚本
- `scripts/run_mock_parity_diff.py` — 场景检查清单 + PARITY 映射运行器
- `mock_parity_scenarios.json` — 场景到 PARITY 的清单

## 功能

| 功能 | 状态 |
|---------|--------|
| Anthropic / OpenAI 兼容的提供方流程 + 流式输出 | ✅ |
| 通过 `ANTHROPIC_AUTH_TOKEN` 进行直接 bearer-token 认证 | ✅ |
| 交互式 REPL（rustyline） | ✅ |
| 工具系统（bash、read、write、edit、grep、glob） | ✅ |
| Web 工具（search、fetch） | ✅ |
| 子代理 / agent 表面 | ✅ |
| Todo 跟踪 | ✅ |
| Notebook 编辑 | ✅ |
| CLAUDE.md / 项目记忆 | ✅ |
| 配置文件层级（`.claw.json` + 合并配置分区） | ✅ |
| 权限系统 | ✅ |
| MCP 服务器生命周期 + 检查 | ✅ |
| 会话持久化 + 恢复 | ✅ |
| 成本 / 使用量 / 统计表面 | ✅ |
| Git 集成 | ✅ |
| Markdown 终端渲染（ANSI） | ✅ |
| 模型别名（opus/sonnet/haiku） | ✅ |
| 直接 CLI 子命令（`status`、`sandbox`、`agents`、`mcp`、`skills`、`doctor`） | ✅ |
| Slash 命令（包括 `/skills`、`/agents`、`/mcp`、`/doctor`、`/plugin`、`/subagent`） | ✅ |
| Hooks（`/hooks`，基于配置的生命周期 hooks） | ✅ |
| 插件管理表面 | ✅ |
| Skills 清单 / 安装表面 | ✅ |
| 覆盖核心 CLI 表面的机器可读 JSON 输出 | ✅ |

## 模型别名

短名称会解析为最新模型版本：

| 别名 | 解析为 |
|-------|------------|
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251213` |

## CLI 标志和命令

当前代表性表面：

```text
claw [OPTIONS] [COMMAND]

Flags:
  --model MODEL
  --output-format text|json
  --permission-mode MODE
  --dangerously-skip-permissions
  --allowedTools TOOLS
  --resume [SESSION.jsonl|session-id|latest]
  --version, -V

Top-level commands:
  prompt <text>
  help
  version
  status
  sandbox
  acp [serve]
  dump-manifests
  bootstrap-plan
  agents
  mcp
  skills
  system-prompt
  init
```

`claw acp` 是面向编辑器优先用户的本地可发现性入口：它会报告当前 ACP/Zed 状态，而不会启动运行时。截至 2026 年 4 月 16 日，claw-code 仍**不**提供 ACP/Zed 守护进程入口，`claw acp serve` 目前只是一个状态别名，直到真正的协议表面落地为止。

命令表面变化很快。要查看权威的实时帮助文本，请运行：

```bash
cargo run -p rusty-claude-cli -- --help
```

## Slash 命令（REPL）

Tab 补全会展开 slash 命令、模型别名、权限模式和最近的会话 ID。

REPL 现在暴露的表面比最初的最小 shell 广泛得多：

- 会话 / 可见性：`/help`、`/status`、`/sandbox`、`/cost`、`/resume`、`/session`、`/version`、`/usage`、`/stats`
- 工作区 / git：`/compact`、`/clear`、`/config`、`/memory`、`/init`、`/diff`、`/commit`、`/pr`、`/issue`、`/export`、`/hooks`、`/files`、`/release-notes`
- 发现 / 调试：`/mcp`、`/agents`、`/skills`、`/doctor`、`/tasks`、`/context`、`/desktop`
- 自动化 / 分析：`/review`、`/advisor`、`/insights`、`/security-review`、`/subagent`、`/team`、`/telemetry`、`/providers`、`/cron` 等
- 插件管理：`/plugin`（别名 `/plugins`、`/marketplace`）

现在可直接通过 slash 形式使用的 claw-first 表面：
- `/skills [list|install <path>|help]`
- `/agents [list|help]`
- `/mcp [list|show <server>|help]`
- `/doctor`
- `/plugin [list|install <path>|enable <name>|disable <name>|uninstall <id>|update <id>]`
- `/subagent [list|steer <target> <msg>|kill <id>]`

有关使用示例，请参见 [`../USAGE.md`](../USAGE.md)，并运行 `cargo run -p rusty-claude-cli -- --help` 查看实时权威命令列表。

## 工作区布局

```text
rust/
├── Cargo.toml              # 工作区根目录
├── Cargo.lock
└── crates/
    ├── api/                # 提供方客户端 + 流式输出 + 请求预检
    ├── commands/           # 共享 slash 命令注册表 + 帮助渲染
    ├── compat-harness/     # TS 清单提取 harness
    ├── mock-anthropic-service/ # 确定性的本地 Anthropic 兼容 mock
    ├── plugins/            # 插件元数据、管理器、安装/启用/禁用表面
    ├── runtime/            # 会话、配置、权限、MCP、提示词、认证/运行时循环
    ├── rusty-claude-cli/   # 主 CLI 二进制（`claw`）
    ├── telemetry/          # 会话追踪与使用量遥测类型
    └── tools/              # 内置工具、skills 解析、工具搜索、agent 运行时表面
```

### Crate 职责

- **api** — 提供方客户端、SSE 流、请求/响应类型、认证（`ANTHROPIC_API_KEY` + bearer-token 支持）、请求大小/上下文窗口预检
- **commands** — slash 命令定义、解析、帮助文本生成、JSON/text 命令渲染
- **compat-harness** — 从上游 TS 源码中提取工具/提示词清单
- **mock-anthropic-service** — 用于 CLI parity 测试和本地 harness 运行的确定性 `/v1/messages` mock
- **plugins** — 插件元数据、安装/启用/禁用/更新流程、插件工具定义、hook 集成表面
- **runtime** — `ConversationRuntime`、配置加载、会话持久化、权限策略、MCP 客户端生命周期、系统提示词组装、使用量跟踪
- **rusty-claude-cli** — REPL、一次性提示、直接 CLI 子命令、流式显示、工具调用渲染、CLI 参数解析
- **telemetry** — 会话轨迹事件与支持性的遥测载荷
- **tools** — 工具规范 + 执行：Bash、ReadFile、WriteFile、EditFile、GlobSearch、GrepSearch、WebSearch、WebFetch、Agent、TodoWrite、NotebookEdit、Skill、ToolSearch，以及面向运行时的工具发现

## 统计

- **约 20K 行** Rust
- 工作区中有 **9 个 crate**
- 二进制名称：`claw`
- 默认模型：`claude-opus-4-6`
- 默认权限：`danger-full-access`

## 许可证

见仓库根目录。
