# AGENTS

## Repository
- **Upstream**: https://github.com/lbjlaq/Antigravity-Manager.git
- **Fork**: https://github.com/wenming-ma/Antigravity-Manager.git
- **Sync**: `git fetch upstream && git merge upstream/main`

## Project
- Configuring OpenAI Base URL in Cursor: `https://<host>/v1`
- Ensure compatibility with Cursor's OpenAI Chat Completions / Responses calls and tool/function calling.

## Logging
- File logs: `~/.antigravity_tools/logs/app.log.YYYY-MM-DD`
- Proxy logs DB: `~/.antigravity_tools/proxy_logs.db`
- Entry point: `src-tauri/src/modules/logger.rs`
- Toggle: `~/.antigravity_tools/gui_config.json` → `proxy.enable_logging`