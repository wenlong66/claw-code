# 模型兼容性指南

本文档描述了 OpenAI 兼容 provider 中针对不同模型的特殊处理。添加新模型或新 provider 时，请先阅读本指南，确保兼容性处理正确。

## 目录

- [概览](#概览)
- [模型特定处理](#模型特定处理)
  - [Kimi 模型（排除 is_error）](#kimi-模型排除-is_error)
  - [推理模型（剥离调参参数）](#推理模型剥离调参参数)
  - [GPT-5（max_completion_tokens）](#gpt-5max_completion_tokens)
  - [Qwen 模型（DashScope 路由）](#qwen-模型dashscope-路由)
- [实现细节](#实现细节)
- [添加新模型](#添加新模型)
- [测试](#测试)

## 概览

`openai_compat.rs` provider 会把 Claw Code 的内部消息格式转换为 OpenAI 兼容的 chat completion 请求。不同模型对以下内容有不同要求：

- 工具结果消息字段（`is_error`）
- 采样参数（temperature、top_p 等）
- token 上限字段（`max_tokens` vs `max_completion_tokens`）
- 基础 URL 路由

## 模型特定处理

### Kimi 模型（排除 is_error）

**受影响的模型：** `kimi-k2.5`、`kimi-k1.5`、`kimi-moonshot`，以及名称中包含 `kimi` 的任意模型（不区分大小写）

**行为：** 工具结果消息中会**排除** `is_error` 字段。

**原因：** Kimi 模型（通过 Moonshot AI 和 DashScope）会拒绝 `is_error` 字段，并返回 400 Bad Request 错误：
```json
{
  "error": {
    "type": "invalid_request_error",
    "message": "Unknown field: is_error"
  }
}
```

**检测方式：**
```rust
fn model_rejects_is_error_field(model: &str) -> bool {
    let lowered = model.to_ascii_lowercase();
    let canonical = lowered.rsplit('/').next().unwrap_or(lowered.as_str());
    canonical.starts_with("kimi-")
}
```

**测试：** 参见 `openai_compat.rs` 中的 `model_rejects_is_error_field_detects_kimi_models` 及相关测试。

---

### 推理模型（剥离调参参数）

**受影响的模型：**
- OpenAI：`o1`、`o1-*`、`o3`、`o3-*`、`o4`、`o4-*`
- xAI：`grok-3-mini`
- Alibaba DashScope：`qwen-qwq-*`、`qwq-*`、`qwen3-*-thinking`

**行为：** 以下调参参数会从请求中**移除**：
- `temperature`
- `top_p`
- `frequency_penalty`
- `presence_penalty`

**原因：** 推理 / chain-of-thought 模型使用固定采样策略，会因为这些参数而返回 400 错误。

**例外：** 对兼容模型，如果显式设置了 `reasoning_effort`，则会保留该字段。

**检测方式：**
```rust
fn is_reasoning_model(model: &str) -> bool {
    let canonical = model.to_ascii_lowercase()
        .rsplit('/')
        .next()
        .unwrap_or(model);
    canonical.starts_with("o1")
        || canonical.starts_with("o3")
        || canonical.starts_with("o4")
        || canonical == "grok-3-mini"
        || canonical.starts_with("qwen-qwq")
        || canonical.starts_with("qwq")
        || (canonical.starts_with("qwen3") && canonical.contains("-thinking"))
}
```

**测试：** 参见 `reasoning_model_strips_tuning_params`、`grok_3_mini_is_reasoning_model` 和 `qwen_reasoning_variants_are_detected` 测试。

---

### GPT-5（max_completion_tokens）

**受影响的模型：** 所有以 `gpt-5` 开头的模型

**行为：** 请求 payload 中使用 `max_completion_tokens`，而不是 `max_tokens`。

**原因：** GPT-5 模型要求使用 `max_completion_tokens` 字段。沿用旧的 `max_tokens` 会导致请求校验失败：
```json
{
  "error": {
    "message": "Unknown field: max_tokens"
  }
}
```

**实现方式：**
```rust
let max_tokens_key = if wire_model.starts_with("gpt-5") {
    "max_completion_tokens"
} else {
    "max_tokens"
};
```

**测试：** 参见 `gpt5_uses_max_completion_tokens_not_max_tokens` 和 `non_gpt5_uses_max_tokens` 测试。

---

### Qwen 模型（DashScope 路由）

**受影响的模型：** 所有以 `qwen` 为前缀的模型

**行为：** 路由到 DashScope（`https://dashscope.aliyuncs.com/compatible-mode/v1`），而不是默认 provider。

**原因：** Qwen 模型托管在阿里云的 DashScope 服务上，而不是 OpenAI 或 Anthropic。

**配置：**
```rust
pub const DEFAULT_DASHSCOPE_BASE_URL: &str = "https://dashscope.aliyuncs.com/compatible-mode/v1";
```

**认证：** 使用 `DASHSCOPE_API_KEY` 环境变量。

**注意：** 某些 Qwen 模型同时也是推理模型（见上面的 [推理模型](#推理模型剥离调参参数)），因此会同时应用两类处理。

## 实现细节

### 文件位置

所有模型特定逻辑都在：
```text
rust/crates/api/src/providers/openai_compat.rs
```

### 关键函数

| 函数 | 作用 |
|----------|---------|
| `model_rejects_is_error_field()` | 检测不支持工具结果中 `is_error` 的模型 |
| `is_reasoning_model()` | 检测需要剥离调参参数的推理模型 |
| `translate_message()` | 将内部消息转换为 OpenAI 格式（应用 `is_error` 逻辑） |
| `build_chat_completion_request()` | 构造完整请求 payload（应用所有模型特定逻辑） |

### Provider 前缀处理

所有模型检测函数都会先去掉 provider 前缀（例如 `dashscope/kimi-k2.5` → `kimi-k2.5`），再进行匹配：

```rust
let canonical = model.to_ascii_lowercase()
    .rsplit('/')
    .next()
    .unwrap_or(model);
```

这样可以确保无论模型是否带 provider 前缀，都能被一致识别。

## 添加新模型

在添加新模型支持时：

1. **确认该模型是否为推理模型**
   - 它是否会拒绝 temperature/top_p 参数？
   - 如有需要，将其加入 `is_reasoning_model()` 检测

2. **确认工具结果兼容性**
   - 它是否会拒绝 `is_error` 字段？
   - 如有需要，将其加入 `model_rejects_is_error_field()` 检测

3. **确认 token 上限字段**
   - 它是否需要 `max_completion_tokens` 而不是 `max_tokens`？
   - 更新 `max_tokens_key` 逻辑

4. **添加测试**
   - 为检测函数编写单元测试
   - 为 `build_chat_completion_request` 编写集成测试

5. **更新本文档**
   - 把模型加入受影响列表
   - 记录任何特殊行为

## 测试

### 运行模型特定测试

```bash
# 所有 OpenAI 兼容测试
cargo test --package api providers::openai_compat

# 指定测试类别
cargo test --package api model_rejects_is_error_field
cargo test --package api reasoning_model
cargo test --package api gpt5
cargo test --package api qwen
```

### 测试文件

- 单元测试：`rust/crates/api/src/providers/openai_compat.rs`（位于 `mod tests` 中）
- 集成测试：`rust/crates/api/tests/openai_compat_integration.rs`

### 验证模型检测

如果想在不发起 API 调用的情况下验证模型是否被正确识别，可以这样做：

```rust
#[test]
fn my_new_model_is_detected() {
    // is_error 处理
    assert!(model_rejects_is_error_field("my-model"));
    
    // 推理模型检测
    assert!(is_reasoning_model("my-model"));
    
    // Provider 前缀处理
    assert!(model_rejects_is_error_field("provider/my-model"));
}
```

---

*最后更新：2026-04-16*

如有问题或更新，请参考 `rust/crates/api/src/providers/openai_compat.rs` 中的实现。
