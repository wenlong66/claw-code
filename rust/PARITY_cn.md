# Parity 状态 — claw-code Rust 迁移

最后更新：2026-04-03

## 摘要

- 规范文档：顶层 [PARITY.md](../PARITY.md) 是 `rust/scripts/run_mock_parity_diff.py` 消费的文件。
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

## 已完成的行为 parity 工作

下面的 hash 来自 `git log --oneline`。合并线的统计来自 `git show --stat <merge>`。

| Lane | 状态 | 功能 commit | 合并 commit | Diff stat |
|------|------|-------------|------------|----------|
| Bash validation（9 个 submodule） | ✅ 完成 | `36dac6c` | —（`jobdori/bash-validation-submodules`） | `1005 insertions` |
| CI fix | ✅ 完成 | `89104eb` | `f1969ce` | `22 insertions, 1 deletion` |
| File-tool edge cases | ✅ 完成 | `284163b` | `a98f2b6` | `195 insertions, 1 deletion` |
| TaskRegistry | ✅ 完成 | `5ea138e` | `21a1e1d` | `336 insertions` |
| Task tool wiring | ✅ 完成 | `e8692e4` | `d994be6` | `79 insertions, 35 deletions` |
| Team + cron runtime | ✅ 完成 | `c486ca6` | `49653fe` | `441 insertions, 37 deletions` |
| MCP lifecycle | ✅ 完成 | `730667f` | `cc0f92e` | `491 insertions, 24 deletions` |
| LSP client | ✅ 完成 | `2d66503` | `d7f0dc6` | `461 insertions, 9 deletions` |
| Permission enforcement | ✅ 完成 | `66283f4` | `336f820` | `357 insertions` |

## 工具表面：40/40（spec parity）

### 真实实现（行为 parity — 深度不同）

| 工具 | Rust 实现 | 行为说明 |
|------|----------|----------|
| **bash** | `runtime::bash`，283 LOC | 子进程执行、超时、后台、sandbox —— **强 parity**。9/9 请求的 validation submodule 已通过 `36dac6c` 标记为完成，main 上也有 sandbox + permission enforcement 支持 |
| **read_file** | `runtime::file_ops` | 支持 offset / limit 读取 —— **良好 parity** |
| **write_file** | `runtime::file_ops` | 文件创建 / 覆写 —— **良好 parity** |
| **edit_file** | `runtime::file_ops` | old/new 字符串替换 —— **良好 parity**。补充：`replace_all` 最近已加入 |
| **glob_search** | `runtime::file_ops` | glob 模式匹配 —— **良好 parity** |
| **grep_search** | `runtime::file_ops` | 类 ripgrep 搜索 —— **良好 parity** |
| **WebFetch** | `tools` | URL 抓取 + 内容抽取 —— **中等 parity**（还需要确认内容截断、重定向处理是否与 upstream 一致） |
| **WebSearch** | `tools` | 搜索查询执行 —— **中等 parity** |
| **TodoWrite** | `tools` | todo / note 持久化 —— **中等 parity** |
| **Skill** | `tools` | skill 发现 / 安装 —— **中等 parity** |
| **Agent** | `tools` | agent 委派 —— **中等 parity** |
| **TaskCreate** | `runtime::task_registry` + `tools` | 通过工具分发接入的内存 task 创建 —— **良好 parity** |
| **TaskGet** | `runtime::task_registry` + `tools` | task 查询 + 元数据 payload —— **良好 parity** |
| **TaskList** | `runtime::task_registry` + `tools` | 基于 registry 的 task 列表 —— **良好 parity** |
| **TaskStop** | `runtime::task_registry` + `tools` | 终态 stop 处理 —— **良好 parity** |
| **TaskUpdate** | `runtime::task_registry` + `tools` | 基于 registry 的消息更新 —— **良好 parity** |
| **TaskOutput** | `runtime::task_registry` + `tools` | 输出捕获读取 —— **良好 parity** |
| **TeamCreate** | `runtime::team_cron_registry` + `tools` | team 生命周期 + task 分配 —— **良好 parity** |
| **TeamDelete** | `runtime::team_cron_registry` + `tools` | team 删除生命周期 —— **良好 parity** |
| **CronCreate** | `runtime::team_cron_registry` + `tools` | cron 条目创建 —— **良好 parity** |
| **CronDelete** | `runtime::team_cron_registry` + `tools` | cron 条目移除 —— **良好 parity** |
| **CronList** | `runtime::team_cron_registry` + `tools` | 基于 registry 的 cron 列表 —— **良好 parity** |
| **LSP** | `runtime::lsp_client` + `tools` | diagnostics、hover、definition、references、completion、symbols、formatting 的 registry + dispatch —— **良好 parity** |
| **ListMcpResources** | `runtime::mcp_tool_bridge` + `tools` | 已连接 server 的资源列表 —— **良好 parity** |
| **ReadMcpResource** | `runtime::mcp_tool_bridge` + `tools` | 已连接 server 的资源读取 —— **良好 parity** |
| **MCP** | `runtime::mcp_tool_bridge` + `tools` | 有状态的 MCP 工具调用桥接 —— **良好 parity** |
| **ToolSearch** | `tools` | 工具发现 —— **良好 parity** |
| **NotebookEdit** | `tools` | Jupyter notebook cell 编辑 —— **中等 parity** |
| **Sleep** | `tools` | 延迟执行 —— **良好 parity** |
| **SendUserMessage/Brief** | `tools` | 面向用户的消息 —— **良好 parity** |
| **Config** | `tools` | 配置检查 —— **中等 parity** |
| **EnterPlanMode** | `tools` | worktree plan mode 切换 —— **良好 parity** |
| **ExitPlanMode** | `tools` | worktree plan mode 恢复 —— **良好 parity** |
| **StructuredOutput** | `tools` | 透传 JSON —— **良好 parity** |
| **REPL** | `tools` | 子进程代码执行 —— **中等 parity** |
| **PowerShell** | `tools` | Windows PowerShell 执行 —— **中等 parity** |

### 仅有 stub（表面 parity，无行为）

| 工具 | 状态 | 备注 |
|------|------|------|
| **AskUserQuestion** | stub | 仍需要真正的用户 I/O 集成 |
| **McpAuth** | stub | 需要超出 MCP lifecycle bridge 的完整 auth UX |
| **RemoteTrigger** | stub | 需要 HTTP client |
| **TestingPermission** | stub | 仅用于测试，优先级较低 |

## Slash Commands：67/141 upstream entries

- 27 个原始 spec（今天之前）—— 都有真实 handler
- 40 个新 spec —— 解析 + stub handler（“尚未实现”）
- 剩余约 74 个 upstream entries 是内部模块 / 对话框 / 步骤，不是用户可见的 `/commands`

### 行为特性检查点（已完成工作 + 仍有缺口）

**Bash 工具 — 9/9 请求的 validation submodule 已完成：**
- [x] `sedValidation` —— 在执行前验证 sed 命令
- [x] `pathValidation` —— 验证命令里的文件路径
- [x] `readOnlyValidation` —— 在只读模式下阻止写入
- [x] `destructiveCommandWarning` —— 对 `rm -rf` 等危险操作发出警告
- [x] `commandSemantics` —— 对命令意图进行分类
- [x] `bashPermissions` —— 按命令类型做权限 gating
- [x] `bashSecurity` —— 安全检查
- [x] `modeValidation` —— 按当前权限模式验证
- [x] `shouldUseSandbox` —— sandbox 决策逻辑

Harness 注记：milestone 2 会验证 bash 成功路径以及 workspace-write 提升的 approve / deny 流程；专门的 validation submodule 已在 `36dac6c` 落地，main 上也具备 sandbox + permission enforcement。

**文件工具 — 已完成检查点：**
- [x] 路径穿越防护（symlink 跟随、`../` 逃逸）
- [x] 读 / 写大小限制
- [x] 二进制文件检测
- [x] 权限模式 enforcement（read-only vs workspace-write）

Harness 注记：`read_file`、`grep_search`、`write_file` 的允许 / 拒绝，以及同一 turn 的多工具组装，现在都由 mock parity harness 覆盖；file edge cases + permission enforcement 已在 `a98f2b6` 和 `336f820` 落地。

**Config / Plugin / MCP 流程：**
- [x] 完整的 MCP server 生命周期（connect、list tools、call tool、disconnect）
- [ ] 插件 install / enable / disable / uninstall 的完整流程
- [ ] 配置 merge 优先级（user > project > local）

Harness 注记：外部插件发现 + 执行现在通过 `plugin_tool_roundtrip` 覆盖；MCP 生命周期已在 `cc0f92e` 落地，而插件生命周期 + 配置 merge 优先级仍然开放。

## 运行时行为缺口

- [x] 贯穿所有工具的权限 enforcement（read-only、workspace-write、danger-full-access）
- [ ] 输出截断（大 stdout / 文件内容）
- [ ] session compaction 行为对齐
- [ ] token 计数 / 成本跟踪准确性
- [x] mock parity harness 已验证流式响应支持

Harness 注记：当前覆盖已经包含 write-file 拒绝、bash 提升 approve / deny，以及 plugin workspace-write 执行路径；permission enforcement 已在 `336f820` 落地。

## 迁移就绪度

- [x] `PARITY.md` 持续维护且内容真实
- [ ] 没有 `#[ignore]` 测试隐藏失败（只允许 1 个：`live_stream_smoke_test`）
- [ ] CI 在每个 commit 上都保持绿色
- [ ] 代码库结构足够干净，适合交接
