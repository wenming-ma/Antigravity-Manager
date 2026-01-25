# AGENTS

- This project is adding a new fucntion by configuring the OpenAI Base URL in Cursor (recommended example: `https://<host>/v1`).
- Ensure compatibility with Cursor's OpenAI Chat Completions / Responses calls and tool/function calling.
- When troubleshooting, prioritize checking agent logs and request parsing errors; add observability logs if necessary.
- Logging methods and locations:
  - Backend file logs: `~/.antigravity_tools/logs/app.log.YYYY-MM-DD`
  - Proxy request logs (SQLite): `~/.antigravity_tools/proxy_logs.db`
  - Windows path: `%USERPROFILE%\.antigravity_tools\logs\`
  - Logging system entry point: `src-tauri/src/modules/logger.rs`