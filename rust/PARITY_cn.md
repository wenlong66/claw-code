# Parity 状态 — claw-code Rust 移植

最后更新：2026-04-03

## Mock parity harness — 里程碑 1

- [x] 确定性的 Anthropic 兼容 mock 服务（`rust/crates/mock-anthropic-service`）
- [x] 可复现的干净环境 CLI harness（`rust/crates/rusty-claude-cli/tests/mock_parity_harness.rs`）
- [x] 脚本化场景：`streaming_text`、`read_file_roundtrip`、`grep_chunk_assembly`、`write_file_allowed`、`write_file_denied`

## Mock parity harness — 里程碑 2（行为扩展）

- [x] 脚本化多工具轮次覆盖：`multi_tool_turn_roundtrip`
- [x] 脚本化 bash 覆盖：`bash_stdout_roundtrip`
- [x] 脚本化权限提示覆盖：`bash_permission_prompt_approved`、`bash_permission_prompt_denied`
- [x] 脚本化插件路径覆盖：`plugin_tool_roundtrip`
- [x] 行为 diff/检查清单运行器：`rust/scripts/run_mock_parity_diff.py`

## Harness v2 行为检查清单

规范场景映射：`rust/mock_parity_scenarios.json`

- 多工具 assistant 轮次
- bash 流程 roundtrip
- 跨工具路径的权限强制执行
- 插件工具执行路径
- 文件工具 — harness 验证过的流程

## 已完成的行为 parity 工作

以下哈希来自 `git log --oneline`。合并行数来自 `git show --stat <merge>`。

| 线路 | 状态 | 功能提交 | 合并提交 | Diff 统计 |
|------|--------|----------------|--------------|-----------|
| Bash 验证（9 个子模块） | ✅ 完成 | `36dac6c` | — (`jobdori/bash-validation-submodules`) | `1005 insertions` |
| CI 修复 | ✅ 完成 | `89104eb` | `f1969ce` | `22 insertions, 1 deletion` |
| 文件工具边缘情况 | ✅ 完成 | `284163b` | `a98f2b6` | `195 insertions, 1 deletion` |
| TaskRegistry | ✅ 完成 | `5ea138e` | `21a1e1d` | `336 insertions` |
| Task 工具接线 | ✅ 完成 | `e8692e4` | `d994be6` | `79 insertions, 35 deletions` |
| Team + cron 运行时 | ✅ 完成 | `c486ca6` | `49653fe` | `441 insertions, 37 deletions` |
| MCP 生命周期 | ✅ 完成 | `730667f` | `cc0f92e` | `491 insertions, 24 deletions` |
| LSP 客户端 | ✅ 完成 | `2d66503` | `d7f0dc6` | `461 insertions, 9 deletions` |
| 权限强制执行 | ✅ 完成 | `66283f4` | `336f820` | `357 insertions` |

## 工具表面：40/40（规范 parity）

### 真实实现（行为 parity — 深度不一）

| 工具 | Rust 实现 | 行为说明 |
|------|-----------|-----------|
| **bash** | `runtime::bash` 283 LOC | 子进程执行、超时、后台、sandbox — **parity 很强**。现在通过 `36dac6c` 追踪到 9/9 个请求的验证子模块已全部完成，同时 main 分支上也已有 sandbox + 权限强制运行时支持 |
| **read_file** | `runtime::file_ops` | 偏移/限制读取 — **parity 良好** |
| **write_file** | `runtime::file_ops` | 文件创建/覆盖 — **parity 良好** |
| **edit_file** | `runtime::file_ops` | old/new 字符串替换 — **parity 良好**。缺失项：最近已添加 replace_all |
| **glob_search** | `runtime::file_ops` | glob 模式匹配 — **parity 良好** |
| **grep_search** | `runtime::file_ops` | 类 ripgrep 搜索 — **parity 良好** |
| **WebFetch** | `tools` | URL 抓取 + 内容提取 — **中等 parity**（需要验证内容截断、重定向处理是否与上游一致） |
| **WebSearch** | `tools` | 搜索查询执行 — **中等 parity** |
| **TodoWrite** | `tools` | todo/笔记持久化 — **中等 parity** |
| **Skill** | `tools` | skill 发现/安装 — **中等 parity** |
| **Agent** | `tools` | agent 委派 — **中等 parity** |
| **TaskCreate** | `runtime::task_registry` + `tools` | 内存中的任务创建已接入工具分发 — **parity 良好** |
| **TaskGet** | `runtime::task_registry` + `tools` | 任务查找 + 元数据载荷 — **parity 良好** |
| **TaskList** | `runtime::task_registry` + `tools` | 基于注册表的任务列表 — **parity 良好** |
| **TaskStop** | `runtime::task_registry` + `tools` | 终止状态停止处理 — **parity 良好** |
| **TaskUpdate** | `runtime::task_registry` + `tools` | 基于注册表的消息更新 — **parity 良好** |
| **TaskOutput** | `runtime::task_registry` + `tools` | 输出捕获检索 — **parity 良好** |
| **TeamCreate** | `runtime::team_cron_registry` + `tools` | team 生命周期 + 任务分配 — **parity 良好** |
| **TeamDelete** | `runtime::team_cron_registry` + `tools` | team 删除生命周期 — **parity 良好** |
| **CronCreate** | `runtime::team_cron_registry` + `tools` | cron 条目创建 — **parity 良好** |
| **CronDelete** | `runtime::team_cron_registry` + `tools` | cron 条目移除 — **parity 良好** |
| **CronList** | `runtime::team_cron_registry` + `tools` | 基于注册表的 cron 列表 — **parity 良好** |
| **LSP** | `runtime::lsp_client` + `tools` | 用于 diagnostics、hover、definition、references、completion、symbols、formatting 的注册表 + 分发 — **parity 良好** |
| **ListMcpResources** | `runtime::mcp_tool_bridge` + `tools` | 已连接服务器资源列表 — **parity 良好** |
| **ReadMcpResource** | `runtime::mcp_tool_bridge` + `tools` | 已连接服务器资源读取 — **parity 良好** |
| **MCP** | `runtime::mcp_tool_bridge` + `tools` | 有状态的 MCP 工具调用桥接 — **parity 良好** |
| **ToolSearch** | `tools` | 工具发现 — **parity 良好** |
| **NotebookEdit** | `tools` | jupyter notebook 单元编辑 — **中等 parity** |
| **Sleep** | `tools` | 延迟执行 — **parity 良好** |
| **SendUserMessage/Brief** | `tools` | 面向用户的消息 — **parity 良好** |
| **Config** | `tools` | 配置检查 — **中等 parity** |
| **EnterPlanMode** | `tools` | worktree 计划模式切换 — **parity 良好** |
| **ExitPlanMode** | `tools` | worktree 计划模式恢复 — **parity 良好** |
| **StructuredOutput** | `tools` | 透传 JSON — **parity 良好** |
| **REPL** | `tools` | 子进程代码执行 — **中等 parity** |
| **PowerShell** | `tools` | Windows PowerShell 执行 — **中等 parity** |

### 仅有 Stub（表面 parity，无行为）

| 工具 | 状态 | 说明 |
|------|--------|-------|
| **AskUserQuestion** | stub | 需要实时用户 I/O 集成 |
| **McpAuth** | stub | 需要超出 MCP 生命周期桥接的完整认证 UX |
| **RemoteTrigger** | stub | 需要 HTTP 客户端 |
| **TestingPermission** | stub | 仅测试使用，优先级低 |

## Slash 命令：67/141 个上游条目

- 27 个原始规范（今天之前）——全部都有真实处理器
- 40 个新规范——解析 + stub 处理器（“尚未实现”）
- 剩余约 74 个上游条目是内部模块/对话/步骤，不是用户 `/commands`

### 行为特性检查点（已完成工作 + 剩余缺口）

**Bash 工具 — 9/9 个请求的验证子模块已完成：**
- [x] `sedValidation` — 在执行前验证 sed 命令
- [x] `pathValidation` — 验证命令中的文件路径
- [x] `readOnlyValidation` — 在只读模式下阻止写入
- [x] `destructiveCommandWarning` — 对 rm -rf 等命令发出警告
- [x] `commandSemantics` — 对命令意图分类
- [x] `bashPermissions` — 按命令类型进行权限门控
- [x] `bashSecurity` — 安全检查
- [x] `modeValidation` — 针对当前权限模式进行验证
- [x] `shouldUseSandbox` — sandbox 决策逻辑

Harness 注：里程碑 2 现在验证 bash 成功以及 workspace-write 升级批准/拒绝流程；专门的验证子模块已落地到 `36dac6c`，而 main 分支运行时也已经带有 sandbox + 权限强制执行。

**文件工具 — 已完成检查点：**
- [x] 路径遍历防护（符号链接跟随、`../` 逃逸）
- [x] 读取/写入大小限制
- [x] 二进制文件检测
- [x] 权限模式强制执行（read-only vs workspace-write）

Harness 注：read_file、grep_search、write_file 的允许/拒绝，以及同一轮多工具组装现在都已被 mock parity harness 覆盖；文件边缘情况 + 权限强制执行已落地到 `a98f2b6` 和 `336f820`。

**Config/Plugin/MCP 流程：**
- [x] 完整 MCP 服务器生命周期（连接、列出工具、调用工具、断开）
- [ ] 插件安装/启用/禁用/卸载完整流程
- [ ] 配置合并优先级（user > project > local）

Harness 注：外部插件发现 + 执行现在已通过 `plugin_tool_roundtrip` 覆盖；MCP 生命周期落地于 `cc0f92e`，而插件生命周期 + 配置合并优先级仍然开放。

## 运行时行为缺口

- [x] 跨所有工具的权限强制执行（read-only、workspace-write、danger-full-access）
- [ ] 输出截断（大 stdout/文件内容）
- [ ] 会话压缩行为匹配
- [ ] token 计数 / 成本跟踪准确性
- [x] mock parity harness 已验证流式响应支持

Harness 注：当前覆盖现在包括 write-file 拒绝、bash 升级批准/拒绝，以及 plugin workspace-write 执行路径；权限强制执行已落地到 `336f820`。

## 迁移就绪度

- [x] `PARITY.md` 保持维护且内容真实
- [ ] 没有用 `#[ignore]` 隐藏失败的测试（只允许 1 个：`live_stream_smoke_test`）
- [ ] 每次提交 CI 都是绿色
- [ ] 代码库形态适合交接
