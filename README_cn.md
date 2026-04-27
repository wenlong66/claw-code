# Claw Code

<p align="center">
  <a href="https://github.com/ultraworkers/claw-code">ultraworkers/claw-code</a>
  ·
  <a href="./USAGE.md">Usage</a>
  ·
  <a href="./rust/README.md">Rust workspace</a>
  ·
  <a href="./PARITY.md">Parity</a>
  ·
  <a href="./ROADMAP.md">Roadmap</a>
  ·
  <a href="https://discord.gg/5TUQKqFWd">UltraWorkers Discord</a>
</p>

<p align="center">
  <a href="https://star-history.com/#ultraworkers/claw-code&Date">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date&theme=dark" />
      <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date" />
      <img alt="ultraworkers/claw-code 的 Star 历史" src="https://api.star-history.com/svg?repos=ultraworkers/claw-code&type=Date" width="600" />
    </picture>
  </a>
</p>

<p align="center">
  <img src="assets/claw-hero.jpeg" alt="Claw Code" width="300" />
</p>

Claw Code 是 `claw` CLI 代理框架的公开 Rust 实现。
规范实现位于 [`rust/`](./rust)，而本仓库当前的 source of truth 是 **ultraworkers/claw-code**。

> [!IMPORTANT]
> 构建、认证、CLI、会话和 parity harness 工作流请先从 [`USAGE.md`](./USAGE.md) 开始。构建完成后先运行 `claw doctor` 做健康检查；按 crate 级别的细节请查看 [`rust/README.md`](./rust/README.md)；当前 Rust 移植检查点见 [`PARITY.md`](./PARITY.md)；容器优先工作流见 [`docs/container.md`](./docs/container.md)。
>
> **ACP / Zed 状态：** `claw-code` 目前还没有随附 ACP/Zed 守护进程入口点。请运行 `claw acp`（或 `claw --acp`）查看当前状态，不要仅凭源码布局猜测；`claw acp serve` 目前只是一个可发现性别名，真正的 ACP 支持仍单独跟踪在 `ROADMAP.md` 中。

## 当前仓库结构

- **`rust/`** — 规范 Rust 工作区和 `claw` CLI 二进制
- **`USAGE.md`** — 面向当前产品表面的任务导向使用指南
- **`PARITY.md`** — Rust 移植对齐状态和迁移说明
- **`ROADMAP.md`** — 活跃路线图和清理待办
- **`PHILOSOPHY.md`** — 项目意图与系统设计框架
- **`src/` + `tests/`** — 配套的 Python/参考工作区和审计辅助工具；不是主要运行时表面

## 快速开始

> [!NOTE]
> [!WARNING]
> **`cargo install claw-code` 安装的是错误的东西。** crates.io 上的 `claw-code` crate 是一个已废弃的 stub，只会放置 `claw-code-deprecated.exe`，而不是 `claw`。运行它只会打印 `"claw-code has been renamed to agent-code"`。**不要使用 `cargo install claw-code`。** 请要么从源码构建（本仓库），要么安装上游二进制：
> ```bash
> cargo install agent-code   # 上游二进制 — 安装的是 'agent.exe'（Windows）/ 'agent'（Unix），不是 'agent-code'
> ```
> 本仓库（`ultraworkers/claw-code`）**只能从源码构建** — 请按照下面步骤操作。

```bash
# 1. 克隆并构建
git clone https://github.com/ultraworkers/claw-code
cd claw-code/rust
cargo build --workspace

# 2. 设置 API key（Anthropic API key — 不是 Claude 订阅）
export ANTHROPIC_API_KEY="sk-ant-..."

# 3. 验证一切是否正确连通
./target/debug/claw doctor

# 4. 运行一个 prompt
./target/debug/claw prompt "say hello"
```

> [!NOTE]
> **Windows（PowerShell）：** 二进制文件是 `claw.exe`，不是 `claw`。请使用 `.\target\debug\claw.exe`，或者直接运行 `cargo run -- prompt "say hello"` 来跳过路径查找。

### Windows 设置

**PowerShell 是受支持的 Windows 路径。** 用你顺手的 shell 即可。Windows 上常见的上手问题是：

1. **先安装 Rust** — 从 <https://rustup.rs/> 下载并运行安装程序。完成后请关闭并重新打开终端。
2. **确认 Rust 在 PATH 中：**
   ```powershell
   cargo --version
   ```
   如果失败，请重开终端，或根据 Rust 安装器输出完成 PATH 设置后再试。
3. **克隆并构建**（PowerShell、Git Bash 或 WSL 都可）：
   ```powershell
   git clone https://github.com/ultraworkers/claw-code
   cd claw-code/rust
   cargo build --workspace
   ```
4. **运行**（PowerShell — 注意 `.exe` 和反斜杠）：
   ```powershell
   $env:ANTHROPIC_API_KEY = "sk-ant-..."
   .\target\debug\claw.exe prompt "say hello"
   ```

**Git Bash / WSL** 只是可选替代方案，不是必需项。如果你更喜欢 bash 风格路径（`/c/Users/you/...` 而不是 `C:\Users\you\...`），Git Bash（随 Git for Windows 提供）很好用。在 Git Bash 中，`MINGW64` 提示符是预期内的正常现象，不是安装坏了。

## 构建后：定位二进制并验证

运行 `cargo build --workspace` 后，`claw` 二进制已经构建完成，但**不会**自动安装到系统里。下面说明它的位置以及如何验证构建成功。

### 二进制位置

在 `claw-code/rust/` 中运行 `cargo build --workspace` 后：

**Debug 构建（默认，更快编译）：**
- **macOS/Linux：** `rust/target/debug/claw`
- **Windows：** `rust/target/debug/claw.exe`

**Release 构建（优化过，更慢编译）：**
- **macOS/Linux：** `rust/target/release/claw`
- **Windows：** `rust/target/release/claw.exe`

如果你运行 `cargo build` 时没有加 `--release`，二进制会在 `debug/` 目录下。

### 验证构建成功

直接用路径测试二进制：

```bash
# macOS/Linux（debug 构建）
./rust/target/debug/claw --help
./rust/target/debug/claw doctor

# Windows PowerShell（debug 构建）
.\rust\target\debug\claw.exe --help
.\rust\target\debug\claw.exe doctor
```

如果这些命令成功，就说明构建可用。`claw doctor` 是你的第一个健康检查 —— 它会验证 API key、模型访问和工具配置。

### 可选：添加到 PATH

如果你希望在任意目录下直接运行 `claw`，可以选择以下任一方式：

**方案 1：符号链接（macOS/Linux）**
```bash
ln -s $(pwd)/rust/target/debug/claw /usr/local/bin/claw
```
然后重新加载 shell 并测试：
```bash
claw --help
```

**方案 2：使用 `cargo install`（所有平台）**

构建并安装到 Cargo 默认位置（`~/.cargo/bin/`，通常已在 PATH 中）：
```bash
# 在 claw-code/rust/ 目录下
cargo install --path . --force

# 然后在任意位置
claw --help
```

**方案 3：更新 shell 配置（bash/zsh）**

把这一行加到 `~/.bashrc` 或 `~/.zshrc`：
```bash
export PATH="$(pwd)/rust/target/debug:$PATH"
```

重新加载 shell：
```bash
source ~/.bashrc  # 或 source ~/.zshrc
claw --help
```

### 故障排查

- **“command not found: claw”** — 二进制位于 `rust/target/debug/claw`，但没有加入 PATH。请使用完整路径 `./rust/target/debug/claw`，或者按上面的方法创建符号链接/安装。
- **“permission denied”** — 在 macOS/Linux 上，如果可执行位没有设置，你可能需要 `chmod +x rust/target/debug/claw`（很少见）。
- **Debug vs. release** — 如果构建很慢，那你处于 debug 模式（默认）。给 `cargo build` 加上 `--release` 可以提升运行时性能，但构建本身会花 5–10 分钟。

> [!NOTE]
> **认证：** claw 需要一个 **API key**（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等）—— Claude 订阅登录不是受支持的认证路径。

在确认二进制可用后，运行工作区测试套件：

```bash
cd rust
cargo test --workspace
```

## 文档地图

- [`USAGE.md`](./USAGE.md) — 快速命令、认证、会话、配置、parity harness
- [`rust/README.md`](./rust/README.md) — crate 地图、CLI 表面、功能、工作区布局
- [`PARITY.md`](./PARITY.md) — Rust 移植的对齐状态
- [`rust/MOCK_PARITY_HARNESS.md`](./rust/MOCK_PARITY_HARNESS.md) — 确定性的 mock 服务 harness 细节
- [`ROADMAP.md`](./ROADMAP.md) — 活跃路线图和待清理工作
- [`PHILOSOPHY.md`](./PHILOSOPHY.md) — 项目为什么存在以及如何运作

## 生态

Claw Code 与更大的 UltraWorkers 工具链一起公开构建：

- [clawhip](https://github.com/Yeachan-Heo/clawhip)
- [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)
- [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)
- [oh-my-codex](https://github.com/Yeachan-Heo/oh-my-codex)
- [UltraWorkers Discord](https://discord.gg/5TUQKqFWd)

## 所有权 / 从属免责声明

- 本仓库**不**声称拥有原始 Claude Code 源材料的所有权。
- 本仓库**不隶属于、不受 Anthropic 背书、也不由 Anthropic 维护**。
