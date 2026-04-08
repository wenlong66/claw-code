# 以容器为先的 claw-code 工作流

在这份文档加入之前，这个仓库的 Rust 运行时里已经具备 **容器检测**：

- `rust/crates/runtime/src/sandbox.rs` 会检测 Docker / Podman / 容器标记，例如 `/.dockerenv`、`/run/.containerenv`、匹配的环境变量，以及 `/proc/1/cgroup` 线索。
- `rust/crates/rusty-claude-cli/src/main.rs` 会通过 `claw sandbox` / `cargo run -p rusty-claude-cli -- sandbox` 报告暴露这一状态。
- `.github/workflows/rust-ci.yml` 运行在 `ubuntu-latest` 上，但它**没有**定义 Docker 或 Podman 容器任务。
- 在这次改动之前，仓库里**没有**检查入库的 `Dockerfile`、`Containerfile` 或 `.devcontainer/` 配置。

这份文档补充了一个小型的入库 `Containerfile`，让 Docker 和 Podman 用户有一条统一的容器工作流。

## 这份容器镜像的用途

仓库根目录下的 [`../Containerfile`](../Containerfile) 会提供一个可复用的 Rust 构建 / 测试 shell，并附带这个工作区常用的额外包（`git`、`pkg-config`、`libssl-dev`、证书）。

它**不会**把仓库复制进镜像。推荐的方式是把你的 checkout 挂载到 `/workspace`，这样编辑内容始终保留在宿主机上。

## 构建镜像

从仓库根目录执行：

### Docker

```bash
docker build -t claw-code-dev -f Containerfile .
```

### Podman

```bash
podman build -t claw-code-dev -f Containerfile .
```

## 在容器中运行 `cargo test --workspace`

这些命令会挂载仓库、将 Cargo 构建产物保留在工作树之外，并且从 `rust/` 工作区目录运行。

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev \
  cargo test --workspace
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev \
  cargo test --workspace
```

如果你想要一次完全干净的重建，可以在 `cargo test --workspace` 前加上 `cargo clean &&`。

## 在容器里打开 shell

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

进入 shell 后可以运行：

```bash
cargo build --workspace
cargo test --workspace
cargo run -p rusty-claude-cli -- --help
cargo run -p rusty-claude-cli -- sandbox
```

`sandbox` 命令是一个很有用的健康检查：在 Docker 或 Podman 里，它应该报告 `In container true`，并列出运行时检测到的标记。

## 同时挂载这个仓库和另一个仓库

如果你想在保持 `claw-code` 本身可写挂载的同时，让 `claw` 针对另一个 checkout 运行：

### Docker

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -v "$HOME/src/other-repo":/repo \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

### Podman

```bash
podman run --rm -it \
  -v "$PWD":/workspace:Z \
  -v "$HOME/src/other-repo":/repo:Z \
  -e CARGO_TARGET_DIR=/tmp/claw-target \
  -w /workspace/rust \
  claw-code-dev
```

然后例如可以执行：

```bash
cargo run -p rusty-claude-cli -- prompt "概括 /repo 的内容"
```

## 注意事项

- Docker 和 Podman 使用同一个入库 `Containerfile`。
- Podman 示例里的 `:Z` 后缀用于 SELinux 重标记；在 Fedora / RHEL 系主机上请保留它。
- 使用 `CARGO_TARGET_DIR=/tmp/claw-target` 可以避免在绑定挂载的 checkout 中留下容器所有权的 `target/` 产物。
- 如果不是容器环境下做本地开发，请继续使用 [`../USAGE_cn.md`](../USAGE_cn.md) 和 [`../rust/README_cn.md`](../rust/README_cn.md)。
