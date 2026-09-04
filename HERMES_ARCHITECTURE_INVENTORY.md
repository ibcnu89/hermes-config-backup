# Hermes Architecture Inventory

Generated: 2026-09-04
Hermes Version: v0.20.6 (2026.8.27) · upstream 63279301
Install Method: git
Install Directory: /home/ibcnu/.hermes/hermes-agent
Python Version: 3.11.15

---

## Runtime

- **Hermes Version**: v0.20.6 (2026.8.27) · upstream 63279301
- **Python Version**: 3.11.15
- **Installation Location**: /home/ibcnu/.hermes/hermes-agent
- **Install Method**: git (cloned from upstream)
- **Entry Point**: `/home/ibcnu/.local/bin/hermes` → `/home/ibcnu/.hermes/hermes-agent/hermes` (shell wrapper) → `cli.py` → `hermes_cli/main.py`
- **Startup Mechanism**: Shell wrapper `hermes` sets up environment, activates venv, runs `python -m hermes_cli.main`

---

## Agent

### Main Agent Loop
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/conversation_loop.py` (495KB)
- **Function**: `conversation_loop()` - main event loop handling user messages, tool calls, model responses

### Context Construction
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/prompt_builder.py` (119KB)
- **Key Class**: `PromptBuilder` - constructs system prompt, injects tools, skills, memories

### Conversation History
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/conversation_compression.py` (272KB)
- **History Storage**: In-memory message list, persisted via `hermes_state.py`

### Context Compression/Summarization
- **Primary Implementation**: `/home/ibcnu/.hermes/hermes-agent/agent/conversation_compression.py`
  - `ConversationCompressor` class - handles compression logic
  - `compress_conversation()` - main entry point
  - Uses summarization model to compress older messages
- **Secondary**: `/home/ibcnu/.hermes/hermes-agent/agent/context_compressor.py` (431KB)
  - `ContextCompressor` class - alternative compression implementation
  - `compress_context()` - context window aware compression
- **Trajectory Compressor**: `/home/ibcnu/.hermes/hermes-agent/trajectory_compressor.py` (700KB)
  - `TrajectoryCompressor` - compresses agent trajectories for learning

### Compression Trigger
- **Location**: `conversation_loop.py` - checked after each model response
- **Trigger Condition**: When estimated token count exceeds `compression.threshold * context_window`
- **Config**: `config.yaml` → `compression.threshold` (default 0.85) and `compression.summary_model`

### Token Counting
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/token_counter.py` (not found as separate file)
- **Implementation**: Uses `tiktoken` via `agent/token_utils.py` or provider-specific tokenizers
- **Model**: Configured via `compression.summary_model` in config.yaml

### Model/Provider Client
- **Multi-Provider Architecture**: `/home/ibcnu/.hermes/hermes-agent/agent/` has multiple adapters:
  - `anthropic_adapter.py` - Anthropic API
  - `bedrock_adapter.py` - AWS Bedrock
  - `gemini_native_adapter.py` - Google Gemini
  - `vertex_adapter.py` - Google Vertex AI
  - `codex_responses_adapter.py` - OpenAI Codex
  - `relay_llm.py` / `relay_runtime.py` - Relay provider
  - `auxiliary_client.py` - Auxiliary/accounting client
- **Model Selection**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/models.py` (295KB) and `model_setup_flows.py`

---

## Tools

### Tool Registry
- **Location**: `/home/ibcnu/.hermes/hermes-agent/toolsets.py` (70KB) and `tools_config.py` (269KB)
- **Registration**: Tools registered via `ToolSet` classes, loaded from `tools/`, `skills/`, `plugins/`

### Tool Dispatcher/Executor
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/tool_executor.py` (134KB)
- **Key Class**: `ToolExecutor` - handles tool invocation, approval, retries
- **Dispatch Helpers**: `/home/ibcnu/.hermes/hermes-agent/agent/tool_dispatch_helpers.py` (33KB)

### Tool Execution Path
1. Model returns tool call
2. `tool_executor.py` → `execute_tool()` 
3. Validates against registered tool schemas
4. Executes via appropriate backend (local, docker, modal, ssh, etc.)
5. Returns result to conversation loop

### Approximate Number of Registered Tools
- Core tools: ~50+ (file, terminal, browser, web, memory, skills, etc.)
- Skills tools: Variable (each skill can register tools)
- Plugins tools: Variable
- MCP tools: Dynamically discovered

### Tool Schema Supply
- Schemas defined in each tool's `schema.py` or inline
- Injected into system prompt via `PromptBuilder`
- Also available via `tools_config.py` for CLI inspection

---

## Skills

### Skill Discovery/Loading
- **Location**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/skills_hub.py` (85KB)
- **Mechanism**: Scans `~/.hermes/skills/`, `~/.hermes/plugins/*/skills/`, built-in `skills/`
- **Loading**: Lazy-loaded on demand, not all at startup

### Approximate Number of Skills
- Built-in skills: ~50+ (in `skills/` directory)
- User skills: Variable (in `~/.hermes/skills/`)
- Plugin skills: Variable

### Loading Strategy
- **Not globally loaded** - skills loaded dynamically when referenced
- Skill manifest (`SKILL.md`) parsed for metadata
- Tools from skills registered on-demand

---

## Plugins/MCP

### Discovery Mechanism
- **MCP Catalog**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/mcp_catalog.py` (39KB)
- **MCP Config**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/mcp_config.py` (46KB)
- **MCP Startup**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/mcp_startup.py` (11KB)

### Registration Mechanism
- MCP servers configured in `config.yaml` under `mcp.servers`
- Started on demand or at startup
- Tools discovered via MCP protocol, registered in tool registry

### Schema Injection
- MCP tool schemas fetched at startup/connection
- Injected into model context like core tools
- Can be filtered via toolset configuration

---

## Persistence

### Conversation Storage
- **Primary**: `/home/ibcnu/.hermes/hermes-agent/hermes_state.py` (698KB)
  - `HermesState` class - manages conversation persistence
  - SQLite backend (`state.db`)
- **Sessions**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/sessions_cmd.py` (65KB)

### Memory
- **Memory Manager**: `/home/ibcnu/.hermes/hermes-agent/agent/memory_manager.py` (56KB)
- **Memory Provider**: `/home/ibcnu/.hermes/hermes-agent/agent/memory_provider.py` (17KB)
- **Storage**: SQLite + vector embeddings (optional)

### Databases
- `state.db` - conversations, sessions, preferences
- `kanban.db` - task management (508KB module)
- `projects.db` - project tracking
- `verification_evidence.db` - verification tracking

### State Management
- **Config**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/config.py` (261KB)
- **Config Migrations**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/config_migrations.py` (41KB)

---

## Subprocesses

### Agent Spawning
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/subagent_lifecycle.py` (20KB)
- **Mechanism**: Spawns isolated subprocesses for delegation

### Shell Execution
- **Location**: `/home/ibcnu/.hermes/hermes-agent/agent/terminal_env_*.py` and `tools/environments/`
- **Backends**: local, docker, modal, singularity, ssh, vercel_sandbox

### Background Tasks
- **Cron**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/cron.py` (35KB)
- **Automation**: `/home/ibcnu/.hermes/hermes-agent/automation/` (separate module)

### Kanban/Task Automation
- **Kanban Engine**: `/home/ibcnu/.hermes/hermes-agent/hermes_cli/kanban.py` (139KB) and `kanban_db.py` (508KB)
- **Orchestrator**: Runs background workers for task execution

---

## Context Compression - Detailed Analysis

### Files/Functions Responsible

| Function | File | Class/Function |
|----------|------|----------------|
| Determine context size | `conversation_compression.py` | `ConversationCompressor.estimate_tokens()` |
| Count tokens | `conversation_compression.py` / `token_utils.py` | `count_tokens()` / `tiktoken` |
| Decide compression needed | `conversation_loop.py` | `should_compress()` check after each turn |
| Initiate compression | `conversation_loop.py` | `compress_if_needed()` → `ConversationCompressor.compress()` |
| Generate summary | `conversation_compression.py` | `summarize_messages()` using `compression.summary_model` |
| Replace messages | `conversation_compression.py` | `compress_conversation()` replaces old messages with summary |
| Handle failures | `conversation_compression.py` | Try/except with fallback to truncation |
| Handle oversized | `conversation_compression.py` | Progressive compression until under threshold |
| Calculate context window | `model_metadata.py` | `get_context_window(model)` |
| Configured threshold | `config.py` / `config.yaml` | `compression.threshold` (default 0.85) |

### Compression Trigger Mechanism

From static inspection of `conversation_loop.py` and `conversation_compression.py`:

1. **Threshold-based**: Triggers when `estimated_tokens > threshold * context_window`
2. **Token-based**: Uses `tiktoken` or provider tokenizer for accurate counting
3. **Automatically triggered**: Checked after each model response in the conversation loop
4. **Model-context based**: Context window determined per-model via `model_metadata.py`

### Potential Issues Identified (Static Analysis)

1. **Dual Compression Systems**: Both `ConversationCompressor` (in `conversation_compression.py`) and `ContextCompressor` (in `context_compressor.py`) exist - unclear which is active
2. **Config Reference**: `compression.summary_model` in config.yaml - if unset or invalid, compression may fail silently
3. **Token Estimation**: May use rough estimation vs actual tokenization, causing premature or missed triggers
4. **Prompt Cache Boundary**: Compression must preserve prompt cache - `prompt_cache_boundary.py` suggests special handling
4. **Threshold Config**: Default 0.85 means compression at 85% of context window - may be too high for some models

### Recent Changes to Check (Git History)
- `conversation_compression.py` - core compression logic
- `context_compressor.py` - alternative implementation
- `conversation_loop.py` - trigger point
- `config.py` / `config.yaml` - threshold and model config
- `model_metadata.py` - context window definitions

---

## Customization

### Approximate Counts
- **Core Tools**: ~60+ (file ops, terminal, browser, web search, memory, skills, git, github, etc.)
- **Built-in Skills**: ~50+ (in `skills/` - autonomous-ai-agents, creative, devops, email, media, productivity, research, software-development, web, etc.)
- **Optional Skills**: ~20+ (in `optional-skills/`)
- **Plugins**: ~20+ (in `plugins/`)
- **MCP Integrations**: ~15+ (in `optional-mcps/`)

### Major Custom Additions
1. **Kanban System** - Full task management with DB, orchestrator, workers
2. **Multi-Provider LLM** - 10+ provider adapters
3. **Browser Automation** - Playwright + Browserbase + local Chrome
4. **Terminal Environments** - 7 backends (local, docker, modal, ssh, etc.)
5. **Memory/Learning System** - Persistent memory with curator backup
6. **Plugin System** - Dynamic plugin loading with capabilities
7. **Desktop App** - Electron/TUI frontend
8. **Messaging Gateway** - 10+ platforms (Telegram, Discord, Slack, etc.)
9. **Cron/Automation** - Scheduled jobs with persistence
10. **Skill Hub** - GitHub-integrated skill discovery/install

---

## Security

### Excluded from Repository
- `.env` - All API keys, tokens, passwords
- `auth.json` - Authentication credentials
- `google_client_secret.json`, `google_token.json` - OAuth
- `mcp-tokens/` - MCP server tokens
- `venv/`, `node_modules/` - Dependencies
- `cache/`, `logs/`, `sessions/` - Runtime data
- `state.db*`, `kanban.db*`, `projects.db*`, `verification_evidence.db*`, `response_store.db*` - Runtime databases
- `models_dev_cache.json`, `provider_models_cache.json` - Model caches
- `.playwright-mcp/`, browser profiles
- `teacher-leads/`, `sales-leads/` - Personal datasets
- `terminal-sessions/`, `sandboxes/` - Session data
- `__pycache__/`, `*.pyc` - Compiled artifacts
- Lock files, PID files, socket files

### Secret Scan Result
- **Staged repository passed secret scan**
- Only environment variable references (`${VAR_NAME}`) found in `config.yaml`
- No actual secret values detected in staged files

---

## Git History (Hermes Source)

### Current Commit
- Upstream: `63279301` (from `hermes --version`)
- Local modifications: Unknown (not checked to avoid modifying runtime)

### Key Files to Inspect History
```bash
cd /home/ibcnu/.hermes/hermes-agent
git log --oneline -20 -- agent/conversation_compression.py
git log --oneline -20 -- agent/context_compressor.py
git log --oneline -20 -- agent/conversation_loop.py
git log --oneline -20 -- hermes_cli/config.py
git log --oneline -20 -- agent/model_metadata.py
git log --oneline -20 -- trajectory_compressor.py
```

---

## Repository Commit

**Commit Message**: "Add sanitized Hermes source and architecture inventory"
**Files Added**: 
- `config.yaml` (sanitized, from previous commit)
- `hermes-agent/` (28MB, ~500+ Python source files)
- `HERMES_ARCHITECTURE_INVENTORY.md` (this document)
- `.gitignore` (comprehensive exclusion rules)

**Total Source Files**: ~500+ Python files in `hermes-agent/`
**Major Directories**: `agent/`, `hermes_cli/`, `tools/`, `skills/`, `providers/`, `acp_adapter/`, `cron/`

---

## Summary

This inventory documents the Hermes v0.20.6 installation at `/home/ibcnu/.hermes/hermes-agent`. The architecture follows a modular design with:

- **Narrow core** (agent loop, context management, tool execution)
- **Wide edges** (10+ LLM providers, 7 terminal backends, 50+ skills, 20+ plugins, 10+ messaging platforms)

**Context Compression** appears to be implemented in `agent/conversation_compression.py` (ConversationCompressor) with a secondary implementation in `agent/context_compressor.py` (ContextCompressor). The trigger is threshold-based (85% of model context window) and automatic in the conversation loop. Potential issues include dual implementations, config-dependent summary model, and prompt cache boundary preservation requirements.

The repository `ibcnu89/hermes-config-backup` now contains a sanitized snapshot suitable for external architectural review.
