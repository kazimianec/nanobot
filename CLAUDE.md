# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install for development (editable, with dev dependencies)
pip install -e ".[dev]"

# Run all tests
pytest

# Run a single test file
pytest tests/test_commands.py

# Run a single test by name
pytest tests/test_commands.py::test_function_name -v

# Lint
ruff check .

# Format
ruff format .

# Run the CLI
nanobot agent          # interactive chat
nanobot gateway        # start multi-channel gateway
nanobot status         # show config/provider status
```

Tests use `pytest-asyncio` with `asyncio_mode = "auto"` — no manual `@pytest.mark.asyncio` needed.

Linting: line-length=100, rules E/F/I/N/W, E501 ignored. Target Python 3.11+.

## Architecture

nanobot is an async AI assistant framework. The core flow is:

**Channel → MessageBus → AgentLoop → Tools → MessageBus → Channel**

### Message Bus (`nanobot/bus/`)

The `MessageBus` is a pair of asyncio queues decoupling channels from the agent. Channels push `InboundMessage` objects; the agent reads them, executes, and pushes `OutboundMessage` objects back. The channel manager then routes outbound messages to the correct platform.

### Agent Loop (`nanobot/agent/loop.py`)

`AgentLoop` is the core orchestrator:
1. Reads `InboundMessage` from the bus
2. Loads session history and builds context via `ContextBuilder`
3. Calls the LLM provider in a loop (max 40 iterations by default)
4. Executes tool calls returned by the LLM
5. Persists the turn to `SessionManager`
6. Pushes `OutboundMessage` back to the bus

Built-in slash commands: `/new` (clear session with memory archival), `/stop` (cancel active tasks), `/help`.

### Context & Memory (`nanobot/agent/context.py`, `memory.py`)

`ContextBuilder` assembles the system prompt from:
- Bootstrap files in workspace: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md`
- Long-term memory from `memory/MEMORY.md` and `memory/HISTORY.md`
- Bundled and user-installed skills

`MemoryStore` implements two-layer persistence: `MEMORY.md` (facts) + `HISTORY.md` (timestamped log). Consolidation runs asynchronously when unconsolidated messages reach `memory_window` (default 100), using an LLM call with a `save_memory` tool.

### Channels (`nanobot/channels/`)

Each channel extends `BaseChannel` and implements `start()`, `stop()`, and `send()`. `BaseChannel._handle_message()` checks the `allowFrom` list and publishes to the inbound bus. **Empty `allowFrom` denies all access; use `["*"]` to allow everyone.**

Session keys are `"{channel}:{chat_id}"` by default, overridable per-message for thread-scoped sessions (used by Slack/Discord threads).

### Providers (`nanobot/providers/`)

`PROVIDERS` tuple in `registry.py` is the single source of truth for all LLM providers. Provider selection order: explicit `provider` config → model-name keyword match → API key prefix → API base URL → first available key.

**To add a new provider (2 steps):**
1. Add a `ProviderSpec` to `PROVIDERS` in `nanobot/providers/registry.py`
2. Add a field to `ProvidersConfig` in `nanobot/config/schema.py`

Most providers route through LiteLLM (`litellm_provider.py`). The `custom` provider and OAuth-based providers (`openai_codex`, `github_copilot`) have dedicated implementations.

### Tools (`nanobot/agent/tools/`)

Each tool extends the `Tool` ABC with `name`, `description`, `parameters` (JSON Schema), and `execute(**kwargs)`. Register tools via `ToolRegistry`. MCP tools are connected lazily on first use.

Built-in tools: `read_file`, `write_file`, `edit_file`, `list_dir`, `exec`, `web_search`, `web_fetch`, `message`, `spawn`, `cron`.

### Configuration (`nanobot/config/`)

Root config object is `Config` (Pydantic `BaseSettings`). All config models extend `Base`, which uses `to_camel` alias generator — so `~/.nanobot/config.json` accepts both camelCase and snake_case. New config models must also extend `Base` to maintain this convention. Also loadable via env vars with `NANOBOT_` prefix and `__` delimiter (e.g. `NANOBOT_AGENTS__DEFAULTS__MODEL`).

### Sessions (`nanobot/session/manager.py`)

`Session` holds a list of message dicts plus metadata. `SessionManager` serializes sessions to JSON files in the workspace. Tool results are truncated to 500 chars in storage to keep context size manageable.

### Subagents & Cron (`nanobot/agent/subagent.py`, `nanobot/cron/`)

`SubagentManager` spawns background `AgentLoop` instances for the `spawn` tool. The cron service runs scheduled jobs defined via `nanobot cron add`. The heartbeat service wakes every 30 minutes and processes tasks in `HEARTBEAT.md`.

### Skills (`nanobot/skills/`)

Each skill is a directory containing a `SKILL.md` with YAML frontmatter (name, description, metadata) and Markdown instructions injected into the agent's context. Skills follow OpenClaw conventions for compatibility. User-installed skills go in `~/.nanobot/skills/`.

### Bridge (`bridge/`)

A separate Node.js (TypeScript) process that provides a WebSocket bridge for WhatsApp (via the `whatsapp-web.js` library). nanobot connects to it via `ws://localhost:3001`.

## Key Conventions

- **Python 3.11+** required. Type hints use `X | Y` union syntax, not `Optional`.
- **Async everywhere**: the core is fully async. Channels, agent loop, tools all use `async`/`await`.
- **Config aliases**: all Pydantic config models use `to_camel` alias generator. Config JSON is camelCase; Python code is snake_case.
- **allowFrom semantics**: empty `allowFrom` list **denies all access**. Use `["*"]` to allow everyone. This is a common gotcha.
- **Tool result truncation**: session storage truncates tool results to 500 chars. Keep this in mind when debugging context issues.
- **CLI via Typer**: entry point is `nanobot/cli/commands.py` → `app` (Typer instance), exposed as `nanobot` console script.
