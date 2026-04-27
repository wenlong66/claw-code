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
- prompt 可能落到 shell 而不是编码代理
- “session 已存在” 并不等于 “session 已就绪”

### 2. 真实状态分散在多层之间
- tmux 状态
- clawhip 事件流
- git / worktree 状态
- 测试状态
- gateway / plugin / MCP 运行时状态

### 3. 事件过于像日志
- claws 目前从噪声文本中推断了太多内容
- 重要状态没有被规范化成机器可读事件

### 4. 恢复循环过于手工
- 重启 worker
- 接受 trust prompt
- 重新注入 prompt
- 检测 stale branch
- 重试失败的启动
- 手动区分基础设施问题和代码问题

### 5. 对 branch 新鲜度的约束不够
- side branches 可能漏掉已经落到 main 的修复
- 大范围测试失败可能只是 stale branch 噪音，而不是真正的回归

### 6. 插件 / MCP 失败分类不足
- 启动失败、握手失败、配置错误、部分启动和降级模式没有被清晰暴露

### 7. 人类 UX 仍然渗入 claw 工作流
- 过多地依赖终端 / TUI 行为，而不是显式的代理状态转换和控制 API

## 产品原则

1. **状态机优先** — 每个 worker 都有明确的生命周期状态。
2. **事件优于抓取散文** — channel 输出应由类型化事件生成。
3. **先恢复，再升级** — 已知失败模式应先自动修复一次，再请求人类帮助。
4. **先看 branch 新鲜度，再归因** — 在把测试红灯当成新回归之前先检测 stale branch。
5. **部分成功是一等公民** — 例如 MCP 启动可以对部分服务器成功、对部分服务器失败，并给出结构化的降级模式报告。
6. **终端只是传输，不是真相** — tmux / TUI 可以保留为实现细节，但编排状态必须存在于它们之上。
7. **策略可执行** — 合并、重试、rebase、stale cleanup 和升级规则都应该由机器强制执行。

## 路线图

## Phase 1 — 可靠的 worker 启动

### 1. coding worker 的 ready-handshake 生命周期
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
- prompt 绝不会在 `ready_for_prompt` 之前发送
- trust prompt 状态可检测且会被发出
- shell 误投递成为可检测的一等失败状态

### 1.5. 首个 prompt 接受 SLA
在 `ready_for_prompt` 之后，暴露第一项任务是否在有界时间窗内真正被接受，而不是让 claws 进入静默悬空状态。

为以下信号发出类型化事件：
- `prompt.sent`
- `prompt.accepted`
- `prompt.acceptance_delayed`
- `prompt.acceptance_timeout`

至少跟踪：
- 从 `ready_for_prompt` 到第一次发送 prompt 的时间
- 从第一次发送 prompt 到 `prompt_accepted` 的时间
- 是否需要重试或恢复才能接受

验收标准：
- clawhip 可以区分 “worker 已就绪但空闲” 与 “prompt 已发送但实际上没有被接受”
- ready 状态和第一项任务执行之间的长时间静默会变成机器可见
- 在人类开始 scrape pane 之前，恢复逻辑就可以基于 acceptance timeout 触发

### 2. Trust prompt resolver
为已知仓库 / worktree 添加 allowlist 化的自动信任行为。

验收标准：
- 受信任的仓库可以自动清除 trust prompt
- `trust_required` 和 `trust_resolved` 会发出事件
- 未加入 allowlist 的仓库仍然会被门控

### 3. 结构化 session 控制 API
在 tmux 之上提供机器控制：
- 创建 worker
- 等待 ready
- 发送任务
- 获取状态
- 获取最后一个错误
- 重启 worker
- 终止 worker

验收标准：
- claw 可以在不把原始 send-keys 作为主要控制平面的情况下操作 coding worker

### 3.5. 启动预检 / doctor 合约
在创建 worker 或发送 prompt 之前，先运行一个机器可读的预检，报告该 lane 是否真的适合启动。

预检应检查并发出类型化结果，覆盖：
- repo / worktree 是否存在以及预期 branch 是否正确
- branch 相对 base branch 的新鲜度
- trust gate 的可能性 / allowlist 状态
- 所需二进制和控制 socket
- plugin 发现 / allowlist / 启动资格
- MCP 配置存在性和服务可达性预期
- 最近一次失败的启动原因（如果有）

验收标准：
- claws 可以在启动一个注定失败的 worker 之前快速失败
- 被阻止的启动会返回一个简短的结构化诊断，而不是让用户靠 pane scrape 进行 triage
- clawhip 可以总结“为什么这个 lane 根本没启动”，而不必从终端噪声中猜测

## Phase 2 — 事件原生的 clawhip 集成

### 4. 规范化 lane event schema
定义类型化事件，例如：
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
- clawhip 消费类型化 lane events
- Discord summary 由结构化事件渲染，而不只是靠 pane scraping

### 4.5. session event 排序 + 终端状态对齐
当同一个 session 在短时间内发出相互矛盾的生命周期事件（`idle`、`error`、`completed`、transport/server-down）时，claw-code 必须给出确定性的最终真相，而不是让下游 claws 去猜。

要求行为：
- 为 session 生命周期事件附加单调递增序列 / 因果排序元数据
- 分类哪些事件是终端事件，哪些只是建议性事件
- 将重复或乱序的终端事件对齐成一个规范化的 lane 结果
- 区分 “session 终端状态未知，因为 transport 挂了” 与真正的 `completed`

验收标准：
- clawhip 可以在 `completed -> idle -> error -> completed` 这类噪声里保持稳定，不会重复上报，也不会信错最终状态
- transport 在一串 session 事件之后挂掉时，会暴露为一种类型化的不确定状态，而不是悄悄改写历史
- 下游自动化对每个 lane / session 只会看到一个规范化终端结果

### 4.6. 事件来源 / 环境标记
每个发出的事件都应该说明它来自 live lane、synthetic test、healthcheck、replay 还是 system transport layer，这样 claws 不会把测试噪声当成生产真实状态。

所需字段：
- event source kind（`live_lane`、`test`、`healthcheck`、`replay`、`transport`）
- environment / channel label
- emitter identity
- 下游自动化使用的 confidence / trust level

验收标准：
- clawhip 可以忽略或降低测试 ping 的优先级，而不需要靠 heuristic 文本匹配
- synthetic / system events 不会污染 lane 状态，也不会触发错误的后续自动化
- 即使测试流量和生产流量共用同一个 channel，事件流也仍然保持机器可信

### 4.7. 创建时的 session identity 完整性
新建 session 不应该在 orchestrator 立即需要的字段上显示为 `(untitled)` 或 `(unknown)`。

要求行为：
- 在创建时发出稳定的标题、workspace / worktree 路径和 lane / session purpose
- 如果某个字段暂时未知，就发出明确的类型化占位原因，而不是裸的 unknown 字符串
- 后续补充的元数据要回填到同一个 session identity 上，而不能制造歧义

验收标准：
- clawhip 可以在不等待后续 chatter 的情况下路由 / 处理一个新 session
- `(untitled)` / `(unknown)` 的创建事件不再迫使人类或 bot 去猜作用域
- session 创建事件会立刻对监控和 owner 决策可操作

### 4.8. 重复终端事件抑制
当同一个 session 重复发出 `completed`、`failed` 或其他终端通知时，claw-code 应该在它们触发重复下游反应之前先把重复项折叠掉。

要求行为：
- 为每个 lane / session 终态附加一个规范化的 terminal-event fingerprint
- 在一个对齐窗口内抑制或合并重复的终端通知
- 保留原始事件历史以便审计，但对下游只暴露一个可操作的终端结果
- 当后来的重复 payload 与原始终端 payload 有实质差异时，要暴露出来

验收标准：
- clawhip 不会因为重复的终端通知而双重上报或双重关闭
- 重复的 `completed` burst 只会变成一个可执行的 finish event，而不是重复噪声
- 即使上游 emitter 很啰嗦，下游自动化依然保持幂等

### 4.9. lane 归属 / 作用域绑定
每个 session 和 lane event 都应该声明它的 owner 以及所属 workflow scope，这样无关的外部 / system 工作就不会污染 claw-code 的后续循环。

要求行为：
- 在已知时附加 owner / assignee identity
- 附加 workflow scope（例如 `claw-code-dogfood`、`external-git-maintenance`、`infra-health`、`manual-operator`）
- 标记当前 watcher 预期应该 act、只观察，还是忽略
- 在 session restart、resume 和晚到的终端事件中保留 scope

验收标准：
- clawhip 可以直接说“out-of-scope external session”，而不需要人类额外写一句 prose 免责声明
- 无关的 session churn 不会触发错误的 claw-code follow-up 或 blocker 报告
- 监控视图可以过滤到“对这个 claw 可操作”，而不是把主机上的所有 session 混在一起

### 4.10. 提示确认 / 去重合约
周期性的 clawhip nudges 应该携带足够状态，让 claws 知道当前 prompt 是新工作、重试，还是已经确认过的 heartbeat。

要求行为：
- 附加 nudge id / cycle id 和 delivery timestamp
- 暴露当前 claw 是否已经对该 cycle 确认或响应过
- 区分 `new nudge`、`retry nudge` 和 `stale duplicate`
- 允许下游 summary 将一个 pinpoint 反向绑定到触发它的 nudge id

验收标准：
- claws 不会因为同一个周期性的 nudge 再次出现，就不断制造新的后续事项
- clawhip 可以判断沉默意味着“还没处理”还是“本轮已经确认过”
- 重复的 dogfood prompt 会在重试中保持幂等并可审计

### 4.11. 新建 pinpoints 的稳定 roadmap-id 分配
当 claw 记录一个新的 pinpoint / follow-up 时，roadmap surface 应该立刻分配或暴露一个稳定 tracking id，而不是让该条目先以匿名 prose 的形式存在。

要求行为：
- 在 filing 时分配 canonical roadmap id
- 在结构化事件 / report payload 中暴露该 id
- 在后续编辑、重排和 summary 压缩中保留同一个 id
- 区分 `new roadmap filing` 与 `update to existing roadmap item`

验收标准：
- channel update 可以在同一轮里用稳定 id 引用刚刚 filed 的 pinpoint
- 下游 claws 不需要靠 heuristic 文本匹配来判断 follow-up 是新的还是已经跟踪过的
- 即使文档被反复编辑，roadmap 驱动的 dogfood 循环仍然保持可审计

### 4.12. roadmap item 生命周期状态合约
每个 roadmap pinpoint 都应该携带一个机器可读的生命周期状态，这样 claws 不会不断重新发现或重复上报已经处于 active、resolved 或 superseded 的条目。

要求行为：
- 暴露生命周期状态（`filed`、`acknowledged`、`in_progress`、`blocked`、`done`、`superseded`）
- 附加最后一次状态变化的时间戳
- 允许新报告声明自己是首次 filing、状态更新还是关闭
- 当一个 pinpoint supersedes 或合并到另一个时，保留 lineage

验收标准：
- clawhip 可以在不解释 prose 的情况下区分“新 gap”和“已有 gap 仍然活跃”
- 已完成或已 superseded 的条目不会再次出现，像是新发现一样
- roadmap 驱动的后续循环会变成有状态的，而不是反复无状态

### 4.13. 多消息 report 的原子性
一次 dogfood / lane update 应该能够表示为一个结构化 report payload，即使聊天表面最终把它渲染成多条消息。

要求行为：
- 为整个 update 分配一个 report id
- 将 `active_sessions`、`exact_pinpoint`、`concrete_delta` 和 `blocker` 字段绑定到同一个 report id
- 暴露消息片段的顺序，供 chat transport 拆分 report 时使用
- 允许下游消费者重建一条规范化 update，而不需要靠 heuristic 去抓取相邻消息

验收标准：
- clawhip 和其他 claws 可以解析出一条逻辑 update，即便 Discord delivery 把它拆成多条 post
- 部分或乱序的 message burst 不会把 `pinpoint`、`delta` 和 `blocker` 搞乱
- dogfood report 会变成机器可靠的 summary，而不是脆弱的聊天考古

### 4.14. 跨 claw pinpoint 去重 / 合并合约
当多个 claws 因为同一个底层失败而记录了近似相同的 pinpoints 时，roadmap surface 应该把它们合并或建立关联，而不是让重复 follow-up 变成多个独立发现。

要求行为：
- 为新 filing 计算或暴露 similarity / dedupe key
- 允许新的 filing 以 `same_root_cause`、`related` 或 `supersedes` 的关系链接到已有 roadmap item
- 保留 reporter-specific evidence，同时折叠 canonical tracked issue
- 当后来的 filing 实际上不同，尽管措辞相似时，要能暴露出来

验收标准：
- 两个 claws 报告同一个 gap 时，不会自动创建两个独立 roadmap item
- roadmap 增长反映的是新的真实发现，而不是重复观察的 churn
- 下游监控可以同时看到 canonical item 和支持它的重复证据，而不会失去可审计性

### 4.15. pinpoint 证据附着合约
每个 filed pinpoint 都应该携带结构化支撑证据，这样后续实现者不必再反向重建为什么会认为这个 gap 存在。

要求行为：
- 附加证据引用，例如 session id、message id、commit、log、stack trace 或文件路径
- 为每个附件标注证据角色（`repro`、`symptom`、`root_cause_hint`、`verification`）
- 保留供人类快速浏览的精简预览，同时为机器保留规范引用
- 允许在 filing 之后继续添加证据，而不改变 pinpoint identity

验收标准：
- 在聊天回溯或 session context 丢失后，roadmap item 依然可执行
- 实现 lane 可以直接从结构化证据开始，而不是重新发现原始失败
- 优先级可以根据证据质量而不是纯 prose 自信程度来加权

### 4.16. pinpoint 优先级 / 严重性合约
每个 filed pinpoint 都应该暴露一个机器可读的紧急度 / 严重性信号，这样 claws 就能把立即的执行阻塞和较低优先级的 clawability 加固区分开来。

要求行为：
- 附加 priority / severity 字段（例如 `p0` / `p1` / `p2` 或 `critical` / `high` / `medium` / `low`）
- 区分面向用户的故障、仅面向 operator 的摩擦、可观测性债务和长尾加固
- 允许在新证据到达时改变 priority，而不改变 pinpoint identity
- 给出分配该 priority 的原因（blast radius、可复现性、自动化破坏、合并风险）

验收标准：
- clawhip 可以在不依赖 prose 紧急语气的情况下对新的 pinpoints 排序
- 实现队列可以优先拿真正的 blocker，而不是只做报告层面的细枝末节
- roadmap dogfood 可以先聚焦最有破坏力的 clawability gap

### 4.17. pinpoint 到 implementation 的交接合约
一个 filed pinpoint 应该能够直接变成执行 lane，而不需要人类手工重新翻译相同上下文。

要求行为：
- 暴露一个结构化 handoff packet，包含目标、疑似作用域、证据引用、优先级和建议验证
- 标记该 pinpoint 是 `implementation_ready`、`needs_repro` 还是 `needs_triage`
- 保留 roadmap item 与任何 spawned execution lane / worktree / PR 之间的链接
- 允许后续执行结果更新原始 pinpoint 状态，而不是分叉出彼此不相连的叙事

验收标准：
- claw 可以接手一个 filed pinpoint，并以最少的重新解释开始实现
- roadmap item 不再只是死 prose，而变成可执行的交接单元
- 后续循环可以看到哪些 pinpoints 已经转成了真正的执行 lane

### 4.18. 报告背压 / 重复摘要折叠
周期性的 dogfood reporting 应该避免在每个周期都把完整的已知 gap inventory 重新广播一遍，尤其是当只有很小的增量发生变化时。

要求行为：
- 区分 `new since last report` 与 `still active but unchanged`
- 发出更紧凑的 delta-first summaries，并可选地展开完整状态
- 为每个 channel / reporting cursor 跟踪已重复的内容，以便自动折叠不变项
- 在别处保留一份 canonical full snapshot 以便审计 / 调试，而不是把 live channel 挤爆

验收标准：
- 新信号不会被每周期重复的 backlog list 淹没
- claws 和人类可以直接扫描最新 update 中真正变化的内容，而不是每次重读全部 inventory
- 重复的 dogfood 循环会保持低噪声，同时不丢失可审计性

### 4.19. 无变化 / no-op 确认合约
当一个 dogfood cycle 没有产生新的 pinpoint、没有新的 delta、也没有新的 blocker 时，claws 也应该能明确地确认这一轮，而不是假装出现了新发现。

要求行为：
- 为 reporting cycle 暴露结构化的 `no_change` / `noop` 结果
- 将该结果绑定到触发它的 nudge / report id
- 区分 `checked and unchanged` 与 `not yet checked`
- 保留最后一个有意义的 pinpoint / delta 引用，而不是把它重新 filing 成新工作

验收标准：
- 重复的 nudges 不会因为真实答案是“没有变化”就被迫制造假新意
- clawhip 可以区分 `handled, no delta` 与沉默 / 漏处理
- 在系统稳定时，dogfood 循环会变得诚实且低噪声

### 4.20. observation freshness / staleness-age 合约
每个被报告的状态、pinpoint 或 blocker 都应携带明确的 observation timestamp / age，这样下游 claws 就能分辨新鲜状态和旧的延续状态。

要求行为：
- 为 active-session state、pinpoints 和 blockers 附加 observed-at timestamp 和派生 age
- 区分新近观测到的事实与上一轮带过来的状态
- 允许 freshness TTL，使旧观测自动从 `current` 降级为 `stale`
- 当一条 report 在不同字段上混合了不同 freshness 窗口时，要能暴露出来

验收标准：
- claws 不会因为某个 observation 在最新 report 里再次出现，就把两小时前的内容误认为当前真相
- 旧的延续状态会变得可见，并且可以被降权或重新验证
- 即使某些字段在多轮中一直不变，dogfood summary 仍然可信

### 4.21. 事实 / 假设 / 置信度标记
Dogfood report 应区分已确认的观测和推测的根因猜测，这样下游 claws 不会把猜想当成既定事实。

要求行为：
- 为每条报告声明打上 `observed_fact`、`inference`、`hypothesis` 或 `recommendation` 标签
- 为非事实类声明附上置信度分数或置信度区间
- 保留支持每条声明的证据
- 允许后续 report 在不改变 underlying pinpoint identity 的前提下，把一个 hypothesis 提升为 confirmed fact

验收标准：
- claws 可以分辨“我们观察到了 X”与“我们认为 Y 导致了它”
- 推测性的 root-cause 文本不会被误认为机器可信状态
- dogfood summary 在保持可操作性的同时，也会诚实说明不确定性

### 4.22. 负证据 / 搜索后未找到合约
当一个 dogfood cycle 报告某件事没找到（没有活跃 session、没有新 delta、没有 repro、没有 blocker）时，report 也应该说明它检查了什么，这样“没有”才是机器有意义的，而不是空洞 prose。

要求行为：
- 为负向结论附加被检查的 surface / source（sessions、logs、roadmap、state file、channel window 等）
- 区分 `not observed in checked scope` 与 `unknown / not checked`
- 在相关时保留负向观测所用的 query / window
- 当搜索范围不完整时，允许后续 report 否定之前的负向结论

验收标准：
- `no blocker` 和 `no new delta` 会变成可审计的结论，而不是无法验证的感受
- 下游 claws 可以分辨缺失意味着“查过了且干净”还是“根本没查”
- 稳定的 dogfood period 仍然可信，不会把不确定性夸大成确定性

### 4.23. 字段级 delta 归因
即使采用 delta-first 报告，claws 也仍然需要知道每个结构化字段到底发生了什么变化，而不是靠 prose 去猜。

要求行为：
- 为核心 report 字段（`active_sessions`、`pinpoint`、`delta`、`blocker`、生命周期状态、priority、freshness）发出字段级变化标记
- 区分 `changed`、`unchanged`、`cleared` 和 `carried_forward`
- 在有用时保留前一个值的引用或哈希，便于机器比较
- 允许一份 report 同时包含 changed 与 unchanged 字段，而不会丢失字段级状态

验收标准：
- 下游 claws 不需要 diff 整条消息正文，也能精确知道这轮变了什么
- delta-first summary 保持紧凑，但仍然可供机器比较
- 重复 report 不再迫使你重新解析文本，只为了回答“到底变了什么？”

### 4.24. report schema 版本化 / 兼容性合约
随着结构化 dogfood report 不断演进，report surface 需要明确的 schema 版本控制，这样下游 claws 才能安全地解析新字段，而不会悄悄破坏。

要求行为：
- 为每个结构化 report payload 附加 schema version
- 定义 additive 与 breaking field 变更
- 为只理解旧 schema 的消费者提供兼容指导
- 保留一个最小稳定 core，确保在部分升级时基本解析仍然可用

验收标准：
- 下游 claws 可以对未知 schema version 选择拒绝、警告或优雅降级，而不是静默误解析
- 新增 reporting 字段不会随机破坏现有自动化
- dogfood reporting 可以快速演进，同时仍然保持机器可信

### 4.25. 面向消费者能力协商的结构化报告
只有 schema versioning 还不够；如果不同 claws 消费 report surface 的子集不同，producer 还应该知道 consumer 真的能理解什么。

要求行为：
- 允许下游消费者声明自己支持的 schema version 和可选字段族 / capability
- 当 consumer 不能处理更丰富的 report 字段时，允许 producer 发出一个 reduced-compatible payload
- 暴露一份 report 是为了兼容而降级，还是以完整保真度发出的
- 即使交付的是降级视图，也要保留一份 canonical full-fidelity representation 以供审计 / 调试

验收标准：
- 使用旧解析器的 claws 仍然可以消费有用的 report，而不会把静默字段丢失误认为字段不存在
- 更丰富的 report 演进不会强迫所有 consumer 同步升级
- 在混合版本 claw fleet 中，reporting 仍然保持机器可信

### 4.26. 自描述的 report schema surface
即使有了版本和能力协商，下游 claws 仍需要一个机器可读的方式来发现某个 report version 具体包含哪些字段和语义。

要求行为：
- 为结构化 report payload 暴露机器可读的 schema / field registry
- 以可消费的格式记录字段含义、枚举、可选性和弃用状态
- 允许消费者获取某个 report version / capability set 对应的 schema
- 为字段保留稳定 identifier，让文档、代码和实时 payload 指向同一个 schema 真相

验收标准：
- 新消费者可以直接集成，而不必从聊天日志里的样例 payload 逆向工程
- schema 漂移可以相对于声明的 source of truth 被检测出来
- 结构化 report 的演进可以保持快速，而不会把每个集成都变成脆弱的考古

### 4.27. 面向受众的 report 投影
同一份 canonical dogfood report 应该能够投影成不同 consumer 视图（clawhip、Jobdori、人类 operator），而不需要每个 consumer 都从头重新总结完整 payload。

要求行为：
- 保留一份 canonical structured report payload
- 支持 consumer-specific projections / views（例如 `delta_brief`、`ops_audit`、`human_readable`、`roadmap_sync`）
- 允许 consumer 声明自己偏好的 projection 形状和详细程度
- 让 projection lineage 显式存在，这样一个精简视图仍然能追溯回 canonical report

验收标准：
- Jobdori / Clawhip / 人类不会反复用略有不同的 prose 重新广播同一份完整 inventory
- 每个 consumer 都能拿到合适的细节量，而不需要自己再造一个有损 summary 层
- 报告噪声下降，而底层事实保持共享且可审计

### 4.28. canonical report identity / content-hash 锚点
一旦有了多个 projection 和 summary，系统就需要一个稳定的 identity 锚点，证明它们都来自同一个底层 report state。

要求行为：
- 为完整的结构化 payload 分配 canonical report id 和 content hash / fingerprint
- 在不改变未变更底层内容的 canonical identity 的前提下，包含 projection-specific metadata
- 暴露两份 projection 是因为源 report 变了，还是只是渲染变了
- 允许下游消费者检测对同一个 report payload 的意外重复发送

验收标准：
- claws 可以验证不同的 audience view 指向的是同一个底层 report truth
- 相同内容的重复 projection 不会看起来像新的状态变化
- 即使同一份 canonical payload 被以多种方式渲染，report lineage 仍然可审计

### 4.29. projection 失效 / stale-view cache 合约
如果 canonical report 发生变化，先前发出的受众特定 projection 必须能识别为 stale，这样下游 claws 才不会继续拿旧的渲染视图做事。

要求行为：
- 将每个 projection 绑定到它所派生自的 canonical report id + content hash / version
- 当底层 canonical payload 变化时，把 projection 标记为 superseded
- 暴露消费者看到的是最新兼容 projection 还是 stale cached 视图
- 允许便宜地重新生成 projection，而不制造假的新 report identity

验收标准：
- claws 不会在 canonical report 更新后，把旧的 `delta_brief` 视图误当成当前真相
- projection caching 会减少噪声和计算量，而不会增加 stale-action 风险
- 面向受众的视图会与底层 report 的新鲜度安全绑定

### 4.30. projection 时的脱敏 / 敏感度标记
随着 canonical report 积累更多证据，projection 需要有明确策略决定哪些内容可以展示给哪类受众，同时仍然保持机器可信。

要求行为：
- 用敏感度分类标记 report 字段 / 证据（例如 `public`、`internal`、`operator_only`、`secret`）
- 让 projection 按受众策略对敏感字段进行脱敏、摘要或哈希，同时保持 canonical report 不变
- 暴露某个 projection 是否因为敏感性原因省略或转换了数据
- 保留足够稳定的 identity / provenance，使脱敏后的 projection 仍能与 canonical report 关联

验收标准：
- 更丰富的 canonical report 不会迫使所有 audience view 暴露同样细节
- consumer 可以分辨“字段没了是因为被脱敏”还是“字段本来就不存在”
- 面向受众的 projection 在安全的同时，不会变成无法验证的黑箱

### 4.31. 脱敏 provenance / 策略可追溯性
当 projection 脱敏或转换数据时，下游消费者应该能看出是哪个策略 / 规则导致的，而不是把脱敏当成无解释的消失。

要求行为：
- 为被转换或被省略的字段附加 redaction reason / policy id
- 区分基于策略的脱敏、大小截断、兼容性降级和源头缺失
- 保留从 projection 回到 canonical field classification 的可审计链接
- 允许 operator 查看是哪个 projection policy version 生成了可见输出

验收标准：
- claws 能看出某个字段为什么被隐藏，而不仅仅是它消失了
- 脱敏后的 projection 依然便于运维调试，而不是模糊不清
- 随着 reporting / projection 策略演进，敏感度控制仍然可审计

### 4.32. 确定性的 projection / 脱敏可复现性
在相同的 canonical report、schema version、consumer capability set 和 projection policy 下，生成的 projection 应该能逐字节复现（或至少规范等价），这样审计和 diff 就不会在重渲染时漂移。

要求行为：
- 让相同输入下的 projection / redaction 输出保持确定性
- 暴露哪些输入参与了 projection identity（schema version、capability set、policy version、canonical content hash）
- 区分内容变化与非确定性渲染噪声
- 即使 transport formatting 不同，也允许做 canonical equivalence 检查

验收标准：
- 用同样的 audience 重新渲染同一份 report，不会制造假 delta
- 审计 / 调试工作流可以复现为什么之前的 projection 会长那样
- projection pipeline 在反复重生成时仍然保持机器可信

### 4.33. projection 的 golden-fixture / 回归锁定
一旦结构化 projection 变成确定性的，claw-code 仍然需要回归 fixture 来锁定预期输出，这样 report rendering 的变化就不会悄悄滑过去。

要求行为：
- 维护覆盖核心 report 形态、redaction 类别和 capability 降级的 canonical fixture inputs
- 为支持的 audience views 做 snapshot 或等价性测试
- 让有意的 rendering / schema 变化通过显式更新 fixture 来体现，而不是静默漂移
- 暴露是哪个 fixture set / version 验证了某次 projection pipeline 变更

验收标准：
- projection 回归会在 downstream claws 发现 broken 或漂移输出之前就被捕获
- 确定性渲染的主张会被持续验证，而不是靠假设
- report / projection 的演进可以保持快速，同时不牺牲机器可信的稳定性

### 4.34. 下游 consumer 的符合性测试合约
仅有 producer 侧的 fixture 覆盖还不够，如果真实的下游 claws 仍然错误解析或解释 reporting 合约，生态系统还是会出问题。需要一种方式来验证 consumer 行为是否符合声明的 report schema / projection 规则。

要求行为：
- 为不同 schema version、capability 降级、redaction 状态和 no-op cycle 定义 consumer 符合性用例
- 提供一个可机器运行的 consumer 测试包或 fixture bundle
- 区分 parse 成功和语义正确（例如：正确处理 `redacted` 与 `missing`、`stale` 与 `current`）
- 暴露哪个 consumer / version 最近一次通过了符合性套件

验收标准：
- report-contract 漂移会在 producer / consumer 边界被捕获，而不只是 producer 内部
- 下游 claws 可以证明它们确实理解自己声称支持的结构化 reporting surface
- 混合 claw fleet 可以保持互操作，而不是靠乐观或人工抽查

### 4.35. 暂定状态去重 / in-flight 确认抑制
当 claw 发出 `working on it`、`please wait` 或 `adding a roadmap gap` 之类的临时状态时，重复的暂定通知不应把 channel 淹没，除非内容有实质变化。

要求行为：
- 为 provisional / in-flight 状态更新单独计算 fingerprint，与终态或带 delta 的 report 分开
- 在一个短的对齐窗口内抑制意义未变的重复 provisional 消息
- 只有当进度状态、owner、blocker 或 ETA 有实质变化时，才允许新的 provisional update 放行
- 原始重复仍可保留用于审计 / 调试，但不会作为新的 channel event 暴露

验收标准：
- 监控流不会因为重复的 `please wait` / `working on it` 消息而抖动
- consumer 可以分辨“仍在进行中，且没有变化”与“新的可操作更新”
- in-flight 确认在不淹没真正状态转换的前提下仍然有用

### 4.36. 暂定状态升级超时
如果一个 provisional / in-flight 状态长时间没有变化，系统就不该再把它当成无害噪声，而应该把它重新提升为可操作的 stale signal。

要求行为：
- 为 provisional 状态附加 timeout / TTL 策略
- 将长期未变化的 provisional 状态升级为类型化的 stale / blocker 信号
- 区分“因为仍然新鲜所以去重”与“去重太久了，现在可疑了”
- 暴露触发升级的 timeout 策略是哪一个

验收标准：
- `working on it` 不会在真正进度停滞时永久压住可见性
- consumer 可以信任 provisional dedupe，而不会丢掉长期卡住的工作
- 低噪声监控会在合适的时候重新浮现长期 in-flight 状态

### 4.37. 策略阻止动作的交接
当请求的动作被 branch / merge / release policy 禁止时（例如直接推送到 `main`），系统应该暴露结构化拒绝和下一条安全执行路径，而不是只留下散文描述。

要求行为：
- 用类型化原因对 policy-blocked 请求分类（`main_push_forbidden`、`release_requires_owner` 等）
- 在可用时附加 governing policy source 和 actor scope
- 发出安全 fallback 路径（`create branch`、`open PR`、`request owner approval` 等）
- 允许下游 claws / operators 区分“被策略阻止”与“被技术故障阻止”

验收标准：
- policy 拒绝会变成机器可操作的内容，而不是死路一条的聊天文本
- claws 可以直接切换到安全替代工作流，而不用重新 triage 同一个请求
- 监控 / reporting 可以把治理阻断与真实产品 / runtime 缺陷区分开来

### 4.38. policy exception / owner-approval token 合约
对于通常会被 policy 阻止、但可在明确 owner 批准下放行的动作，批准路径应该是机器可读的，而不是依赖含糊 prose 的解释。

要求行为：
- 将 policy exception 表示为类型化的 approval grant 或 token，并限定作用于 action / repo / branch / 时间窗
- 将批准绑定到批准者身份以及被覆盖的 policy
- 区分 `no approval`、`approval pending`、`approval granted` 和 `approval expired/revoked`
- 允许下游 claws 在执行原本会被阻止的动作前先验证批准凭证

验收标准：
- 特例批准不再依赖模糊的聊天解释
- claws 可以安全地执行 policy-exception 流程，而不会把它们和普通 blocked 请求混淆
- 即使出现 owner 授权的例外，治理依然保持可审计

### 4.39. approval-token replay / 一次性使用 enforcement
如果 policy-exception 的批准变成了机器可读 token，它也需要 replay protection，这样一次显式例外就不会在无声中被重复利用到超出预期的范围。

要求行为：
- 在适用时支持一次性或有限次数使用的 approval grant
- 记录 token 的消耗与其授权的精确 action / repo / branch / commit 作用域绑定
- 以类型化 policy error 拒绝 replay、作用域扩张或过期后复用
- 暴露批准是未使用、已消耗、部分消耗、已过期还是已撤销

验收标准：
- 一个 owner 批准的例外不会悄悄授权重复或更危险的更广泛动作
- claws 可以区分“有一个有效批准”与“批准已经用完”
- 治理例外在自动化下仍然可审计且不可 replay

### 4.40. approval-token delegation / 执行链可追溯性
如果一个 actor 批准了例外，而另一个 claw / bot / session 执行了它，系统应该保留 delegation chain，这样 policy exception 仍然能端到端归因。

要求行为：
- 记录 approver identity、requesting actor、executing actor，以及任何中间 relay / orchestrator 跳转
- 在 approval verification 和 token consumption events 中保留 delegation chain
- 区分直接自用和委托执行
- 暴露执行是否通过了非预期或未授权的 delegate

验收标准：
- 即使跨 bot / session 跳转，policy-exception 执行仍然可归因
- 审计可以回答“谁批准的”、“谁请求的”和“谁实际用了它”
- 委托式例外流程依然可治理，而不会坍缩成泛化的 bot activity

### 4.41. token 优化 / repo 作用域指引合约
新用户一上来就会遇到 token burn 和 context 膨胀，但产品表面并没有清楚解释 repo scope、ignore 路径和工作目录选择如何影响 clawability。

要求行为：
- 明确说明 `.clawignore` / `.claudeignore` / `.gitignore` 是否被遵守，以及具体如何遵守
- 在可能的情况下，给出一个简单建议：优先从最小可用子目录开始，而不是整个 monorepo
- 提供首次运行指引，建议排除高成本 / 生成型目录（`node_modules`、`dist`、`build`、`.next`、coverage、logs、dumps、generated reports`）
- 让节省 token 的 repo scope 指引出现在 onboarding / help 中，而不是藏在外部聊天建议里

验收标准：
- 新用户只看产品文档 / help 就能回答“我怎么才能不把垃圾拖进 context？”
- 首次使用时对 ignore file 和 repo scope 的困惑显著下降
- clawability 会在用户把明显可以避免的 junk 烧掉 token 之前就提升

### 4.42. workspace-scope 重量预览 / token 风险预检
在用户进入某个 repo 的 session 之前，claw-code 应该先给出一个轻量估算，告诉用户当前 workspace 有多重，以及为什么它可能很贵。

要求行为：
- 检查当前工作树里高风险的 token sink（巨大目录、生成物、vendored deps、logs、dumps）
- 在深入索引或第一次大 prompt 流程之前，总结可能导致 context 膨胀的来源
- 推荐更安全的作用域选择（例如更窄的子目录、ignore 模式、清理目标）
- 区分 `workspace 看起来很干净` 与 `workspace 很可能很快就会烧 token`

验收标准：
- 用户在不小心把整个垃圾场拿来 dogfood 之前，就会收到早期警告
- 节省 token 的指引变得具体而情境化，而不是泛泛的文档
- onboarding 能在可避免的 repo scope 错误变成成本 / 性能抱怨之前就把它们拦住

### 4.43. safer-scope quick-apply 动作
当系统警告当前 workspace 太重时，claw-code 应该提供一个直接采用更安全 scope 的方式，而不是把用户留给自己去手动理解建议。

要求行为：
- 把 scope 建议变成可操作的选择（例如切换到子目录、生成 ignore stub、排除检测到的重路径）
- 在应用变更前预览会被包含 / 排除的内容
- 保留一条容易回到原始更大 scope 的路径
- 区分 advisory 建议与用户确认过的 scope 变更

验收标准：
- 用户可以一步从“这个 workspace 太重了”变成“使用这个更安全的 scope”
- token 风险预检变成了操作指引，而不只是警告文本
- 首次运行用户不会再卡在诊断与手工清理之间

### 4.44.5. ship / provenance 不透明性 — 已于 2026-04-20 实现

**状态：** 事件已在 `lane_events.rs` 中实现。surface 现在会发出结构化的 ship provenance。

当 dogfood 工作落到 `main` 时，delivery path（scoped branch → PR → merge → push vs direct push）以及实际 ship 的 commit 集合并没有被作为一等事件暴露出来。这太容易让人丢失三件事之间的边界：“dogfood fix 已经落地”、“到底 ship 了哪些 commit”以及“实际使用了什么 review / merge 路径”。2026-04-20 dogfood 期间的 56 个 commit 推送（#122/#127/#129/#130/#131/#132）暴露了这个缺口：工作一开始是 scoped pinpoint branches，后来却收缩成直接 `origin/main` push，而且没有结构化 provenance trail。

**已实现的行为：**
- `ship.prepared` event —— 确认了发版意图
- `ship.commits_selected` event —— 锁定了 commit 范围
- `ship.merged` event —— merge 已完成并携带元数据
- `ship.pushed_main` event —— 确认已投递到 main
- 所有事件都携带 `ShipProvenance { source_branch, base_commit, commit_count, commit_range, merge_method, actor, pr_number }`
- `ShipMergeMethod` 枚举：`direct_push`、`fast_forward`、`merge_commit`、`squash_merge`、`rebase_merge`

要求行为：

当 dogfood 工作落到 `main` 时，delivery path（scoped branch → PR → merge → push vs direct push）和实际 ship 的 commit 集合不应只是作为一等事件出现。这让人很容易丢失三件事之间的边界：“dogfood fix 已经落地”、“到底 ship 了哪些 commit”以及“实际使用了什么 review / merge 路径”。2026-04-20 dogfood 期间的 56 个 commit 推送（#122/#127/#129/#130/#131/#132）暴露了这个缺口：工作一开始是 scoped pinpoint branches，后来却收缩成直接 `origin/main` push，而且没有结构化 provenance trail。

要求行为：
- 发出 `ship.provenance` event，包含：source branch、merge method（PR #、direct push、fast-forward）、commit range（first..last）和 actor
- 区分 `intentional.ship`（像 #122-#132 这样的明确交付）与 `incidental.rider`（同一 push 里的其他 commit）
- 在 lane events 和 `claw state` 输出中暴露这些信息
- clawhip 可以直接说“6 个 pinpoints shipped，50 个 riders，通过 direct push”而无需 git 考古

验收标准：
- 无需事后人工重建，就能回答“刚刚 ship 了什么、通过了哪条路径”
- delivery path 是机器可读且可审计的

来源：gaebal-gajae 于 2026-04-20 的 dogfood 观察——正是那次运行暴露了这个缺口。

**2026-04-20 识别出的未完成 gap：**
`sla` —— 我们保留原文含义 —— schema 和事件构造器已经在 `lane_events.rs::ShipProvenance` 和 `LaneEvent::ship_*()` 方法中实现。**缺失的是 wiring。** rusty-claude-cli 中的 git push 操作还没有发出这些事件。执行 `git push origin main` 时，observability layer 并不会收到 `ship.prepared/commits_selected/merged/pushed_main` 事件。事件仍然只是死代码（仅测试使用）。

**下一个 pinpoint（§4.44.5.1）：** Ship event wiring
把 `LaneEvent::ship_*()` 的发射接到实际的 git push 调用点：
1. 在 `main.rs`、`tools/lib.rs` 或 `worker_boot.rs` 中定位 `git push origin <branch>` 的执行位置
2. 在 push 前后拦截：在 merge 前发 `ship.prepared`，锁定范围时发 `ship.commits_selected`，merge 后发 `ship.merged`，push 到 `origin/main` 后发 `ship.pushed_main`
3. 捕获真实元数据：`source_branch`、`commit_range`、`merge_method`、`actor`、`pr_number`
4. 将事件路由到 lane event stream
5. 验证 `claw state` 输出能展示 ship provenance

验收标准：git push 会发出全部 4 个事件并带有真实元数据，`claw state` JSON 会包含 `ship` provenance。

### 4.44. Typed-error envelope 合约（静默状态清单汇总）
Claw-code 目前把所有错误类别——filesystem、auth、session、parse、runtime、MCP、usage——都压扁成同一个损失严重的 `{type:"error", error:"<prose>"}` envelope。人类 operator 和下游 claws 都失去了程序化判断“哪个操作失败了、哪个路径 / 资源失败了、失败类型是什么、是否可重试、是否可行动、还是终局”的能力。这个汇总锁定了 typed-error 合约，它关闭了目前散落在 **#102 + #129**（MCP readiness 不透明）、**#127 + #245**（delivery surface 不透明）以及 **#121 + #130**（error-text 谎言 / errno 剥离上下文）中的一组 pinpoints。

要求行为：
- 结构化 `error.kind` 枚举：至少包括 `filesystem | auth | session | parse | runtime | mcp | delivery | usage | policy | unknown`（可扩展）
- `error.operation` 字段，命名失败的 syscall / method（例如 `"write"`、`"open"`、`"resolve_session"`、`"mcp.initialize_handshake"`、`"deliver_prompt"`）
- `error.target` 字段，命名失败资源（fs error 用 path，session error 用 session-id，MCP error 用 server-name，delivery error 用 channel-id）
- `error.errno` / `error.detail` 字段，承载平台相关的底层细节（作为嵌套诊断数据，而不是整个面向用户的 surface）
- `error.hint` 字段，给出可操作的下一步（`"intermediate directory does not exist; try mkdir -p"`、`"export ANTHROPIC_AUTH_TOKEN"`、`"this session id was already cleared via /clear; try /session list"`）
- `error.retryable` 布尔值，指示下游自动化是否可以安全重试而无需 operator 干预
- 文本模式渲染保留这五个字段的 operator 可读 prose；JSON 模式渲染则把它们暴露为结构化子字段
- 只有当 `error.kind == usage` 时才附加 `Run claw --help for usage` 尾巴，而不是把它加到 filesystem、auth、session、MCP 或 runtime 错误里误导 operator
- 向后兼容：保留顶层 `{error: "<prose>", type: "error"}` 形状，这样依赖字符串解析 envelope 的现有 claws 仍可继续工作；新字段只做增量扩展
- 用 golden-fixture tests 锁定回归——矩阵中的每个（verb、error-kind）单元格都有一个 fixture 文件来捕获精确的 envelope 形状
- `kind` 枚举与 schema registry（Phase 2 §2）一起注册，以便下游消费者协商它们理解的版本

验收标准：
- 消费 `--output-format json` 的 claw 可以按 `error.kind` 分支，决定重试、升级还是终止，而不需要正则抓取 prose
- `claw export --output /tmp/nonexistent/dir/out.md` 返回 `{error:{kind:"filesystem",operation:"write",target:"/tmp/nonexistent/dir/out.md",errno:"ENOENT",hint:"intermediate directory does not exist; try mkdir -p /tmp/nonexistent/dir first",retryable:true},type:"error"}`，而不是 `{error:"No such file or directory (os error 2)",type:"error"}`
- `claw "prompt"` 在缺少凭证时返回 `{error:{kind:"auth",operation:"resolve_anthropic_auth",target:"ANTHROPIC_AUTH_TOKEN",hint:"export ANTHROPIC_AUTH_TOKEN or ANTHROPIC_API_KEY",retryable:false},type:"error"}`，而不是现在那种裸 prose
- `claw --resume does-not-exist /status` 返回 `{error:{kind:"session",operation:"resolve_session_id",target:"does-not-exist",hint:"managed sessions live in .claw/sessions/; try latest or /session list",retryable:false},type:"error"}`
- 这些 cluster pinpoints（#102、#121、#127、#129、#130、#245）最终都收敛成符合此 envelope 合约的独立修复工作
- 在 80%+ 的错误路径里，`Run claw --help for usage` 尾巴会消失，因为它不再误导
- 监控 / observability 工具可以构建 typed dashboards（`group by error.kind`、`count where error.kind="mcp" AND error.operation="initialize_handshake"`），而不需要不断做 regex 变换

为什么这是自然的汇总：
- 六个 pinpoints（#102、#121、#127、#129、#130、#245）本质上是同一种根病：重要失败状态没有以类型化、结构化、对 operator 有用的结果发出
- 如果把每个 pinpoint 单独修，可能会出现六种 ad-hoc envelope 形状；先锁定合约可以保证它们收敛到同一个地方
- 这份合约就是 Phase 2 §4 “Canonical lane event schema”的展示样例——typed errors 是 typed lane events 的前提
- 它与产品原则 #5（部分成功是一等公民）一致，因为它让部分失败状态也能被机器读取

**来源。** 这份草案于 2026-04-20 与 gaebal-gajae 一起在 clawcode-dogfood cycle（`#clawcode-building-in-public` 频道）中起草，原因是 #130 的 filing 暴露出了与 gaebal-gajae 的 #245 控制平面 delivery 不透明同样的 envelope flattening 模式。cluster bundle：**#102 + #121 + #127 + #129 + #130 + #245** —— 六个 pinpoints 都提供了证据；本 §4.44 条目锁定的是一个 contract，后续每个 pinpoint 的修复工作都必须符合它。它与下面的 **§5 Failure taxonomy** 是兄弟关系——§5 列出 failure CLASS 名称；§4.44 则规定 envelope 的 SHAPE，它携带 class 以及 operation、target、hint、errno 和 retryable 信号。

### 5. Failure taxonomy
规范化 failure class：
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
- blocker 会被机器分类
- 仪表盘和重试策略可以按 failure type 分支

### 5.5. transport outage 与 lane failure 的边界
当控制 server 或 transport 挂掉时，claw-code 应该区分 host-level outage 与 lane-local failure，而不是让所有 active lane 看起来都同样坏掉。

要求行为：
- 发出类型化 transport outage 事件，与 lane failure 事件分离
- 给受影响的 lane 标注依赖状态（`blocked_by_transport`），而不是把它们改写成普通 lane error
- 保留 transport 丢失前最后已知的 lane good state
- 暴露 outage 范围（`single session`、`single worker host`、`shared control server`）

验收标准：
- clawhip 可以说“server down blocked 3 lanes”，而不是假装发生了 3 个独立 lane failure
- 恢复策略可以把 transport 重启与 lane-local recovery recipe 分开
- 事后分析可以区分基础设施 blast radius 与真实代码 lane defect

### 6. 可操作摘要压缩
把噪声事件流压缩成：
- 当前 phase
- 上一个成功 checkpoint
- 当前 blocker
- 推荐的下一步恢复动作

验收标准：
- channel 状态更新保持简短且与机器事实对齐
- claws 不再从原始 build spam 中猜测状态

### 140. 已弃用 `permissionMode` 迁移会把 `DangerFullAccess` 静默降级为 `WorkspaceWrite`

**录入时间：** 2026-04-21，来自 dogfood cycle — `main` HEAD 上的 `cargo test --workspace`（`36b3a09`）显示 1 个确定性失败。

**问题：** `tests::punctuation_bearing_single_token_still_dispatches_to_prompt` 失败，并给出：
```
assert left == right failed
  left:  ... permission_mode: WorkspaceWrite ...
  right: ... permission_mode: DangerFullAccess ...
warning: .claw/settings.json: field "permissionMode" is deprecated (line 1). Use "permissions.defaultMode" instead
```
测试 fixture 写入了一个带有已弃用 `permissionMode: "dangerFullAccess"` 键的 `.claw/settings.json`。迁移 / 弃用 shim 虽然读到了它，但解析结果却变成了 `WorkspaceWrite`，而不是 `DangerFullAccess`。结果是：`cargo test --workspace` 在 `main` 上红了，172 通过，1 失败。

**根因假设：** `parse_args` 或 `ConfigLoader` 里的已弃用字段读取路径，把 `permissionMode` 的值交给了一个权限模式 resolver，而这个 resolver 没有把 `"dangerFullAccess"` 映射到 `PermissionMode::DangerFullAccess`，很可能是默认值或回退成了 `WorkspaceWrite`。

**修复形态：**
- 确保弃用 key 的迁移路径会正确把 `permissionMode: "dangerFullAccess"` 映射成 `PermissionMode::DangerFullAccess`（与 `permissions.defaultMode: "dangerFullAccess"` 一致）
- 或者，改测试 fixture 直接使用 canonical 的 `permissions.defaultMode` 键，这样它检验的是迁移 shim，而不是依赖它来完成转换
- 验证 `cargo test --workspace` 返回 0 失败

**验收标准：**
- `cargo test --workspace` 在 `main` 上通过，0 失败
- 已弃用的 `permissionMode: "dangerFullAccess"` 能平滑迁移到 `DangerFullAccess`，而不会降级成 `WorkspaceWrite`

### 137. 测试套件中的模型别名简写回归 — 在 `feat/134-135-session-identity` 分支上裸别名解析失效

**录入时间：** 2026-04-21，来自 dogfood cycle — `feat/134-135-session-identity` HEAD（`91ba54d`）上的 `cargo test --workspace` 显示 3 个失败测试。

**问题：** `tests::parses_bare_prompt_and_json_output_flag`、`tests::multi_word_prompt_still_uses_shorthand_prompt_mode` 和 `tests::env_permission_mode_overrides_project_config_default` 都因以下错误 panic：
```
args should parse: "invalid model syntax: 'claude-opus'. Expected provider/model (e.g., anthropic/claude-opus-4-6) or known alias (opus, sonnet, haiku)"
```
#134/#135 的 session-identity 工作收紧了 model syntax 验证，但测试 fixture 仍然传入 `claude-opus` 这种新的 validator 会拒绝的裸字符串。162 个测试通过；只有这 3 个使用旧式裸别名 model 名称的测试失败。

**修复形态：**
- 把这 3 个失败测试 fixture 改成使用合法 alias（`opus`、`sonnet`、`haiku`）或完整模型 id（`anthropic/claude-opus-4-6`）
- 或者，如果 `claude-opus` 确实应该被支持，就把它加入 alias registry
- 在把 feat 分支合并到 `main` 之前验证 `cargo test --workspace` 返回 0 失败

**验收标准：**
- `feat/134-135-session-identity` 分支上的 `cargo test --workspace` 通过，0 失败
- 当前已经通过的 162 个测试不回退

### 133. 被阻止状态的 subphase 合约（原 §6.5）
**录入时间：** 2026-04-20，来自 dogfood cycle — 上一轮已经识别出 §4.44.5 的 provenance gap，这一轮则聚焦 §6.5 的实现。

**问题：** 目前 `lane.blocked` 只是一个单一而不透明的状态。恢复 recipe 无法区分 trust-gate blocker、MCP handshake failure、branch freshness issue 或测试挂起。所有 blocked lane 看起来都一样，只能靠 pane scraping 做 triage。

**具体实现：**
当 lane 处于 `blocked` 时，也要暴露它停止推进的精确 subphase，而不是逼 claws 从日志里自己猜。

subphase 至少应包括：
- `blocked.trust_prompt`
- `blocked.prompt_delivery`
- `blocked.plugin_init`
- `blocked.mcp_handshake`
- `blocked.branch_freshness`
- `blocked.test_hang`
- `blocked.report_pending`

验收标准：
- `lane.blocked` 携带稳定的 subphase enum + 简短的人类摘要
- clawhip 可以直接说“blocked at MCP handshake”或“blocked waiting for trust clear”，而无需 pane scraping
- 重试可以指向正确的恢复 recipe，而不是把所有 blocked 状态一视同仁

## Phase 3 — branch / test 感知与自动恢复

### 7. 在广泛验证前检测 stale branch
在做广泛测试之前，把当前 branch 与 `main` 对比，检测是否缺少已知修复。

验收标准：
- 发出 `branch.stale_against_main`
- 根据 policy 建议或自动执行 rebase / merge-forward
- 避免把 stale branch 失败误判成新的回归

### 8. 常见失败的恢复 recipes
为以下情况编码自动恢复：
- trust prompt 未解决
- prompt 被投递到了 shell
- stale branch
- 跨 crate 重构后编译变红
- MCP 启动握手失败
- 部分插件启动失败

验收标准：
- 在升级之前会先自动尝试一次恢复
- 该恢复尝试本身也会作为结构化事件数据发出

### 8.5. 恢复尝试账本
暴露机器可读的恢复进展，这样 claws 可以看到已经尝试过哪些自动恢复、哪些还在运行，以及为什么会升级。

账本至少应包含：
- recovery recipe id
- 尝试次数
- 当前 recovery 状态（`queued`、`running`、`succeeded`、`failed`、`exhausted`）
- 开始 / 结束时间戳
- 最近一次失败摘要
- 当重试停止时的升级原因

验收标准：
- clawhip 可以直接报告“auto-recover tried prompt replay twice, then escalated”，而无需日志考古
- operator 可以区分“没有尝试恢复”与“恢复已经耗尽”
- 重复的静默重试循环会变得可见且可审计

### 9. Green-ness 合约
Worker 应区分：
- targeted tests green
- package green
- workspace green
- merge-ready green

验收标准：
- 不再出现含糊的“tests passed”表述
- merge policy 可以针对不同 lane type 要求正确的 green level
- 单个卡住的测试不能掩盖其他失败：在 CI（`cargo test --workspace`）中对每个测试强制 timeout，这样某个 crate 里 6 分钟的挂起不会阻止后续 crate 跑完测试
- 当 CI job 因为 hang 而失败时，worker 必须把它报告成 `test.hung`，而不是泛化的 failure，这样 triage 才不会把它和普通 `assertion failed` 混为一谈
- 记录的 pinpoint（2026-04-08）：`be561bf` 把本地字节估算 preflight 换成了 `count_tokens` round-trip，并在任何错误时都静默返回 `Ok(())`，导致 `send_message_blocks_oversized_*` 每次尝试都挂起约 6 分钟；随后 workspace job crash 又掩盖了 6 个早已存在、彼此独立的 CLI 回归（compact flag 被丢弃、管道 stdin 与 permission prompter、旧 session 布局、help/prompt 断言、mock harness count），这些问题只有在 `8c6dfe5` + `5851f2d` 恢复快速失败路径后才变得可诊断

## Phase 4 — 以 claws 为中心的任务执行

### 10. 结构化 task packet 格式
定义一个结构化 task packet，字段示例：
- objective
- scope
- repo / worktree
- branch policy
- acceptance tests
- commit policy
- reporting contract
- escalation policy

验收标准：
- claws 可以在不完全依赖超长自然语言 prompt blob 的情况下分发工作
- task packet 可以被记录、重试并安全转换

### 11. 自动编码 policy engine
编码自动化规则，例如：
- 如果 green + scoped diff + review 通过 -> 合并到 dev
- 如果 branch stale -> 先 merge-forward 再做广泛测试
- 如果启动被阻塞 -> 先恢复一次，然后升级
- 如果 lane 完成 -> 发出 closeout 并清理 session

验收标准：
- doctrine 从聊天说明迁移到可执行规则

### 12. claw-native 仪表盘 / lane board
暴露一个机器可读的 board，内容包括：
- repos
- 活跃 claws
- worktree
- branch freshness
- red / green 状态
- 当前 blocker
- merge readiness
- 最后一个有意义的事件

验收标准：
- claws 可以直接查询状态
- 面向人类的视图只是一个渲染层，而不是事实来源

### 12.5. running-state liveness heartbeat
当 lane 处于 `working` 或其他进行中状态时，发出一个轻量的 liveness heartbeat，让 claws 区分安静的进展与静默挂起。

heartbeat 至少应包含：
- 当前 phase / subphase
- 距离上一次有意义进展的秒数
- 距离上一次 heartbeat 的秒数
- 当前 active step label
- 是否预期还有后台工作

验收标准：
- clawhip 可以区分“安静但还活着”与“working state 已失活”
- stale 检测不再只依赖原始 pane churn
- 长时间运行的编译 / 测试 / 后台步骤可以在不靠日志 scraping 的情况下保持机器可见

## Phase 5 — 插件和 MCP 生命周期成熟度

### 13. first-class 插件 / MCP 生命周期合约
每个插件 / MCP 集成都应该暴露：
- config validation 合约
- 启动 healthcheck
- discovery 结果
- 降级模式行为
- shutdown / cleanup 合约

验收标准：
- 部分启动和单个 server 失败会以结构化方式报告
- 即使一个 server 失败，成功的 server 仍然可以使用

### 14. MCP 端到端生命周期对齐
补齐以下环节的差距：
- config load
- server 注册
- spawn / connect
- initialize handshake
- tool / resource discovery
- 调用路径
- 错误呈现
- shutdown / cleanup

验收标准：
- parity harness 和 runtime tests 同时覆盖健康启动和降级启动场景
- 出问题的 server 会被暴露为结构化失败，而不是不透明警告

## 立即待办（来自当前真实痛点）

优先级顺序：P0 = 阻塞 CI / green 状态，P1 = 阻塞集成 wiring，P2 = clawability 加固，P3 = swarm 效率提升。

**P0 — 先修（CI 可靠性）**
1. 将 `render_diff_report` 测试隔离到 tmpdir —— **已完成**：`render_diff_report_for()` 测试现在在临时 git repo 中运行，而不是在真实工作树里，针对性的 `cargo test -p rusty-claude-cli render_diff_report -- --nocapture` 在 branch / worktree 活动期间也能保持绿色
2. 将 GitHub CI 从单 crate 覆盖扩展到 workspace 级验证 —— **已完成**：`.github/workflows/rust-ci.yml` 现在运行 `cargo test --workspace`，并在 workspace 级别执行 fmt / clippy
3. 添加 release-grade binary workflow —— **已完成**：`.github/workflows/release.yml` 现在会为 CLI 构建带标签的 Rust release artifacts
4. 添加 container-first 测试 / 运行文档 —— **已完成**：`Containerfile` + `docs/container.md` 记录了构建、bind-mount 和 `cargo test --workspace` 的标准 Docker / Podman 工作流
5. 在 onboarding 文档和 help 中暴露 `doctor` / preflight diagnostics —— **已完成**：README + USAGE 现在把 `claw doctor` / `/doctor` 放到首次运行路径中，并指向内置的 preflight report
6. 在 CI 中自动检查 branding / source-of-truth 残留 —— **已完成**：`.github/scripts/check_doc_source_of_truth.py` 和 `doc-source-of-truth` CI job 现在会阻止已跟踪文档与元数据中的旧 repo / org / invite 残留
7. 消除首次运行 help / build 路径里的 warning spam —— **已完成**：当前 `cargo run -q -p rusty-claude-cli -- --help` 会干净地渲染帮助输出，不会在产品界面前堆一墙 warning
8. 将 `doctor` 从仅 slash 命令提升为顶层 CLI 入口 —— **已完成**：`claw doctor` 现在是本地 shell 入口，并有针对直接 help 和 health-report 输出的回归覆盖
9. 让机器可读的 status 命令真正机器可读 —— **已完成**：`claw --output-format json status` 和 `claw --output-format json sandbox` 现在输出结构化 JSON snapshot，而不是散文表格
10. 统一用户可见输出中的 legacy config / skill 命名空间 —— **已完成**：skills / help JSON / text 输出现在把 `.claw` 作为 canonical namespace，并把 legacy roots 折叠进 `.claw` 形状的 source id / label 中
11. 让像 `skills` 和 `mcp` 这样的 inventory 命令尊重 JSON 输出 —— **已完成**：直接 CLI inventory 命令现在会对 `--output-format json` 做出响应，并为 skills 和 MCP inventory 输出结构化 payload
12. 审计整个 CLI 表面的 `--output-format` 合约 —— **已完成**：直接 CLI 命令现在在 help / version / status / sandbox / agents / mcp / skills / bootstrap-plan / system-prompt / init / doctor 上都遵守确定性的 JSON / text 处理，且在 `output_format_contract.rs` 和 resumed `/status` JSON 覆盖中有回归测试

**P1 — 接下来（集成 wiring，解除验证阻塞）**
1. Worker readiness handshake + trust resolution —— **已完成**：`WorkerStatus` 状态机具备 `Spawning` → `TrustRequired` → `ReadyForPrompt` → `PromptAccepted` → `Running` 生命周期，且有 `trust_auto_resolve` + `trust_gate_cleared` 门控
2. 添加跨模块集成测试 —— **已完成**：12 个集成测试覆盖了 worker→recovery→policy、stale_branch→policy、green_contract→policy、reconciliation flows
3. 接入 lane-completion emitter —— **已完成**：`lane_completion` 模块通过 `detect_lane_completion()` 自动根据 session-finished + tests-green + push-complete 将 `LaneContext::completed` 设为完成，然后交给 policy closeout
4. 将 `SummaryCompressor` 接入 lane event pipeline —— **已完成**：`compress_summary_text()` 会把结果喂给 `tools/src/lib.rs` 中的 `LaneEvent::Finished` detail field

**P2 — clawability 加固（原始 backlog）**
5. Worker readiness handshake + trust resolution —— **已完成**：`WorkerStatus` 状态机具备 `Spawning` → `TrustRequired` → `ReadyForPrompt` → `PromptAccepted` → `Running` 生命周期，且有 `trust_auto_resolve` + `trust_gate_cleared` 门控
6. prompt misdelivery detection and recovery —— **已完成**：`prompt_delivery_attempts` 计数器、`PromptMisdelivery` 事件检测、`auto_recover_prompt_misdelivery` + `replay_prompt` 恢复分支
7. clawhip 中的 canonical lane event schema —— **已完成**：`LaneEvent` 枚举包含 `Started/Blocked/Failed/Finished` 变体，`LaneEvent::new()` 提供类型化构造器，`tools/src/lib.rs` 已集成
8. failure taxonomy + blocker normalization —— **已完成**：`WorkerFailureKind` 枚举（`TrustGate/PromptDelivery/Protocol/Provider`），以及 `FailureScenario::from_worker_failure_kind()` 到恢复 recipe 的桥接
9. 在 workspace tests 之前检测 stale branch —— **已完成**：`stale_branch.rs` 模块提供 freshness detection、ahead/behind 指标和 policy 集成
10. MCP 结构化降级启动报告 —— **已完成**：`McpManager` 的降级启动报告（`mcp_stdio.rs` 中 +183 行）、失败 server 分类（startup / handshake / config / partial），以及 tool 输出中的结构化 `failed_servers` + `recovery_recommendations`
11. Structured task packet 格式 —— **已完成**：`task_packet.rs` 模块提供 `TaskPacket` struct、校验、序列化、`TaskScope` 解析（workspace / module / single-file / custom），并已集成进 `tools/src/lib.rs`
12. Lane board / 机器可读状态 API —— **已完成**：lane completion 加固 + `LaneContext::completed` 自动检测 + MCP 降级报告 surface 共同提供机器可读状态
13. **Session completion failure classification** —— **已完成**：`WorkerFailureKind::Provider` + `observe_completion()` + recovery recipe bridge 已落地
14. **Config merge validation gap** —— **已完成**：`config.rs` 中在深度合并前加入了 hook validation（+56 行），格式错误的条目会带着 source-path 上下文失败，而不是在合并后才报 parse error
15. **MCP manager discovery flaky test** —— **已完成**：`manager_discovery_report_keeps_healthy_servers_when_one_server_fails` 在反复稳定通过后又恢复成普通 workspace test，因此降级启动覆盖不再被 `#[ignore]` 隐藏

16. **Commit provenance / worktree-aware push events** —— **已完成**：`LaneCommitProvenance` 现在在 lane events 中携带 branch / worktree / canonical-commit / supersession 元数据，并且在写入 agent manifests 之前会应用 `dedupe_superseded_commit_events()`，因此被 supersede 的 commit 事件会收敛到最新的 canonical lineage
17. **Orphaned module integration audit** —— **已完成**：`runtime` 现在把 `session_control` 和 `trust_resolver` 放在 `#[cfg(test)]` 下，直到它们接入真实的非测试执行路径；这样普通构建就不会再宣称存在死掉的 clawability surface area
18. **Context-window preflight gap** —— **已完成**：provider request sizing 现在会在 oversized request 离开进程前发出 `context_window_blocked`，并使用 model-context registry，而不再是旧的 naive max-token heuristic
19. **Subcommand help falls through into runtime/API path** —— **已完成**：`claw doctor --help`、`claw status --help`、`claw sandbox --help`，以及嵌套的 `mcp` / `skills` help 现在都会在本地被拦截，不会启动 runtime / provider，并且有回归测试覆盖这些直接 CLI path
20. **Session state classification gap（working vs blocked vs finished vs truly stale）** —— **已完成**：agent manifests 现在会推导出 `working`、`blocked_background_job`、`blocked_merge_conflict`、`degraded_mcp`、`interrupted_transport`、`finished_pending_report` 和 `finished_cleanable` 等 machine state，而 terminal-state 持久化会记录 commit provenance 和派生 state，让下游监控能够区分安静进展与真正空闲的 session
21. **Resumed `/status` JSON parity gap** —— **已完成**：已由更广泛的“Resumed local-command JSON parity gap”工作处理，该工作记录为 #26。已在 `main` HEAD `8dc6580` 上复核——`cargo test --release -p rusty-claude-cli resumed_status_command_emits_structured_json_when_requested` 干净通过（1 passed, 0 failed），因此 resumed `/status --output-format json` 现在与 fresh CLI path 使用同一套结构化 renderer。原先的失败（因为 resumed dispatch 回退到 prose 导致的 `expected value at line 1 column 1`）已不再复现。
22. **Opaque failure surface for session/runtime crashes** —— **已完成**：`error.rs` 中的 `safe_failure_class()` 将所有 API 错误归类为 8 个 user-safe class（`provider_auth`、`provider_internal`、`provider_retry_exhausted`、`provider_rate_limit`、`provider_transport`、`provider_error`、`context_window`、`runtime_io`）。`main.rs` 中的 `format_user_visible_api_error` 会为每个用户可见错误附加 session ID + request trace ID。覆盖测试为 `opaque_provider_wrapper_surfaces_failure_class_session_and_trace` 及另外 3 个相关测试。
23. **`doctor --output-format json` check-level structure gap** —— **已完成**：`claw doctor --output-format json` 现在会保留人类可读的 `message` / `report`，同时输出结构化的每项检查诊断（`name`、`status`、`summary`、`details`，以及 workspace path 和 sandbox fallback data 等类型化字段），并由 `output_format_contract.rs` 提供回归覆盖。
24. **插件生命周期 init/shutdown 测试在 workspace-parallel 执行下不稳定** —— dogfooding 发现 `build_runtime_runs_plugin_lifecycle_init_and_shutdown` 在 `cargo test --workspace` 下可能失败，但单独运行时通过，因为同级测试在 tempdir-backed shell init script path 上竞争。**已完成（2026-04-11 复核）：** 当前 mainline helpers 已经足够稳健地隔离 plugin lifecycle temp 资源，以至于 `cargo test -p rusty-claude-cli build_runtime_runs_plugin_lifecycle_init_and_shutdown -- --nocapture` 和 `cargo test -p plugins plugin_registry_runs_initialize_and_shutdown_for_enabled_plugins -- --nocapture` 都通过，而且当前的 `cargo test --workspace` 结果也把这两个测试标成绿色。除非出现新的并行执行复现，否则请把旧条目标记为过时。
25. **`plugins::hooks::collects_and_runs_hooks_from_enabled_plugins` 在 Linux CI 上 flake，根因是 stdin-write race 而不是缺少 exec bit** —— **已于 2026-04-08 在 `172a2ad` 完成**。dogfooding 在 `main` 上四次复现了这个问题（CI runs [24120271422](https://github.com/ultraworkers/claw-code/actions/runs/24120271422)、[24120538408](https://github.com/ultraworkers/claw-code/actions/runs/24120538408)、[24121392171](https://github.com/ultraworkers/claw-code/actions/runs/24121392171)、[24121776826](https://github.com/ultraworkers/claw-code/actions/runs/24121776826)），从第一次 flake 逐步升级为第三次 push 的确定性红灯。失败模式是 `PostToolUse hook .../hooks/post.sh failed to start for "Read": Broken pipe (os error 32)`，源自 `HookRunResult`。**最初诊断是错的。** 之前版本里记录的第一个假设，以及 commit `79da4b8` 上的根因说明，都认为 `rust/crates/plugins/src/hooks.rs` 里的 `write_hook_plugin` 只是写出了没有 execute bit 的 `.sh` 文件，而 `Command::new(path).spawn()` 又在 fork/exec 上发生竞争。基于这个假设先发了一个只修 chmod 的修复（`4f7b674`），结果 **在 CI run `24121776826` 上仍然失败**，并且还是同样的 `Broken pipe` 症状，这就证伪了 chmod-only 假设。**真实根因。** `rust/crates/plugins/src/hooks.rs` 里的 `CommandWithStdin::output_with_stdin` 无条件地把子进程 stdin pipe 上的 `write_all` 错误继续向上传播，包括 `std::io::ErrorKind::BrokenPipe`。测试 hook 脚本运行极快（`#!/bin/sh` + 一条 `printf`），所以 child 在 parent 写完大约 200 字节的 JSON hook payload 之前就退出并关闭了自己的 stdin。在 Linux 上 pipe 会立刻触发 `EPIPE`；而在 macOS 上 pipe 恰好会先把这点小 payload 缓冲完，所以这个 race 只在 ubuntu CI runner 上显现。parent 的 `write_all` 返回 `Err(BrokenPipe)`，`output_with_stdin` 把它当成 hook failure 返回，`run_command` 于是把这个 hook 归类成“failed to start”，虽然 child 实际上已经完整运行并把预期消息打印到了 stdout。**修复（commit `172a2ad`，覆盖 `4f7b674` force-push）。** 这次修复分三部分：（1）**真正修复** —— `output_with_stdin` 现在会匹配 `write_all` 的结果并专门吞掉 `BrokenPipe`，同时继续向上传播其他所有写错误；在吞掉 `BrokenPipe` 之后，代码仍然会调用 `wait_with_output()`，因此 stdout / stderr / exit code 仍会从正常退出的 child 中捕获。（2）**卫生加固** —— 新增 `make_executable` helper，使用 `std::os::unix::fs::PermissionsExt` 在 `#[cfg(unix)]` 下把每个生成的 `.sh` 设为 `0o755`。这只是防御性措施，用于未来非 sh hook runner，不是导致 CI 出问题的那个 bug。（3）**回归保护** —— 新的 `generated_hook_scripts_are_executable` 测试在 `#[cfg(unix)]` 下断言每个生成的 `.sh` 文件至少有一个 execute bit（`mode & 0o111 != 0`），这样未来的改动就不能悄悄把这个卫生修复回退。**验证。** `cargo test --release -p plugins` 35 项全通过，fmt 干净，`clippy -D warnings` 干净；CI run [24121999385](https://github.com/ultraworkers/claw-code/actions/runs/24121999385) 在 `main` 上首次就通过了这个 hotfix commit。**经验教训。** 来自 child-process spawn 路径的 `Broken pipe (os error 32)` 既可能意味着“无法 exec”，也可能意味着“exec 成功但 child 在 parent 写完 stdin 之前就退出了”。第一次推断时，因为 ROADMAP 的脚手架把人引向了 exec-bit 猜测，所以容易产生“could not exec”的误读；但真正的证伪来自实证 CI，而不是代码观察。请记住这个模式：当 pipe 错误出现在 fork/exec 路径上时，先看 `wait_with_output()` 实际报告了 child 什么，再把失败归因到权限或 issue。
26. **Resumed local-command JSON parity gap** —— **已完成**：直接 `claw --output-format json` 已经为 `sandbox`、`mcp`、`skills`、`version` 和 `init` 提供了结构化 renderer，但 resumed `claw --output-format json --resume <session> /…` 这条路径仍然会回退到 prose，因为 resumed slash dispatch 只对 `/status` 发 JSON。现在 resumed `/sandbox`、`/mcp`、`/skills`、`/version` 和 `/init` 都复用了与直接 CLI 对应命令相同的 JSON envelope，并在 `rust/crates/rusty-claude-cli/tests/resume_slash_commands.rs` 和 `rust/crates/rusty-claude-cli/tests/output_format_contract.rs` 中有回归覆盖。
