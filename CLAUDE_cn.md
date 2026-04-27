# CLAUDE.md

本文件在你于本仓库工作时，为 Claude Code（claude.ai/code）提供指导。

## 检测到的技术栈
- 语言：Rust。
- 框架：未检测到受支持的脚手架标记。

## 验证
- 从 `rust/` 运行 Rust 验证：`cargo fmt`、`cargo clippy --workspace --all-targets -- -D warnings`、`cargo test --workspace`
- `src/` 和 `tests/` 都存在；当行为发生变化时，请同时更新这两个面向。

## 仓库结构
- `rust/` 包含 Rust 工作区和当前活跃的 CLI/运行时实现。
- `src/` 包含应与生成的指导和测试保持一致的源文件。
- `tests/` 包含应与代码变更一起审查的验证面。

## 工作约定
- 优先做小而可审阅的改动，并让生成的启动脚手架文件与实际仓库工作流保持一致。
- 将共享默认值保存在 `.claude.json` 中；把 `.claude/settings.local.json` 留给本机本地覆盖。
- 不要自动覆盖现有的 `CLAUDE.md` 内容；当仓库工作流发生变化时，应有意地更新它。
