# Mock LLM parity harness

这一里程碑新增了一个确定性的 Anthropic-compatible mock service，以及一个可复现的 CLI harness，用于验证 Rust 版 `claw` 二进制。

## 产物

- `crates/mock-anthropic-service/` —— mock `/v1/messages` service
- `crates/rusty-claude-cli/tests/mock_parity_harness.rs` —— 端到端的 clean-environment harness
- `scripts/run_mock_parity_harness.sh` —— 便捷封装脚本

## 场景

该 harness 会在一个全新的工作区和隔离的环境变量下运行以下脚本化场景：

1. `streaming_text`
2. `read_file_roundtrip`
3. `grep_chunk_assembly`
4. `write_file_allowed`
5. `write_file_denied`
6. `multi_tool_turn_roundtrip`
7. `bash_stdout_roundtrip`
8. `bash_permission_prompt_approved`
9. `bash_permission_prompt_denied`
10. `plugin_tool_roundtrip`

## 运行

```bash
cd rust/
./scripts/run_mock_parity_harness.sh
```

行为检查清单 / parity diff：

```bash
cd rust/
python3 scripts/run_mock_parity_diff.py
```

场景到 PARITY 的映射存放在 `mock_parity_scenarios.json` 中。

## 手动启动 mock server

```bash
cd rust/
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

server 会打印 `MOCK_ANTHROPIC_BASE_URL=...`；把 `ANTHROPIC_BASE_URL` 指向这个 URL，并使用任意非空的 `ANTHROPIC_API_KEY` 即可。
