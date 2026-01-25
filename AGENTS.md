# AGENTS

- This project is adding a new fucntion by configuring the OpenAI Base URL in Cursor (recommended example: `https://<host>/v1`).
- Ensure compatibility with Cursor's OpenAI Chat Completions / Responses calls and tool/function calling.
- When troubleshooting, prioritize checking agent logs and request parsing errors; add observability logs if necessary.
- Logging methods and locations:
  - Backend file logs (tracing) are written to `~/.antigravity_tools/logs/`, with daily rolling filenames (`app.log.YYYY-MM-DD`).
  - Proxy request logs are stored in SQLite: `~/.antigravity_tools/proxy_logs.db` (for querying requests/responses and errors).
  - Logging system entry point: `src-tauri/src/modules/logger.rs`.
  - Proxy request logging toggle: `~/.antigravity_tools/gui_config.json` → `proxy.enable_logging`.