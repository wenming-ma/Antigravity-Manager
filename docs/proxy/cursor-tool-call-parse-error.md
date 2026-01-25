# Proxy: Cursor tool-call JSON parse error

## Problem
After configuring Cursor to use the proxy, tool execution failed with:
`Unexpected non-whitespace character after JSON at position 76 (line 1 column 77)`.
Proxy logs showed valid JSON arguments for tool calls (e.g. Read), yet Cursor
still failed to parse the tool arguments.

## Root cause
In the OpenAI streaming path, every tool call delta was emitted with
`tool_calls[].index = 0`. When the model produced multiple tool calls in the
same response, Cursor merged deltas by index and concatenated argument payloads.
That produced invalid JSON like:
`{"path":"..."}{"path":"..."}` which triggers the parse error.

## Fix
Assign a stable, unique index per tool call in the OpenAI streaming tool-call
delta emission. This prevents Cursor from coalescing multiple tool calls into
one arguments string.

Updated file:
- `src-tauri/src/proxy/mappers/openai/streaming.rs`

## Validation
After the change (January 25, 2026), Cursor tool calls executed normally and
the parse error no longer occurred.
