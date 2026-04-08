# Claw Code

<p align="center">
  <a href="https://github.com/ultraworkers/claw-code">ultraworkers/claw-code</a>
  ·
  <a href="./USAGE_cn.md">使用指南</a>
  ·
  <a href="./rust/README_cn.md">Rust 工作区</a>
  ·
  <a href="./PARITY_cn.md">Parity</a>
  ·
  <a href="./ROADMAP_cn.md">路线图</a>
  ·
  <a href="https://discord.gg/5TUQKqFWd">UltraWorkers Discord</a>
</p>

<p align="center">
  <a href="https://star-history.com/#ultraworkers/claw-code&Date">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date&theme=dark" />
      <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date" />
      <img alt="Claw Code 的星标历史" src="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date" width="600" />
    </picture>
  </a>
</p>

<p align="center">
  <img src="assets/claw-hero.jpeg" alt="Claw Code" width="300" />
</p>

Claw Code 是 `claw` CLI agent harness 的公开 Rust 实现。
核心实现位于 [`rust/`](./rust)，本仓库当前的 source of truth 是 **ultraworkers/claw-code**。

> [!IMPORTANT]
> 如果你要开始使用，请先阅读 [`USAGE_cn.md`](./USAGE_cn.md) 获取构建、认证、CLI、session 和 parity harness 的工作流说明。构建完成后请先运行 `claw doctor` 作为健康检查；如需 crate 级别细节，请阅读 [`rust/README_cn.md`](./rust/README_cn.md)；如需了解当前 Rust 迁移进度，请查看 [`PARITY_cn.md`](./PARITY_cn.md)；如需容器优先工作流，请查看 [`docs/container_cn.md`](./docs/container_cn.md)。

## 当前仓库结构

- **`rust/`** —— canonical Rust 工作区和 `claw` CLI 二进制
- **`USAGE_cn.md`** —— 面向任务的当前产品使用指南
- **`PARITY_cn.md`** —— Rust 迁移的 parity 状态与迁移说明
- **`ROADMAP_cn.md`** —— 当前路线图和待清理 backlog
- **`PHILOSOPHY.md`** —— 项目理念与系统设计框架
- **`src/` + `tests/`** —— 配套的 Python / reference 工作区与审计辅助工具；它们不是主要运行时表面

## 快速开始

```bash
cd rust
cargo build --workspace
./target/debug/claw --help
./target/debug/claw prompt "summarize this repository"
```

使用 API key 或内置 OAuth 流程完成认证：

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# 或者
cd rust
./target/debug/claw login
```

运行工作区测试套件：

```bash
cd rust
cargo test --workspace
```

## 文档地图

- [`USAGE_cn.md`](./USAGE_cn.md) —— 快速命令、认证、session、配置、parity harness
- [`rust/USAGE_cn.md`](./rust/USAGE_cn.md) —— Rust 工作区补充说明
- [`rust/MOCK_PARITY_HARNESS_cn.md`](./rust/MOCK_PARITY_HARNESS_cn.md) —— 确定性的 mock service harness 细节
- [`rust/README_cn.md`](./rust/README_cn.md) —— crate 映射、CLI 表面、特性、工作区布局
- [`PARITY_cn.md`](./PARITY_cn.md) —— Rust 迁移的 parity 状态
- [`ROADMAP_cn.md`](./ROADMAP_cn.md) —— 当前路线图和开放的清理工作
- [`PHILOSOPHY.md`](./PHILOSOPHY.md) —— 项目为什么存在，以及如何运作

## 生态

Claw Code 与更广泛的 UltraWorkers 工具链一起在开放环境中构建：

- [clawhip](https://github.com/Yeachan-Heo/clawhip)
- [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)
- [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)
- [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex)
- [UltraWorkers Discord](https://discord.gg/5TUQKqFWd)

## 所有权 / 归属声明

- 本仓库 **不** 声称拥有原始 Claude Code 源材料的所有权。
- 本仓库 **不** 隶属于 Anthropic，也未获其认可或维护。
