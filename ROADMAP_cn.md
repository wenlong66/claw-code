# ROADMAP.md 中文正式版

# Clawable 编码执行框架路线图

## 目标

把 claw-code 打造成最 **clawable** 的编码执行框架：
- 启动时不依赖以人为中心的终端假设
- 不依赖脆弱的 prompt 注入时机
- 不依赖不透明的 session 状态
- 不依赖隐藏的插件或 MCP 失败
- 无需人工值守即可处理常规恢复

本路线图假定主要用户是通过 hooks、plugins、sessions 和 channel events 连接起来的 **claws**。

## “clawable”的定义

一个 clawable 的 harness 具有以下特征：
- 启动过程可预测
- 状态与失败模式均可机器读取
- 无需人工持续观察终端也能恢复
- 感知 branch / test / worktree
- 感知插件 / MCP 生命周期
- 以事件为先，而不是以日志为先
- 能够自主执行下一步动作

## 当前痛点

### 1. session 启动流程较脆弱
- trust prompt 可能阻塞 TUI 启动
- prompt 可能被误发送至 shell，而不是 coding agent 里
- “session 已存在” 不等于 “session 已就绪”

### 2. 事实分散在多个层面
- tmux 状态
- clawhip 事件流
- git / worktree 状态
- test 状态
- gateway / plugin / MCP 运行时状态

### 3. 事件过于接近日志形态
- claws 目前需要从嘈杂文本中推断过多信息
- 重要状态没有被规范化成机器可读事件

### 4. 恢复循环对人工操作依赖过高
- 重启 worker
- 接受 trust prompt
- 重新注入 prompt
- 检测陈旧分支
- 重试失败的启动
- 手工区分基础设施失败与代码失败

### 5. 分支新鲜度约束不足
- side branch 可能错过已经落到 main 的修复
- 大范围测试失败可能只是陈旧分支噪声，而不是真正的回归

### 6. 插件 / MCP 失败分类不够明确
- 启动失败、握手失败、配置错误、部分启动、降级模式都没有被清晰暴露

### 7. 面向人类的 UX 仍然渗入 claw 工作流
- 过多流程依赖终端 / TUI 行为，而非显式的 agent 状态转换和控制 API

## 产品原则

1. **以状态机为先** —— 每个 worker 都有明确的生命周期状态。
2. **事件优先于抓取式文本** —— channel 输出应来自类型化事件。
3. **先恢复，再升级处理** —— 对已知失败模式先自动修复一次，再寻求人工帮助。
4. **先检查分支新鲜度，再追责** —— 在把红色测试当成新回归之前先检测旧分支问题。
5. **部分成功是第一等公民** —— 例如 MCP 启动可以有部分服务器成功、部分失败，并给出结构化降级报告。
6. **终端只是传输层，不是真相** —— tmux / TUI 可以继续作为实现细节，但编排状态必须存在于其上层。
7. **策略必须可执行** —— 合并、重试、rebase、陈旧清理和升级处理规则都应由机器强制执行。

## 路线图

## 阶段 1 — 可靠的 Worker 启动

### 1. 为 coding workers 增加 ready 握手生命周期
添加显式状态：
- `spawning`
- `trust_required`
- `ready_for_prompt`
- `prompt_accepted`
- `running`
- `blocked`
- `finished`
- `failed`

验收标准：
- 在 `ready_for_prompt` 之前绝不会发送 prompt
- trust prompt 状态可检测且会被发出
- shell 误投递会变成第一等失败状态，能够被检测到

### 2. trust prompt 解析器
为已知仓库 / worktree 增加 allowlist 白名单自动信任行为。

验收标准：
- 受信任的仓库会自动清除 trust prompt
- 会发出 `trust_required` 和 `trust_resolved` 事件
- 不在 allowlist 内的仓库仍然会被门控

### 3. 结构化 session 控制 API
在 tmux 之上提供机器控制能力：
- 创建 worker
- 等待就绪
- 发送任务
- 获取状态
- 获取最后一个错误
- 重启 worker
- 终止 worker

验收标准：
- claw 可以在不把 raw send-keys 作为主要控制面的前提下操作 coding worker

## 阶段 2 — 事件原生的 Clawhip 集成

### 4. 规范化的 lane 事件 schema
定义如下类型化事件：
- `lane.started`
- `lane.ready`
- `lane.prompt_misdelivery`
- `lane.blocked`
- `lane.red`
- `lane.green`
- `lane.commit.created`
- `lane.pr.opened`
- `lane.merge.ready`
- `lane.finished`
- `lane.failed`
- `branch.stale_against_main`

验收标准：
- clawhip 消费类型化 lane 事件
- Discord 摘要基于结构化事件生成，而不是只靠 pane scraping

### 5. 失败分类法
规范化失败类别：
- `prompt_delivery`
- `trust_gate`
- `branch_divergence`
- `compile`
- `test`
- `plugin_startup`
- `mcp_startup`
- `mcp_handshake`
- `gateway_routing`
- `tool_runtime`
- `infra`

验收标准：
- 阻塞项可被机器分类
- 仪表盘和重试策略可以按失败类型分支

### 6. 可操作摘要压缩
把噪声很大的事件流压缩成：
- 当前阶段
- 最近一次成功检查点
- 当前阻塞点
- 建议的后续恢复动作

验收标准：
- channel 状态更新保持简短且基于机器事实
- claws 不再需要从原始构建输出中推断状态

## 阶段 3 — 分支 / 测试感知与自动恢复

### 7. 在大范围验证前检测陈旧分支
在进行大范围测试之前，将当前分支与 `main` 比较，并检测是否遗漏了已知修复。

验收标准：
- 发出 `branch.stale_against_main`
- 根据策略建议或自动执行 rebase / merge-forward
- 避免把陈旧分支失败误判为新回归

### 8. 常见失败的恢复配方
为以下情况编码自动恢复：
- trust prompt 未解决
- prompt 被发到了 shell
- 分支过旧
- 跨 crate 重构后编译变红
- MCP 启动握手失败
- 插件部分启动

验收标准：
- 在升级处理之前会先自动尝试一次恢复
- 这次恢复尝试本身也会作为结构化事件数据发出

### 9. Green 级别契约
worker 应区分：
- targeted tests green
- package green
- workspace green
- merge-ready green

验收标准：
- 不再出现含糊的 “tests passed” 表述
- merge 策略可以针对 lane 类型要求正确的 green 级别

## 阶段 4 — 以 Claw 为中心的任务执行

### 10. 类型化 task packet 格式
定义一个结构化 task packet，字段例如：
- objective
- scope
- repo / worktree
- branch policy
- acceptance tests
- commit policy
- reporting contract
- escalation policy

验收标准：
- claws 可以在不依赖冗长自然语言 prompt blob 的情况下分派工作
- task packet 可以被安全地记录、重试和转换

### 11. 面向自主编码的策略引擎
编码如下自动化规则：
- 如果 green + scoped diff + review 通过 -> merge 到 dev
- 如果分支陈旧 -> 先 merge-forward，再跑大范围测试
- 如果启动被阻塞 -> 先恢复一次，然后升级处理
- 如果 lane 完成 -> 发出 closeout 并清理 session

验收标准：
- 规则从聊天指令迁移到可执行策略中

### 12. Claw 原生仪表盘 / lane 看板
提供一个机器可读的看板，包含：
- repos
- active claws
- worktrees
- 分支新鲜度
- 红 / 绿状态
- 当前阻塞点
- merge readiness
- 最近一次有意义的事件

验收标准：
- claws 可以直接查询状态
- 面向人的视图变成渲染层，而不是事实来源

## 阶段 5 — 插件与 MCP 生命周期成熟化

### 13. 一等公民的插件 / MCP 生命周期契约
每个插件 / MCP 集成都应暴露：
- 配置校验契约
- 启动健康检查
- 发现结果
- 降级模式行为
- shutdown / cleanup 契约

验收标准：
- 部分启动和按服务器失败都会以结构化方式报告
- 即使其中一个服务器失败，成功的服务器仍然可用

### 14. MCP 端到端生命周期对齐
补齐以下环节的差距：
- config load
- server registration
- spawn / connect
- initialize handshake
- tool / resource discovery
- invocation path
- error surfacing
- shutdown / cleanup

验收标准：
- parity harness 和 runtime tests 都覆盖健康与降级启动场景
- 故障服务器将以结构化失败形式暴露，而非不透明的 warning

## 由当前真实痛点导出的即时待办

优先级顺序：P0 = 阻塞 CI / green 状态，P1 = 阻塞集成接线，P2 = clawability 加固，P3 = swarm 效率提升。

**P0 — 先修（CI 可靠性）**
1. 将 `render_diff_report` 测试隔离到 tmpdir —— **已完成**：`render_diff_report_for()` 测试现在在临时 git 仓库中运行，而不是在真实工作树里运行；定向执行 `cargo test -p rusty-claude-cli render_diff_report -- --nocapture` 在分支 / worktree 活动期间也能保持绿色
2. 将 GitHub CI 从单 crate 覆盖扩展到 workspace 级验证 —— **已完成**：`.github/workflows/rust-ci.yml` 现在在 workspace 级别运行 `cargo test --workspace`，并执行 fmt / clippy
3. 增加发布级二进制工作流 —— **已完成**：`.github/workflows/release.yml` 现在会为 CLI 构建带 tag 的 Rust release artifacts
4. 增加以容器为先的测试 / 运行文档 —— **已完成**：`Containerfile` + `docs/container.md` 文档化了 build、bind-mount 和 `cargo test --workspace` 的 canonical Docker / Podman 工作流
5. 在 onboarding 文档和 help 中暴露 `doctor` / preflight 诊断 —— **已完成**：README + USAGE 现在把 `claw doctor` / `/doctor` 放在首次运行路径中，并指向内置的 preflight 报告
6. 在 CI 中自动检查 branding / source-of-truth 残留 —— **已完成**：`.github/scripts/check_doc_source_of_truth.py` 和 `doc-source-of-truth` CI job 现在会阻止已跟踪文档和 metadata 中陈旧的 repo / org / invite 残留
7. 消除首次运行 help / build 路径中的 warning 噪音 —— **已完成**：当前 `cargo run -q -p rusty-claude-cli -- --help` 会直接渲染干净的 help 输出，不再先出现一屏 warning
8. 将 `doctor` 从仅 slash 命令提升为顶层 CLI 入口 —— **已完成**：`claw doctor` 现在是本地 shell 入口，并补了直接 help 和 health-report 输出的回归测试
9. 让机器可读的 status 命令真的机器可读 —— **已完成**：`claw --output-format json status` 和 `claw --output-format json sandbox` 现在输出结构化 JSON 快照，而不是散文式表格
10. 统一用户可见输出中的旧 config / skill 命名空间 —— **已完成**：skills / help 的 JSON / 文本输出现在把 `.claw` 作为 canonical namespace，并将旧根路径折叠成 `.claw` 风格的 source ids / labels
11. 让 `skills` 和 `mcp` 这类 inventory 命令尊重 JSON 输出 —— **已完成**：直接 CLI inventory 命令现在支持 `--output-format json`，skills 和 MCP inventory 都返回结构化 payload
12. 审计整个 CLI surface 上的 `--output-format` 契约 —— **已完成**：直接 CLI 命令现在在 help / version / status / sandbox / agents / mcp / skills / bootstrap-plan / system-prompt / init / doctor 上都遵守确定性的 JSON / 文本处理，并在 `output_format_contract.rs` 和恢复后的 `/status` JSON 覆盖中有回归测试

**P1 — 下一步（集成接线，解除验证阻塞）**
2. 增加跨模块集成测试 —— **已完成**：12 个集成测试覆盖 worker → recovery → policy、stale_branch → policy、green_contract → policy、reconciliation 流程
3. 接上线完成事件 emitter —— **已完成**：`lane_completion` 模块中的 `detect_lane_completion()` 会根据 session finished + tests green + push complete 自动把 `LaneContext::completed` 置为真，随后进入 policy closeout
4. 将 `SummaryCompressor` 接入 lane event pipeline —— **已完成**：`compress_summary_text()` 会进入 `tools/src/lib.rs` 中 `LaneEvent::Finished` 的 detail 字段

**P2 — clawability 加固（原始 backlog）**
5. Worker 就绪握手 + trust 解析 —— **已完成**：`WorkerStatus` 状态机具备 `Spawning` → `TrustRequired` → `ReadyForPrompt` → `PromptAccepted` → `Running` 生命周期，`trust_auto_resolve` + `trust_gate_cleared` 负责门控
6. prompt 误投递检测与恢复 —— **已完成**：`prompt_delivery_attempts` 计数器、`PromptMisdelivery` 事件检测、`auto_recover_prompt_misdelivery` + `replay_prompt` 恢复分支已经落地
7. clawhip 中的规范化 lane 事件 schema —— **已完成**：`LaneEvent` enum 提供 `Started/Blocked/Failed/Finished` 变体，`LaneEvent::new()` 作为类型化构造函数，已集成到 `tools/src/lib.rs`
8. 失败分类法 + blocker 规范化 —— **已完成**：`WorkerFailureKind` enum（`TrustGate/PromptDelivery/Protocol/Provider`）以及 `FailureScenario::from_worker_failure_kind()` 已接通恢复配方
9. 在 workspace tests 之前检测陈旧分支 —— **已完成**：`stale_branch.rs` 模块提供新鲜度检测、behind/ahead 指标和策略集成
10. MCP 结构化降级启动报告 —— **已完成**：`McpManager` 的降级启动报告（`mcp_stdio.rs` 中新增 +183 行）、失败服务器分类（startup / handshake / config / partial），以及工具输出中的结构化 `failed_servers` + `recovery_recommendations`
11. 结构化 task packet 格式 —— **已完成**：`task_packet.rs` 模块提供 `TaskPacket` 结构体、校验、序列化、`TaskScope` 解析（workspace / module / single-file / custom），并集成到 `tools/src/lib.rs`
12. lane board / 机器可读状态 API —— **已完成**：lane completion 加固 + `LaneContext::completed` 自动检测 + MCP 降级报告一起提供了机器可读状态
13. **session 完成失败分类** —— **已完成**：`WorkerFailureKind::Provider` + `observe_completion()` + 恢复配方桥接已经落地
14. **config merge 校验缺口** —— **已完成**：`config.rs` 在 deep-merge 前增加 hook 校验（+56 行），格式错误的条目会带着 source-path 上下文失败，而不是在 merge 后才报 parse error
15. **MCP manager discovery 的 flaky 测试** —— **已完成**：`manager_discovery_report_keeps_healthy_servers_when_one_server_fails` 在多次稳定通过后重新作为正常 workspace test 运行，因此降级启动覆盖不再被 `#[ignore]` 隐藏

16. **commit provenance / worktree-aware push events** —— **已完成**：`LaneCommitProvenance` 现在在 lane events 中携带 branch / worktree / canonical-commit / supersession 元数据，并且在写入 agent manifests 之前会先执行 `dedupe_superseded_commit_events()`，把被 supersede 的 commit 事件折叠为最新的 canonical lineage
17. **孤立模块集成审计** —— **已完成**：`runtime` 现在把 `session_control` 和 `trust_resolver` 保持在 `#[cfg(test)]` 下，直到它们被接入真实的非测试执行路径；这样正常构建就不会再宣称存在实际上不存在的 clawability surface area
18. **context-window 预检缺口** —— **已完成**：provider request sizing 现在会在超大请求离开进程前发出 `context_window_blocked`，并使用 model-context registry，而不是旧的 naive max-token heuristic
19. **子命令 help 直接掉进 runtime / API 路径** —— **已完成**：`claw doctor --help`、`claw status --help`、`claw sandbox --help` 以及嵌套的 `mcp` / `skills` help 现在都会在本地被拦截，不再触发 runtime / provider 启动；相关回归测试覆盖了直接 CLI 路径
20. **session 状态分类缺口（working / blocked / finished / truly stale）** —— **已完成**：agent manifests 现在会推导出诸如 `working`、`blocked_background_job`、`blocked_merge_conflict`、`degraded_mcp`、`interrupted_transport`、`finished_pending_report` 和 `finished_cleanable` 这样的机器状态；terminal-state persistence 还会记录 commit provenance 和派生状态，让下游监控可以区分“安静但还在推进”与“真正空闲”的 session
21. **恢复后的 `/status` JSON 对齐缺口** —— dogfooding 显示，新的 `claw status --output-format json` 现在会输出结构化 JSON，但恢复后的 slash-command status 在至少一条 dispatch 路径里仍然会走文本形态。`rust/crates/rusty-claude-cli/tests/resume_slash_commands.rs::resumed_status_command_emits_structured_json_when_requested` 的本地 CI 等效复现会失败，报 `expected value at line 1 column 1`，说明 automation 明确要求 JSON 时仍可能拿到文本。**行动：** 统一 fresh 与 resumed 的 `/status` 渲染路径，让它们共用一套 output-format 契约，并补回归覆盖，确保 resumed JSON 输出始终有效。
22. **session / runtime 崩溃的失败面过于不透明** —— 持续的 dogfood 失败现在会被包进像 `Something went wrong while processing your request. Please try again, or use /new to start a fresh session.` 这样的通用包装里，但不会暴露到底是 provider auth、session corruption、slash-command dispatch、render failure 还是 transport / runtime panic。这会阻碍快速自我恢复，并把可操作的 clawability bug 变成盲目重试。**行动：** 保留一个简短的面向用户的失败类别（`provider_auth`、`session_load`、`command_dispatch`、`render`、`runtime_panic` 等），附带本地 trace / session id，并确保操作者能够从聊天可见错误直接跳到对应的失败日志。
23. **`doctor --output-format json` 的 check 级结构缺口** —— **已完成**：`claw doctor --output-format json` 现在在保留 human-readable `message` / `report` 的同时，也会输出结构化的逐项检查诊断（`name`、`status`、`summary`、`details`，以及像 workspace paths 和 sandbox fallback data 这样的类型化字段），并在 `output_format_contract.rs` 中有回归覆盖
24. **插件生命周期 init / shutdown 测试在 workspace 并行执行下出现 flaky** —— dogfooding 发现 `build_runtime_runs_plugin_lifecycle_init_and_shutdown` 在单独运行时通过，但在 `cargo test --workspace` 下会失败，因为兄弟测试会在 tempdir 里的 shell init script 路径上竞争。这是测试脆弱性，而不是代码路径回归，但它会破坏 CI 信心并浪费排障时间。**行动：** 更稳妥地隔离每个测试的临时资源（独立目录 + 不共享 cwd 假设），审查清理时机，并加一条回归保护，确保插件生命周期测试在 workspace 并行执行下依然稳定。
26. **恢复后的本地命令 JSON 对齐缺口** —— **已完成**：直接 `claw --output-format json` 早已为 `sandbox`、`mcp`、`skills`、`version` 和 `init` 提供结构化 renderer，但恢复后的 `claw --output-format json --resume <session> /…` 路径仍然会因为 resumed slash dispatch 只给 `/status` 输出 JSON 而回落到散文文本。现在恢复后的 `/sandbox`、`/mcp`、`/skills`、`/version` 和 `/init` 都会复用与直接 CLI 对应路径相同的 JSON envelope，并在 `rust/crates/rusty-claude-cli/tests/resume_slash_commands.rs` 与 `rust/crates/rusty-claude-cli/tests/output_format_contract.rs` 中有回归覆盖

**P3 — swarm 效率**
13. branch-lock 协议 —— **已完成**：`branch_lock::detect_branch_lock_collisions()` 现在能在并行 lanes 还没 drift 成重复实现之前，检测同分支 / 同 scope 以及嵌套模块之间的冲突
14. commit provenance / worktree-aware push events —— **已完成**：lane event provenance 现在包含 branch / worktree / superseded / canonical lineage 元数据，并且在下游消费者渲染之前，manifest 持久化会对被 superseded 的 commit events 去重

## 建议的 Session 划分

### Session A — worker 启动协议
关注点：
- trust prompt 检测
- ready-for-prompt 握手
- prompt 误投递检测

### Session B — clawhip lane events
关注点：
- 规范化 lane event schema
- 失败分类法
- 摘要压缩

### Session C — 分支 / 测试智能
关注点：
- 陈旧分支检测
- green 级别契约
- 恢复配方

### Session D — MCP 生命周期加固
关注点：
- 启动 / 握手可靠性
- 结构化 failed server 报告
- 降级模式运行时行为
- 生命周期测试 / harness 覆盖

### Session E — 类型化 task packet + 策略引擎
关注点：
- 结构化任务格式
- 重试 / 合并 / 升级处理规则
- 自主 lane 关闭行为

## MVP 成功标准

当以下条件满足时，我们就应认为 claw-code 在实质上更 clawable：
- 一个 claw 可以启动 worker，并且能确定无疑地知道它何时就绪
- claws 不再会不小心把任务误发送至 shell
- 陈旧分支失败会在浪费调试时间之前被识别出来
- clawhip 报告的是机器状态，而不只是 tmux 散文
- MCP / plugin 启动失败能够被清晰分类并暴露出来
- 单个 coding lane 可以在无需人工值守的情况下，自行从常见的启动和分支问题中恢复

## 简述

claw-code 应该从：
- 一个人类也能直接驱动的 CLI

演进为：
- 一个 **claw-native 执行运行时**
- 一个 **event-native 编排底座**
- 一个 **plugin / hook-first 的自主编码 harness**

## 部署架构缺口（来自 2026-04-08 的内部试用反馈）

### WorkerState 在 runtime 里；/state 不在 opencode serve 里

**在第 8 批内部试用中发现的根因。**

`worker_boot.rs` 里有一套较为完整的 `WorkerStatus` 状态机（`Spawning → TrustRequired → ReadyForPrompt → Running → Finished/Failed`）。它通过 `runtime/src/lib.rs` 作为公共 API 导出。但 claw-code 是加载在 `opencode` 二进制内部的 **plugin**，它不能给 `opencode serve` 添加 HTTP route。HTTP server 完全由上游 opencode 进程（v1.3.15）所有。

**影响：** 不能通过 `curl localhost:4710/state` 拿到 JSON 版 `WorkerStatus`。要实现这类 endpoint，只能：
1. 将 `/state` route 上游到 opencode 的 HTTP server（需要给 `sst/opencode` 提 PR），或
2. 写一个 sidecar HTTP 进程，去进程内查询 `WorkerRegistry`（可行但脆弱），或
3. 把 `WorkerStatus` 写到一个约定好的文件路径（`.claw/worker-state.json`），让外部观察者轮询。

**推荐路径：** 采用方案 3 —— 在每次状态变化时把 `WorkerStatus` transition 输出到 `.claw/worker-state.json`。这完全在 claw-code 的 plugin 范围内，不需要上游改动，并且能给 clawhip 一个可轮询的文件，用来区分真正卡死的 worker 和只是安静但仍在推进的 worker。

**行动项：** 在每次状态转换时，把 `WorkerRegistry::transition()` 改成原子写入 `.claw/worker-state.json`。再加一个 `claw state` CLI 子命令读取并打印这个文件。补回归测试。

**前次会话备注：** 之前某次会话摘要声称 `0984cca` 已经通过 axum 落地了一个 `/state` HTTP endpoint。这个说法是错的—— main 分支上没有这个 commit，axum 也不是依赖项，而且 HTTP server 不是我们自己的。实际存在的工作只有：`worker_boot.rs` 里的 `WorkerStatus` enum + `WorkerRegistry`，并且已经作为公共导出完整接入 `runtime/src/lib.rs`。
