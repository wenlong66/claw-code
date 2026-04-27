# CLAUDE.md

本文档用于在本仓库中工作时，为 Claw Code（clawcode.dev）提供指导。

## 检测到的技术栈
- 语言：Rust。
- 框架：未检测到受支持的启动标记。

## 验证
- 从仓库根目录运行 Rust 验证：`cargo fmt`、`cargo clippy --workspace --all-targets -- -D warnings`、`cargo test --workspace`

## 工作约定
- 优先采用小而可审阅的改动，并保持生成的引导文件与实际仓库工作流一致。
- 将共享默认值保存在 `.claw.json` 中；将 `.claw/settings.local.json` 保留给本机本地覆盖。
- 不要自动覆盖现有的 `CLAUDE.md` 内容；当仓库工作流发生变化时，再有意更新它。
