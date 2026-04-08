# 🦞 Claw Code — Rust 实现

Claw Code CLI agent harness 的高性能 Rust 重写版本，面向速度、安全性和原生工具执行而构建。

如果你需要带复制 / 粘贴示例的任务型指南，请查看 [`../USAGE_cn.md`](../USAGE_cn.md)。
如果你需要 parity harness 的细节，请查看 [`MOCK_PARITY_HARNESS_cn.md`](./MOCK_PARITY_HARNESS_cn.md)。

## 快速开始

```bash
# 查看可用命令
cd rust/
cargo run -p rusty-claude-cli -- --help

# 构建工作区
cargo build --workspace

# 运行交互式 REPL
cargo run -p rusty-claude-cli -- --model claude-opus-4-6

# 一次性 prompt
cargo run -p rusty-claude-cli -- prompt "explain this codebase"

# 用于自动化的 JSON 输出
cargo run -p rusty-claude-cli -- --output-format json prompt "summarize src/main.rs"
```

## 配置

设置你的 API 凭据：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# 或使用代理
export ANTHROPIC_BASE_URL="https://your-proxy.com"
```

或者通过 OAuth 认证，让 CLI 在本地持久化凭据：

```bash
cargo run -p rusty-claude-cli -- login
```

## Mock parity harness

当前工作区包含一个确定性的 Anthropic-compatible mock service，以及一个 clean-environment CLI harness，用于端到端 parity 检查。

```bash
cd rust/

# 运行脚本化的 clean-environment harness
./scripts/run_mock_parity_harness.sh

# 或者手动启动 mock service，供临时 CLI 运行使用
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

- `crates/mock-anthropic-service/` —— 可复用的 mock Anthropic-compatible service
- `crates/rusty-claude-cli/tests/mock_parity_harness.rs` —— clean-env CLI harness
- `scripts/run_mock_parity_harness.sh` —— 可复现的封装脚本
- `scripts/run_mock_parity_diff.py` —— 场景检查清单 + PARITY 映射运行器
- `mock_parity_scenarios.json` —— 场景到 PARITY 的清单映射

## 特性

| 特性 | 状态 |
|---------|--------|
| Anthropic / OpenAI-compatible provider 流程 + streaming | ✅ |
| OAuth 登录 / 登出 | ✅ |
| 交互式 REPL（rustyline） | ✅ |
| 工具系统（bash、read、write、edit、grep、glob） | ✅ |
| Web 工具（search、fetch） | ✅ |
| 子 agent / agent 表面 | ✅ |
| Todo 跟踪 | ✅ |
| Notebook 编辑 | ✅ |
| CLAUDE.md / 项目 memory | ✅ |
| 配置文件层级（`.claw.json` + 合并后的 config sections） | ✅ |
| 权限系统 | ✅ |
| MCP server 生命周期 + 检查 | ✅ |
| Session 持久化 + 恢复 | ✅ |
| 成本 / 使用量 / 统计表面 | ✅ |
| Git 集成 | ✅ |
| Markdown 终端渲染（ANSI） | ✅ |
| 模型别名（opus/sonnet/haiku） | ✅ |
| 直接 CLI 子命令（`status`、`sandbox`、`agents`、`mcp`、`skills`、`doctor`） | ✅ |
| Slash 命令（包含 `/skills`、`/agents`、`/mcp`、`/doctor`、`/plugin`、`/subagent`） | ✅ |
| Hooks（`/hooks`、config-backed 生命周期 hooks） | ✅ |
| 插件管理表面 | ✅ |
| Skills inventory / install 表面 | ✅ |
| 核心 CLI 表面上的机器可读 JSON 输出 | ✅ |

## 模型别名

短名称会解析到最新的模型版本：

| 别名 | 解析为 |
|-------|------------|
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251213` |

## CLI 标志与命令

代表性的当前表面：

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
  dump-manifests
  bootstrap-plan
  agents
  mcp
  skills
  system-prompt
  login
  logout
  init
```

命令表面变化很快。要查看 canonical live help，请运行：

```bash
cargo run -p rusty-claude-cli -- --help
```

## Slash 命令（REPL）

Tab completion 会展开 slash 命令、模型别名、权限模式和最近的 session ID。

REPL 现在暴露的表面比原始最小 shell 广泛得多：

- session / visibility：`/help`、`/status`、`/sandbox`、`/cost`、`/resume`、`/session`、`/version`、`/usage`、`/stats`
- workspace / git：`/compact`、`/clear`、`/config`、`/memory`、`/init`、`/diff`、`/commit`、`/pr`、`/issue`、`/export`、`/hooks`、`/files`、`/branch`、`/release-notes`、`/add-dir`
- discovery / debugging：`/mcp`、`/agents`、`/skills`、`/doctor`、`/tasks`、`/context`、`/desktop`、`/ide`
- automation / analysis：`/review`、`/advisor`、`/insights`、`/security-review`、`/subagent`、`/team`、`/telemetry`、`/providers`、`/cron`，以及更多
- plugin management：`/plugin`（别名 `/plugins`、`/marketplace`）

现在可直接使用的 claw-first 表面：

- `/skills [list|install <path>|help]`
- `/agents [list|help]`
- `/mcp [list|show <server>|help]`
- `/doctor`
- `/plugin [list|install <path>|enable <name>|disable <name>|uninstall <id>|update <id>]`
- `/subagent [list|steer <target> <msg>|kill <id>]`

有关使用示例，请查看 [`../USAGE_cn.md`](../USAGE_cn.md)，并运行 `cargo run -p rusty-claude-cli -- --help` 获取实时 canonical 命令列表。

## 工作区布局

```text
rust/
├── Cargo.toml              # 工作区根目录
├── Cargo.lock
└── crates/
    ├── api/                # provider 客户端 + streaming + request preflight
    ├── commands/           # 共享的 slash-command registry + help 渲染
    ├── compat-harness/     # TS manifest 提取 harness
    ├── mock-anthropic-service/ # 确定性的本地 Anthropic-compatible mock
    ├── plugins/            # 插件 metadata、manager、install/enable/disable 表面
    ├── runtime/            # session、config、permissions、MCP、prompts、auth/runtime loop
    ├── rusty-claude-cli/   # 主 CLI 二进制（`claw`）
    ├── telemetry/          # session trace 事件与支持性 telemetry payload
    └── tools/              # 内置工具、skill 解析、tool search、agent runtime 表面
```

### Crate 职责

- **api** —— provider 客户端、SSE streaming、request/response types、auth（API key + OAuth bearer）、request-size/context-window preflight
- **commands** —— slash command 定义、解析、help text 生成、JSON/text 命令渲染
- **compat-harness** —— 从上游 TS 源码提取 tool/prompt manifests
- **mock-anthropic-service** —— 用于 CLI parity tests 和本地 harness 运行的确定性 `/v1/messages` mock
- **plugins** —— 插件 metadata、install/enable/disable/update 流程、plugin tool 定义、hook 集成表面
- **runtime** —— `ConversationRuntime`、config 加载、session 持久化、permission policy、MCP client 生命周期、system prompt 组装、usage 跟踪
- **rusty-claude-cli** —— REPL、一次性 prompt、直接 CLI 子命令、streaming 展示、tool call 渲染、CLI 参数解析
- **telemetry** —— session trace events 与 supporting telemetry payload
- **tools** —— 工具 spec + 执行：Bash、ReadFile、WriteFile、EditFile、GlobSearch、GrepSearch、WebSearch、WebFetch、Agent、TodoWrite、NotebookEdit、Skill、ToolSearch，以及面向 runtime 的 tool discovery

## 统计

- **约 2 万行** Rust
- **9 个 crates** 的工作区
- **二进制名：** `claw`
- **默认模型：** `claude-opus-4-6`
- **默认权限：** `danger-full-access`

## 许可证

参见仓库根目录。
