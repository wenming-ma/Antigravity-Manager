# Cursor IDE 适配指南

本文档记录了将 Antigravity Manager 代理服务适配到 Cursor IDE 的经验和最佳实践。

## 背景

Cursor IDE 使用 OpenAI 兼容的 API 格式与 AI 后端通信。当我们将 Gemini API 响应转换为 OpenAI 格式时，需要确保完全符合 OpenAI 规范，否则 Cursor 会出现解析错误。

## 核心问题与解决方案

### 1. Tool Call Index 必须唯一递增

**问题现象**
```
Error: Unexpected JSON token at position xxx
```

Cursor 在解析多个工具调用时报 JSON 解析错误。

**根本原因**

OpenAI 规范要求每个 `tool_calls[].index` 必须是唯一的递增整数。当多个工具调用的 index 都是 `0` 时，Cursor 会将它们的 `arguments` 字符串拼接在一起，导致 JSON 格式损坏。

**错误代码**
```rust
// ❌ 所有工具调用的 index 都是 0
let tool_call_chunk = json!({
    "choices": [{
        "delta": {
            "tool_calls": [{
                "index": 0,  // 始终是 0
                // ...
            }]
        }
    }]
});
```

**修复方案**
```rust
// ✅ 使用 HashMap 跟踪每个工具调用的唯一索引
let mut tool_call_index_map: HashMap<String, usize> = HashMap::new();
let mut next_tool_call_index: usize = 0;

// 为每个工具调用分配唯一索引
let tool_index = tool_call_index_map
    .entry(call_key.clone())
    .or_insert_with(|| {
        let idx = next_tool_call_index;
        next_tool_call_index += 1;
        idx
    });

let tool_call_chunk = json!({
    "choices": [{
        "delta": {
            "tool_calls": [{
                "index": *tool_index,  // 递增: 0, 1, 2...
                // ...
            }]
        }
    }]
});
```

**修改位置**: `src-tauri/src/proxy/mappers/openai/streaming.rs`

---

### 2. Thinking Signature 处理

**问题现象**
```
400 Bad Request: Invalid `signature`: thinking.signature must be valid
```

Gemini 思维模型要求历史消息中的思考块必须带有有效的 `thoughtSignature`。

**解决方案**

1. **捕获签名**: 从 Gemini 响应中提取 `thoughtSignature` 并存储到全局变量
2. **注入签名**: 在后续请求中，为历史思考块注入存储的签名
3. **失效清除**: 当签名失效时（400 错误），自动清除并重试

```rust
// 存储签名
pub fn store_thought_signature(sig: &str) {
    if let Ok(mut guard) = get_thought_sig_storage().lock() {
        *guard = Some(sig.to_string());
    }
}

// 清除失效签名
pub fn clear_thought_signature() {
    if let Ok(mut guard) = get_thought_sig_storage().lock() {
        *guard = None;
    }
}
```

**修改位置**: 
- `streaming.rs` - 捕获和存储签名
- `request.rs` - 注入签名到历史消息

---

### 3. 空白内容过滤

**问题现象**
```
400 Bad Request: messages: text content blocks must contain non-whitespace text
```

**原因**: Gemini API 不接受空白或纯空格的文本块。

**修复方案**
```rust
// ❌ 错误: 只检查 is_empty()
if !s.is_empty() {
    parts.push(json!({"text": s}));
}

// ✅ 正确: 使用 trim().is_empty() 检查
if !s.trim().is_empty() {
    parts.push(json!({"text": s}));
}
```

**修改位置**: `src-tauri/src/proxy/mappers/openai/request.rs`

---

### 4. Args 类型处理

**问题现象**: 工具调用参数格式错误。

**原因**: Gemini 返回的 `args` 可能是 JSON 对象或已经是字符串，需要正确识别。

**修复方案**
```rust
// ✅ 正确处理不同类型的 args
let args = match func_call.get("args") {
    Some(Value::String(s)) => s.clone(),       // 已是字符串
    Some(v) if v.is_null() => "{}".to_string(),
    Some(v) => v.to_string(),                  // 对象需要序列化
    None => "{}".to_string(),
};
```

**修改位置**: `src-tauri/src/proxy/mappers/openai/streaming.rs`

---

## 数据流概览

```
Cursor (OpenAI 格式)
       │
       ▼
┌──────────────────────────────┐
│  transform_openai_request()  │  request.rs
│  - 空白内容过滤              │
│  - 思考签名注入              │
└──────────────────────────────┘
       │
       ▼
     Gemini API
       │
       ▼
┌──────────────────────────────┐
│  create_openai_sse_stream()  │  streaming.rs
│  - 捕获 thoughtSignature     │
│  - 工具调用 index 递增       │
│  - args 类型处理             │
└──────────────────────────────┘
       │
       ▼
Cursor (OpenAI 格式)
```

---

## 测试验证

### 多工具调用测试

发送包含多个工具调用的请求，验证 Cursor 能正确解析：

```bash
# 预期: Cursor 能正常显示多个文件操作
# 验证: 检查日志中的 tool_calls[].index 是否递增
```

### 思考模型测试

使用 Gemini 3 Pro 或 Claude Thinking 模型进行多轮对话：

```bash
# 预期: 历史消息中的思考块能正确注入签名
# 验证: 无 400 签名错误
```

---

## 相关文件

| 文件 | 职责 |
|------|------|
| `streaming.rs` | Gemini → OpenAI 响应转换 |
| `request.rs` | OpenAI → Gemini 请求转换 |
| `openai.rs` | 错误处理和重试逻辑 |

---

## 注意事项

1. **保持同步**: 合并上游代码时，需确保这些修复不被覆盖
2. **日志调试**: 添加 `[OpenAI-SSE]` 前缀的日志便于排查
3. **版本兼容**: 不同版本的 Cursor 可能有不同的解析行为

---

*文档最后更新: 2026-01-27*
