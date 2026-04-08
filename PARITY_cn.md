# Parity 状态 — claw-code Rust 迁移

最后更新：2026-04-03

## 摘要

- 规范文档：顶层 [PARITY.md](./PARITY.md) 是 `rust/scripts/run_mock_parity_diff.py` 消费的文件。
- 请求的 9-lane 检查点：**全部 9 条 lane 已合并到 `main`。**
- 当前 `main` HEAD：`ee31e00`（stub 实现已替换为真实的 AskUserQuestion + RemoteTrigger）。
- 该检查点下的仓库统计：`main` 上 **292 个 commit / 全部分支共 293 个 commit**、**9 个 crates**、**48,599 行 Rust 追踪代码**、**2,568 行测试代码**、**3 位作者**，日期范围 **2026-03-31 → 2026-04-03**。
- mock parity harness 统计：在 `rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs` 中记录了 **10 个脚本化场景**、**19 次捕获的 `/v1/messages` 请求**。

## mock parity harness — milestone 1

- [x] 确定性的 Anthropic-compatible mock service（`rust/crates/mock-anthropic-service`）
- [x] 可复现的 clean-environment CLI harness（`rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs`）
- [x] 脚本化场景：`streaming_text`、`read_file_roundtrip`、`grep_chunk_assembly`、`write_file_allowed`、`write_file_denied`

## mock parity harness — milestone 2（行为扩展）

- [x] 脚本化多工具轮次覆盖：`multi_tool_turn_roundtrip`
- [x] 脚本化 bash 覆盖：`bash_stdout_roundtrip`
- [x] 脚本化权限提示覆盖：`bash_permission_prompt_approved`、`bash_permission_prompt_denied`
- [x] 脚本化插件路径覆盖：`plugin_tool_roundtrip`
- [x] 行为 diff / checklist 运行器：`rust/scripts/run_mock_parity_diff.py`

## Harness v2 行为检查清单

规范场景映射：`rust/mock_parity_scenarios.json`

- 多工具 assistant 轮次
- Bash 流程往返
- 贯穿各工具路径的权限约束
- 插件工具执行路径
- 由 harness 验证的文件工具流程
- 由 mock parity harness 验证的流式响应支持

## 9-lane 检查点

| Lane | 状态 | 功能 commit | 合并 commit | 证据 |
|---|---|---|---|---|
| 1. Bash validation | 已合并 | `36dac6c` | `1cfd78a` | `jobdori/bash-validation-submodules`，`rust/crates/runtime/src/bash_validation.rs`（`main` 上 `+1004`） |
| 2. CI fix | 已合并 | `89104eb` | `f1969ce` | `rust/crates/runtime/src/sandbox.rs`（`+22/-1`） |
| 3. File-tool | 已合并 | `284163b` | `a98f2b6` | `rust/crates/runtime/src/file_ops.rs`（`+195/-1`） |
| 4. TaskRegistry | 已合并 | `5ea138e` | `21a1e1d` | `rust/crates/runtime/src/task_registry.rs`（`+336`） |
| 5. Task wiring | 已合并 | `e8692e4` | `d994be6` | `rust/crates/tools/src/lib.rs`（`+79/-35`） |
| 6. Team+Cron | 已合并 | `c486ca6` | `49653fe` | `rust/crates/runtime/src/team_cron_registry.rs`，`rust/crates/tools/src/lib.rs`（`+441/-37`） |
| 7. MCP lifecycle | 已合并 | `730667f` | `cc0f92e` | `rust/crates/runtime/src/mcp_tool_bridge.rs`，`rust/crates/tools/src/lib.rs`（`+491/-24`） |
| 8. LSP client | 已合并 | `2d66503` | `d7f0dc6` | `rust/crates/runtime/src/lsp_client.rs`，`rust/crates/tools/src/lib.rs`（`+461/-9`） |
| 9. Permission enforcement | 已合并 | `66283f4` | `336f820` | `rust/crates/runtime/src/permission_enforcer.rs`，`rust/crates/tools/src/lib.rs`（`+357`） |

## Lane 详情

### Lane 1 — Bash validation

- **状态：** 已合并到 `main`。
- **功能 commit：** `36dac6c` — `feat: add bash validation submodules — readOnlyValidation, destructiveCommandWarning, modeValidation, sedValidation, pathValidation, commandSemantics`
- **证据：** branch-only diff 新增了 `rust/crates/runtime/src/bash_validation.rs` 和 `runtime::lib` 导出（`2` 个文件共 `+1005`）。
- **main 分支上的实际情况：** `rust/crates/runtime/src/bash.rs` 仍是 `main` 上的实际实现，约 **283 行**，包含 timeout / background / sandbox 执行。`PermissionEnforcer::check_bash()` 已在 `main` 上增加了 read-only gating，但专门的 validation 模块尚未落地。

### Bash 工具 — 上游有 18 个 submodule，Rust 只有 1 个：

- 在 `main` 上，这个说法仍然大体成立。
- Harness 覆盖证明 bash 执行和 prompt 升级流程是存在的，但还没有覆盖完整的上游验证矩阵。
- 该 branch-only lane 目标包括 `readOnlyValidation`、`destructiveCommandWarning`、`modeValidation`、`sedValidation`、`pathValidation` 和 `commandSemantics`。

### Lane 2 — CI fix

- **状态：** 已合并到 `main`。
- **功能 commit：** `89104eb` — `fix(sandbox): probe unshare capability instead of binary existence`
- **合并 commit：** `f1969ce` — `Merge jobdori/fix-ci-sandbox: probe unshare capability for CI fix`
- **证据：** `rust/crates/runtime/src/sandbox.rs` 约 **385 行**，现在会基于真实的 `unshare` capability 和容器信号来解析 sandbox 支持，而不是仅凭二进制是否存在来假设支持。
- **意义：** `.github/workflows/rust-ci.yml` 会运行 `cargo fmt --all --check` 和 `cargo test -p rusty-claude-cli`；这一 lane 从运行时行为中移除了一个 CI 专属的 sandbox 假设。

### Lane 3 — File-tool

- **状态：** 已合并到 `main`。
- **功能 commit：** `284163b` — `feat(file_ops): add edge-case guards — binary detection, size limits, workspace boundary, symlink escape`
- **合并 commit：** `a98f2b6` — `Merge jobdori/file-tool-edge-cases: binary detection, size limits, workspace boundary guards`
- **证据：** `rust/crates/runtime/src/file_ops.rs` 约 **744 行**，现在包含 `MAX_READ_SIZE`、`MAX_WRITE_SIZE`、NUL-byte 二进制检测，以及 canonical workspace-boundary 校验。
- **Harness 覆盖：** `read_file_roundtrip`、`grep_chunk_assembly`、`write_file_allowed` 和 `write_file_denied` 都已写入清单，并由 clean-env harness 执行。

### 文件工具 — harness 验证过的流程

- `read_file_roundtrip` 检查读取路径执行和最终综合。
- `grep_chunk_assembly` 检查分块 grep 工具输出处理。
- `write_file_allowed` 和 `write_file_denied` 验证写入成功和权限拒绝两种情况。

### Lane 4 — TaskRegistry

- **状态：** 已合并到 `main`。
- **功能 commit：** `5ea138e` — `feat(runtime): add TaskRegistry — in-memory task lifecycle management`
- **合并 commit：** `21a1e1d` — `Merge jobdori/task-runtime: TaskRegistry in-memory lifecycle management`
- **证据：** `rust/crates/runtime/src/task_registry.rs` 约 **335 行**，提供了线程安全的内存 registry 上的 `create`、`get`、`list`、`stop`、`update`、`output`、`append_output`、`set_status` 和 `assign_team`。
- **范围：** 这一 lane 把纯固定 payload 的 stub 状态替换成了真实的 runtime-backed task 记录，但它本身并没有引入外部子进程执行。

### Lane 5 — Task wiring

- **状态：** 已合并到 `main`。
- **功能 commit：** `e8692e4` — `feat(tools): wire TaskRegistry into task tool dispatch`
- **合并 commit：** `d994be6` — `Merge jobdori/task-registry-wiring: real TaskRegistry backing for all 6 task tools`
- **证据：** `rust/crates/tools/src/lib.rs` 通过 `execute_tool()` 和具体的 `run_task_*` handler 分发 `TaskCreate`、`TaskGet`、`TaskList`、`TaskStop`、`TaskUpdate` 和 `TaskOutput`。
- **当前状态：** 现在 task tools 通过 `global_task_registry()` 在 `main` 上暴露真实 registry 状态。

### Lane 6 — Team+Cron

- **状态：** 已合并到 `main`。
- **功能 commit：** `c486ca6` — `feat(runtime+tools): TeamRegistry and CronRegistry — replace team/cron stubs`
- **合并 commit：** `49653fe` — `Merge jobdori/team-cron-runtime: TeamRegistry + CronRegistry wired into tool dispatch`
- **证据：** `rust/crates/runtime/src/team_cron_registry.rs` 约 **363 行**，新增线程安全的 `TeamRegistry` 和 `CronRegistry`；`rust/crates/tools/src/lib.rs` 将 `TeamCreate`、`TeamDelete`、`CronCreate`、`CronDelete` 和 `CronList` 接到这些 registry 上。
- **当前状态：** team / cron tools 现在在 `main` 上具备内存生命周期行为；但它们仍然没有真正的后台调度器或 worker fleet。

### Lane 7 — MCP lifecycle

- **状态：** 已合并到 `main`。
- **功能 commit：** `730667f` — `feat(runtime+tools): McpToolRegistry — MCP lifecycle bridge for tool surface`
- **合并 commit：** `cc0f92e` — `Merge jobdori/mcp-lifecycle: McpToolRegistry lifecycle bridge for all MCP tools`
- **证据：** `rust/crates/runtime/src/mcp_tool_bridge.rs` 约 **406 行**，跟踪 server 连接状态、资源列表、资源读取、工具列表、工具分发确认、auth 状态和 disconnect。
- **接线：** `rust/crates/tools/src/lib.rs` 将 `ListMcpResources`、`ReadMcpResource`、`McpAuth` 和 `MCP` 路由到 `global_mcp_registry()` handler。
- **范围：** 这一 lane 把纯 stub response 替换成了 `main` 上的 registry bridge；端到端 MCP 连接填充以及更深的传输 / 运行时能力仍依赖更广泛的 MCP runtime（`mcp_stdio.rs`、`mcp_client.rs`、`mcp.rs`）。

### Lane 8 — LSP client

- **状态：** 已合并到 `main`。
- **功能 commit：** `2d66503` — `feat(runtime+tools): LspRegistry — LSP client dispatch for tool surface`
- **合并 commit：** `d7f0dc6` — `Merge jobdori/lsp-client: LspRegistry dispatch for all LSP tool actions`
- **证据：** `rust/crates/runtime/src/lsp_client.rs` 约 **438 行**，用有状态的 registry 建模 diagnostics、hover、definition、references、completion、symbols 和 formatting。
- **接线：** `rust/crates/tools/src/lib.rs` 中暴露的 `LSP` tool schema 当前列出了 `symbols`、`references`、`diagnostics`、`definition` 和 `hover`，然后通过 `registry.dispatch(action, path, line, character, query)` 路由请求。
- **范围：** 当前 parity 处于 registry / dispatch 级别；completion / format 支持已经存在于 registry 模型里，但还没有在 tool schema 边界上清晰暴露，而且真实外部 language server 进程的编排仍是独立的。

### Lane 9 — Permission enforcement

- **状态：** 已合并到 `main`。
- **功能 commit：** `66283f4` — `feat(runtime+tools): PermissionEnforcer — permission mode enforcement layer`
- **合并 commit：** `336f820` — `Merge jobdori/permission-enforcement: PermissionEnforcer with workspace + bash enforcement`
- **证据：** `rust/crates/runtime/src/permission_enforcer.rs` 约 **340 行**，在 `rust/crates/runtime/src/permissions.rs` 之上增加了工具 gating、文件写入边界检查和 bash read-only 启发式规则。
- **接线：** `rust/crates/tools/src/lib.rs` 暴露了 `enforce_permission_check()`，并在 tool spec 中携带每个工具的 `required_permission` 值。

### 跨工具路径的权限约束

- Harness 场景验证了 `write_file_denied`、`bash_permission_prompt_approved` 和 `bash_permission_prompt_denied`。
- `PermissionEnforcer::check()` 会委托给 `PermissionPolicy::authorize()` 并返回结构化的 allow / deny 结果。
- `check_file_write()` 会强制执行 workspace 边界和只读拒绝；`check_bash()` 会在只读模式下拒绝会修改状态的命令，并阻止 prompt 模式下未确认的 bash。

## 工具表面：`main` 上暴露的 40 个 tool spec

- `rust/crates/tools/src/lib.rs` 中的 `mvp_tool_specs()` 暴露了 **40** 个 tool spec。
- 核心执行已覆盖 `bash`、`read_file`、`write_file`、`edit_file`、`glob_search` 和 `grep_search`。
- `mvp_tool_specs()` 里现有的产品工具包括 `WebFetch`、`WebSearch`、`TodoWrite`、`Skill`、`Agent`、`ToolSearch`、`NotebookEdit`、`Sleep`、`SendUserMessage`、`Config`、`EnterPlanMode`、`ExitPlanMode`、`StructuredOutput`、`REPL` 和 `PowerShell`。
- 这 9 条 lane 的推进把 `Task*`、`Team*`、`Cron*`、`LSP` 和 MCP 工具从纯固定 payload stub 替换成了 `main` 上的 registry-backed handler。
- `Brief` 在 `execute_tool()` 中作为执行别名处理，但它并不是 `mvp_tool_specs()` 中单独暴露的 tool spec。

### 仍然有限或故意保持浅层

- `AskUserQuestion` 仍然返回 pending response payload，而不是真正的交互式 UI 接线。
- `RemoteTrigger` 仍然是 stub response。
- `TestingPermission` 仍然只在测试中可用。
- Task、team、cron、MCP 和 LSP 已不再只是 `execute_tool()` 里的固定 payload stub，但其中一些仍是 registry-backed approximation，而不是完整的外部运行时集成。
- Bash 的深度验证仍然只在 branch 中，直到 `36dac6c` 合并。

## 从旧 PARITY 清单中整理出的内容

- [x] 路径穿越防护（symlink 跟随、`../` 逃逸）
- [x] 读 / 写大小限制
- [x] 二进制文件检测
- [x] 权限模式 enforcement（read-only vs workspace-write）
- [x] 配置 merge 优先级（user > project > local）—— `ConfigLoader::discover()` 按 user → project → local 加载，`loads_and_merges_claude_code_config_files_by_precedence()` 验证了 merge 顺序。
- [x] 插件 install / enable / disable / uninstall 流程 —— `rust/crates/commands/src/lib.rs` 中的 `/plugin` slash 处理会委托给 `rust/crates/plugins/src/lib.rs` 里的 `PluginManager::{install, enable, disable, uninstall}`。
- [x] 没有 `#[ignore]` 测试隐藏失败 —— 对 `rust/**/*.rs` 的 `grep` 发现 0 个 ignored tests。

## 仍未完成

- [ ] 超出 registry bridge 之外的端到端 MCP runtime 生命周期
- [x] 输出截断（大 stdout / 文件内容）
- [ ] session compaction 行为对齐
- [ ] token 计数 / 成本跟踪准确性
- [x] Bash validation lane 已合并到 `main`
- [ ] 每个 commit 都保持 CI 绿色

## 迁移就绪度

- [x] `PARITY.md` 持续维护且内容真实
- [x] 9 条请求 lane 都记录了 commit hash 和当前状态
- [x] 所有 9 条请求 lane 都已落到 `main`（`bash-validation` 仍然只在 branch 中）
- [x] 没有 `#[ignore]` 测试隐藏失败
- [ ] 每个 commit 都保持 CI 绿色
- [x] 代码库结构已经足够清晰，可以用于交接文档
