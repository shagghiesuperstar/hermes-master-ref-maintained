# Hermes Agent — Master Architecture Reference
## `Hermes_Architecture.md`
### Version sourced from: NousResearch/hermes-agent (GitHub) + https://hermes-agent.nousresearch.com/docs/
### Compiled: 2026-06-14 | SSOT for offline agentic development

---

# TABLE OF CONTENTS

1. [System Overview & Design Principles](#1-system-overview--design-principles)
2. [Directory Structure](#2-directory-structure)
3. [Configuration — config.yaml SSOT](#3-configuration--configyaml-ssot)
4. [Environment Variables & .env](#4-environment-variables--env)
5. [HERMES_HOME & Profile Isolation](#5-hermes_home--profile-isolation)
6. [Agent Identity — SOUL.md](#6-agent-identity--soulmd)
7. [Prompt Assembly — System Prompt Architecture](#7-prompt-assembly--system-prompt-architecture)
8. [Agent Loop Internals (AIAgent)](#8-agent-loop-internals-aiagent)
9. [Tool System — Registry, Toolsets, Dispatch](#9-tool-system--registry-toolsets-dispatch)
10. [Built-in Tools Reference (All ~71 Tools)](#10-built-in-tools-reference-all-71-tools)
11. [Toolsets Reference (All Platform & Core Toolsets)](#11-toolsets-reference-all-platform--core-toolsets)
12. [Skills System — SKILL.md Format & Full Spec](#12-skills-system--skillmd-format--full-spec)
13. [Memory System — MEMORY.md, USER.md, Providers](#13-memory-system--memorymd-usermd-providers)
14. [Context Compression & Prompt Caching](#14-context-compression--prompt-caching)
15. [Session Storage — SQLite Schema](#15-session-storage--sqlite-schema)
16. [MCP (Model Context Protocol) Integration](#16-mcp-model-context-protocol-integration)
17. [Provider Runtime Resolution](#17-provider-runtime-resolution)
18. [Gateway Internals (Messaging Platforms)](#18-gateway-internals-messaging-platforms)
19. [Cron Internals](#19-cron-internals)
20. [Plugin System](#20-plugin-system)
21. [Memory Provider Plugin Development](#21-memory-provider-plugin-development)
22. [Adding Custom Tools](#22-adding-custom-tools)
23. [Programmatic Integration (ACP / TUI / API Server)](#23-programmatic-integration-acp--tui--api-server)
24. [Slash Commands Reference](#24-slash-commands-reference)
25. [Security Model](#25-security-model)
26. [File Dependency Chain & Import Order](#26-file-dependency-chain--import-order)
27. [Key Constants & Defaults Reference](#27-key-constants--defaults-reference)

---

# 1. System Overview & Design Principles

## What Hermes Agent Is

Hermes Agent is a production-grade, local AI assistant and agent framework built by Nous Research. It orchestrates LLM-driven conversations with tool use, persistent memory, scheduling, multi-platform messaging, multi-agent delegation, and a plugin ecosystem — all controlled from a single config file and HERMES_HOME directory.

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Entry Points                                  │
│                                                                      │
│  CLI (cli.py)    Gateway (gateway/run.py)    ACP (acp_adapter/)     │
│  Batch Runner    API Server                  Python Library          │
└──────────┬──────────────┬───────────────────────┬───────────────────┘
           │              │                       │
           ▼              ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     AIAgent (run_agent.py)                          │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Prompt       │  │ Provider     │  │ Tool         │               │
│  │ Builder      │  │ Resolution   │  │ Dispatch     │               │
│  │ (prompt_     │  │ (runtime_    │  │ (model_      │               │
│  │  builder.py) │  │  provider.py)│  │  tools.py)   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                       │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐               │
│  │ Compression  │  │ 3 API Modes  │  │ Tool Registry│               │
│  │ & Caching    │  │ chat_compl.  │  │ (registry.py)│               │
│  │              │  │ codex_resp.  │  │ ~71 tools    │               │
│  │              │  │ anthropic    │  │ ~28 toolsets │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└─────────┴─────────────────┴─────────────────┴───────────────────────┘
           │                                    │
           ▼                                    ▼
┌───────────────────┐              ┌──────────────────────┐
│ Session Storage   │              │ Tool Backends         │
│ (SQLite + FTS5)   │              │ Terminal (6 backends) │
│ hermes_state.py   │              │ Browser (5 backends)  │
│ gateway/session.py│              │ Web (4 backends)      │
└───────────────────┘              │ MCP (dynamic)         │
                                   │ File, Vision, etc.    │
                                   └──────────────────────┘
```

## Design Principles

| Principle | Practical Meaning |
|-----------|------------------|
| **Prompt stability** | System prompt doesn't change mid-conversation. No cache-breaking mutations except explicit user actions (`/model`). |
| **Observable execution** | Every tool call is visible via callbacks. Progress updates in CLI (spinner) and gateway (chat messages). |
| **Interruptible** | API calls and tool execution can be cancelled mid-flight by user input or signals. |
| **Platform-agnostic core** | One `AIAgent` class serves CLI, gateway, ACP, batch, and API server. Platform differences live in the entry point. |
| **Loose coupling** | Optional subsystems (MCP, plugins, memory providers, RL environments) use registry patterns and `check_fn` gating. |
| **Profile isolation** | Each profile (`hermes -p <name>`) gets its own `HERMES_HOME`, config, memory, sessions, gateway PID. Multiple profiles run concurrently. |

---

# 2. Directory Structure

```
hermes-agent/
├── run_agent.py              # AIAgent — core conversation loop (LARGE)
├── cli.py                    # HermesCLI — interactive terminal UI (LARGE)
├── model_tools.py            # Tool discovery, schema collection, dispatch
├── toolsets.py               # Tool groupings and platform presets
├── hermes_state.py           # SQLite session/state database with FTS5
├── hermes_constants.py       # HERMES_HOME, profile-aware paths
├── batch_runner.py           # Batch trajectory generation
│
├── agent/                    # Agent internals
│   ├── prompt_builder.py     # System prompt assembly (CRITICAL)
│   ├── context_engine.py     # ContextEngine ABC (pluggable)
│   ├── context_compressor.py # Default engine — lossy summarization
│   ├── prompt_caching.py     # Anthropic prompt caching
│   ├── auxiliary_client.py   # Auxiliary LLM for side tasks
│   ├── model_metadata.py     # Model context lengths, token estimation
│   ├── models_dev.py         # models.dev registry integration
│   ├── anthropic_adapter.py  # Anthropic Messages API format conversion
│   ├── display.py            # KawaiiSpinner, tool preview formatting
│   ├── skill_commands.py     # Skill slash commands
│   ├── memory_manager.py     # Memory manager orchestration
│   ├── memory_provider.py    # Memory provider ABC
│   └── trajectory.py         # Trajectory saving helpers
│
├── hermes_cli/               # CLI subcommands and setup
│   ├── main.py               # Entry point — all `hermes` subcommands (LARGE)
│   ├── config.py             # DEFAULT_CONFIG, OPTIONAL_ENV_VARS, migration
│   ├── commands.py           # COMMAND_REGISTRY — central slash command defs
│   ├── auth.py               # PROVIDER_REGISTRY, credential resolution
│   ├── runtime_provider.py   # Provider → api_mode + credentials
│   ├── models.py             # Model catalog, provider model lists
│   ├── model_switch.py       # /model command logic (CLI + gateway shared)
│   ├── setup.py              # Interactive setup wizard (LARGE)
│   ├── skin_engine.py        # CLI theming engine
│   ├── skills_config.py      # hermes skills — enable/disable per platform
│   ├── skills_hub.py         # /skills slash command
│   ├── tools_config.py       # hermes tools — enable/disable per platform
│   ├── plugins.py            # PluginManager — discovery, loading, hooks
│   ├── callbacks.py          # Terminal callbacks (clarify, sudo, approval)
│   └── gateway.py            # hermes gateway start/stop
│
├── tools/                    # Tool implementations (one file per tool)
│   ├── registry.py           # Central tool registry (CRITICAL)
│   ├── approval.py           # Dangerous command detection
│   ├── terminal_tool.py      # Terminal orchestration
│   ├── process_registry.py   # Background process management
│   ├── file_tools.py         # read_file, write_file, patch, search_files
│   ├── web_tools.py          # web_search, web_extract
│   ├── browser_tool.py       # 10 browser automation tools
│   ├── code_execution_tool.py# execute_code sandbox
│   ├── delegate_tool.py      # Subagent delegation
│   ├── mcp_tool.py           # MCP client (LARGE)
│   ├── credential_files.py   # File-based credential passthrough
│   ├── env_passthrough.py    # Env var passthrough for sandboxes
│   ├── ansi_strip.py         # ANSI escape stripping
│   └── environments/         # Terminal backends
│       ├── local.py
│       ├── docker.py
│       ├── ssh.py
│       ├── modal.py
│       ├── daytona.py
│       └── singularity.py
│
├── gateway/                  # Messaging platform gateway
│   ├── run.py                # GatewayRunner — message dispatch (LARGE)
│   ├── session.py            # SessionStore — conversation persistence
│   ├── delivery.py           # Outbound message delivery
│   ├── pairing.py            # DM pairing authorization
│   ├── hooks.py              # Hook discovery and lifecycle events
│   ├── mirror.py             # Cross-session message mirroring
│   ├── status.py             # Token locks, profile-scoped process tracking
│   ├── builtin_hooks/        # Always-registered hooks (none shipped; stub)
│   └── platforms/            # 20+ adapters (see Section 18)
│
├── acp_adapter/              # ACP server (VS Code / Zed / JetBrains)
├── tui_gateway/              # TUI gateway JSON-RPC server
├── cron/                     # Scheduler
│   ├── jobs.py               # Job model, storage, atomic R/W to jobs.json
│   └── scheduler.py          # Scheduler loop — due-job detection, execution
├── providers/                # Provider ABC + plugin registry
├── plugins/
│   ├── memory/               # Memory provider plugins
│   │   └── honcho/           # Honcho memory provider (reference impl)
│   ├── context_engine/       # Context engine plugins (LCM, etc.)
│   ├── model-providers/      # Per-provider bundled plugins
│   └── video_gen/            # Video generation plugins
├── skills/                   # Bundled skills (always available)
├── optional-skills/          # Official optional skills (install explicitly)
├── website/                  # Docusaurus documentation site
│   └── docs/                 # All official documentation source
└── tests/                    # Pytest suite (~25,000 tests, ~1,250 files)
```

### HERMES_HOME Directory Layout (~/.hermes/ by default)

```
~/.hermes/
├── config.yaml               # SSOT for all configuration
├── .env                      # Secrets: API keys, tokens
├── SOUL.md                   # Agent identity / persona (replaces default)
├── MEMORY.md                 # Persistent memory (agent-maintained)
├── USER.md                   # User profile (agent-maintained)
├── state.db                  # SQLite session storage + FTS5
├── state.db-wal              # SQLite WAL
├── state.db-shm              # SQLite shared memory
├── cron/
│   ├── jobs.json             # Cron job definitions
│   └── output/               # Local cron delivery output
├── skills/                   # User-created custom skills
├── plugins/                  # User-installed plugins
├── hooks/                    # User-installed gateway hooks
├── scripts/                  # User scripts for cron/skills
├── mcp-tokens/               # MCP OAuth token files
│   └── <server>.json
└── <memory-provider>/        # Memory provider data (e.g., honcho/)
```

---

# 3. Configuration — config.yaml SSOT

`~/.hermes/config.yaml` is the **single source of truth** for all non-secret configuration. All keys, defaults, and semantics are listed below.

## Complete config.yaml Structure

```yaml
# ─── Model / Provider ───────────────────────────────────────────────
model: "anthropic/claude-opus-4.6"     # Active model identifier
provider: "openrouter"                 # Active provider name
                                       # Valid providers listed in Section 17

# ─── Toolsets ───────────────────────────────────────────────────────
toolsets:
  - hermes-cli                         # Default; see Section 11

# Custom toolset bundles (project-specific)
custom_toolsets:
  data-science:
    - file
    - terminal
    - code_execution
    - web
    - vision

# ─── Memory ─────────────────────────────────────────────────────────
memory:
  provider: null                       # External provider name; null = built-in only
  # Example: provider: "honcho"

# ─── Context Engine ─────────────────────────────────────────────────
context:
  engine: "compressor"                 # "compressor" (default) | plugin name (e.g., "lcm")

# ─── Compression ────────────────────────────────────────────────────
compression:
  enabled: true
  threshold: 0.50                      # Fraction of context window (default 50%)
  target_ratio: 0.20                   # Tail protection token budget ratio
  protect_last_n: 20                   # Min messages always preserved in tail
  codex_gpt55_autoraise: true          # Raise trigger to 85% for gpt-5.5 on Codex OAuth

# ─── Prompt Caching (Anthropic) ─────────────────────────────────────
prompt_caching:
  cache_ttl: "5m"                      # "5m" | "1h"

# ─── Auxiliary Models ────────────────────────────────────────────────
auxiliary:
  compression:
    model: null                        # null = auto-detect from main model
    provider: auto                     # "auto" | "openrouter" | "nous" | "main" | etc.
    base_url: null
  vision:
    model: null
    provider: auto
    base_url: null
  web_extraction:
    model: null
    provider: auto
    base_url: null
  monitor:
    model: null                        # Used for mail classification
    provider: auto
    base_url: null

# ─── Fallback ────────────────────────────────────────────────────────
fallback_providers:
  - provider: "openrouter"
    model: "anthropic/claude-3.5-haiku"

# Legacy single fallback (still accepted, migrated on first write):
# fallback_model:
#   provider: "openrouter"
#   model: "anthropic/claude-3.5-haiku"

# ─── Terminal ────────────────────────────────────────────────────────
terminal:
  env_passthrough:                     # Extra env vars passed into sandboxes
    - MY_CUSTOM_VAR
  backend: local                       # local | docker | ssh | modal | daytona | singularity

# ─── Skills ──────────────────────────────────────────────────────────
skills:
  template_vars: true                  # Enable ${HERMES_SKILL_DIR} substitution (default: true)
  inline_shell: false                  # Enable !`cmd` snippets in SKILL.md (default: false)
  inline_shell_timeout: 10             # Seconds per inline shell snippet
  config:                              # Per-skill config values (populated by hermes config migrate)
    myplugin:
      path: ~/my-data

# ─── MCP Servers ─────────────────────────────────────────────────────
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "***"
    enabled: true
    timeout: 120
    connect_timeout: 60
    supports_parallel_tool_calls: false
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      exclude: []
      resources: false
      prompts: false

  remote_api:
    url: "https://mcp.example.com/mcp"
    headers:
      Authorization: "Bearer ***"
    auth: oauth                        # OAuth 2.1 PKCE flow
    ssl_verify: true                   # true | false | "/path/to/ca-bundle.pem"
    client_cert: "~/secrets/client.pem"  # mTLS cert
    client_key: "~/secrets/client.key"   # mTLS key (optional if combined PEM)

# ─── Plugins ─────────────────────────────────────────────────────────
plugins:
  enabled: []                          # List of enabled plugin names

# ─── Cron ────────────────────────────────────────────────────────────
cron:
  wrap_response: true                  # Wrap cron outputs in name/task header
  script_timeout_seconds: 120         # Default script execution timeout

# ─── Command Allowlist ────────────────────────────────────────────────
command_allowlist:                     # Permanently approve dangerous pattern matches
  - "rm -rf node_modules/"

# ─── Custom Providers ────────────────────────────────────────────────
custom_providers:
  - name: "my-local"
    base_url: "http://localhost:8000/v1"
    model: "my-model"
    api_key: ""                        # Optional; sent as Bearer token

# ─── Gateway / Display ───────────────────────────────────────────────
display:
  theme: "default"                     # CLI theme (see skin_engine.py)

# ─── Agent Behavior ──────────────────────────────────────────────────
agent:
  max_turns: 90                        # Default iteration budget
```

## Key Config Behaviors

- **Reading**: The CLI reads `config.yaml` via `load_cli_config()` with hardcoded defaults. The gateway reads YAML directly — keys absent from user's file may differ between CLI and gateway.
- **Writing**: `hermes config set <key> <value>` writes atomically. Use dotpath notation: `hermes config set compression.threshold 0.6`
- **Migration**: `hermes config migrate` scans enabled skills for unconfigured settings and prompts interactively.
- **Secrets**: Never put secrets in `config.yaml` — use `~/.hermes/.env` instead.

---

# 4. Environment Variables & .env

`~/.hermes/.env` stores all secrets. It is loaded at agent startup and NEVER exposed to the LLM model.

## Core Provider Keys

```bash
# OpenRouter (primary recommended route)
OPENROUTER_API_KEY=sk-or-...

# Anthropic (native)
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_TOKEN=...                # Claude Code OAuth token
CLAUDE_CODE_OAUTH_TOKEN=...        # Alternative Claude Code OAuth

# OpenAI / Codex
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://...        # Custom OpenAI-compatible endpoint

# Google / Gemini
GEMINI_API_KEY=...
GOOGLE_API_KEY=...

# xAI
XAI_API_KEY=...

# DeepSeek
DEEPSEEK_API_KEY=...

# Nous Portal
NOUS_API_KEY=...
```

## Web / Search Keys

```bash
EXA_API_KEY=...
FIRECRAWL_API_KEY=...
TAVILY_API_KEY=...
PARALLEL_API_KEY=...
SERP_API_KEY=...
```

## Gateway / Platform Keys

```bash
# Telegram
TELEGRAM_BOT_TOKEN=...
TELEGRAM_ALLOWED_USERS=user1,user2
TELEGRAM_ALLOW_ALL_USERS=false

# Discord
DISCORD_BOT_TOKEN=...
DISCORD_ALLOWED_USERS=...

# Slack
SLACK_BOT_TOKEN=...
SLACK_APP_TOKEN=...

# Home Assistant
HASS_TOKEN=...
HASS_URL=http://homeassistant.local:8123
```

## Image / Video / Media Keys

```bash
FAL_KEY=...                        # FAL.ai image/video generation
```

## Important Env Var Behaviors

| Variable | Behavior |
|----------|----------|
| `HERMES_HOME` | Override default `~/.hermes/` path |
| `HERMES_PROFILE` | Active profile name |
| `HERMES_EPHEMERAL_SYSTEM_PROMPT` | Append to system prompt for one turn only (not cached) |
| `HERMES_KANBAN_TASK` | When set, activates kanban toolset for dispatcher-spawned workers |
| `HERMES_CRON_SCRIPT_TIMEOUT` | Override cron script timeout |

## Env Var Passthrough to Sandboxes

When a skill declares `required_environment_variables`, those vars are automatically passed through to:
- `execute_code` sandbox
- `terminal` (local, Docker, Modal, SSH)

Additional vars can be added in `config.yaml`:
```yaml
terminal:
  env_passthrough:
    - MY_CUSTOM_VAR
    - ANOTHER_VAR
```

---

# 5. HERMES_HOME & Profile Isolation

```
Default HERMES_HOME:  ~/.hermes/
Profile HERMES_HOME:  ~/.hermes/profiles/<name>/
```

### Switching Profiles

```bash
hermes -p myprofile chat         # Use named profile
hermes --profile myprofile chat
HERMES_PROFILE=myprofile hermes chat
```

Each profile has its own:
- `config.yaml`
- `.env`
- `SOUL.md`, `MEMORY.md`, `USER.md`
- `state.db` (sessions)
- `cron/jobs.json`
- Gateway PID file

Multiple profiles can run simultaneously with no shared state.

### In Code

```python
from hermes_constants import get_hermes_home
hermes_home = get_hermes_home()   # Returns profile-aware Path object
```

**CRITICAL**: Always use `get_hermes_home()`. Never hardcode `~/.hermes`.

---

# 6. Agent Identity — SOUL.md

`~/.hermes/SOUL.md` is the first section of every system prompt. It defines the agent's persona, standing instructions, and behavioral rules.

## Loading Logic

```python
# From agent/prompt_builder.py (simplified)
def load_soul_md() -> Optional[str]:
    soul_path = get_hermes_home() / "SOUL.md"
    if not soul_path.exists():
        return None
    content = soul_path.read_text(encoding="utf-8").strip()
    content = _scan_context_content(content, "SOUL.md")  # Security scan
    content = _truncate_content(content, "SOUL.md")       # Cap at 20k chars
    return content
```

## Default Fallback (when SOUL.md absent)

```
You are Hermes Agent, an intelligent AI assistant created by Nous Research.
You are helpful, knowledgeable, and direct. You assist users with a wide
range of tasks including answering questions, writing and editing code,
analyzing information, creative work, and executing actions via your tools.
You communicate clearly, admit uncertainty when appropriate, and prioritize
being genuinely useful over being verbose unless otherwise directed below.
Be targeted and efficient in your exploration and investigations.
```

## SOUL.md Constraints

- Maximum: 20,000 characters (hard cap; excess is truncated with 70/20 head/tail ratio)
- Security scanned before injection (prompt injection patterns blocked)
- YAML frontmatter is stripped if present
- Format: Plain markdown, no required structure

## When SOUL.md Loads Successfully

`build_context_files_prompt(skip_soul=True)` is called to prevent SOUL.md from appearing twice (once as identity, once as a context file).

---

# 7. Prompt Assembly — System Prompt Architecture

## The Three Tiers

The system prompt is assembled from three ordered tiers, joined in sequence:

```
[stable] → [context] → [volatile]
```

### Tier 1: stable
Contains: agent identity (SOUL.md or fallback) + tool/model guidance + skills index + environment hints + platform hints

### Tier 2: context
Contains: caller-supplied `system_message` + project context files

### Tier 3: volatile
Contains: built-in memory snapshot (MEMORY.md) + user profile snapshot (USER.md) + external memory-provider block + timestamp/session/model/provider line

## Full System Prompt Layer Order

```
# Layer 1: Agent Identity (from ~/.hermes/SOUL.md)
You are Hermes, an AI assistant...

# Layer 2: Tool-aware behavior guidance
You have persistent memory across sessions...
When the user references something from a past conversation...

# Layer 3: Tool-use enforcement (GPT/Codex models only)
You MUST use your tools to take action...

# Layer 4: Optional Honcho static block (when active)
[Honcho personality/context data]

# Layer 5: Optional system message (from config or API)
[User-configured system message override]

# Layer 6: Frozen MEMORY snapshot
## Persistent Memory
- User prefers Python 3.12, uses pyproject.toml
- Default editor is nvim
...

# Layer 7: Frozen USER profile snapshot
## User Profile
- Name: Alice
- GitHub: alice-dev

# Layer 8: Skills index
## Skills (mandatory)
Before replying, scan the skills below. If one clearly matches
your task, load it with skill_view(name) and follow its instructions.
...
<available_skills>
  software-development:
    - code-review: Structured code review workflow
</available_skills>

# Layer 9: Context files (from project directory)
# Project Context
## AGENTS.md
...

# Layer 10: Timestamp + session
Current time: 2026-03-30T14:30:00-07:00
Session: abc123

# Layer 11: Platform hint
You are a CLI AI Agent...
```

## Project Context File Discovery

Only ONE project context type loads (first match wins):

| Priority | Files | Search Scope |
|----------|-------|-------------|
| 1 | `.hermes.md`, `HERMES.md` | CWD up to git root (walks up) |
| 2 | `AGENTS.md` | CWD only |
| 3 | `CLAUDE.md` | CWD only |
| 4 | `.cursorrules`, `.cursor/rules/*.mdc` | CWD only |

All context files are:
- Security scanned (prompt injection blocked)
- Truncated at 20,000 chars (70/20 head/tail ratio)
- YAML frontmatter stripped (`.hermes.md` only)

## Ephemeral Layers (NOT in cached system prompt)

These are injected at API-call time only and do NOT persist:
- `ephemeral_system_prompt` (env var `HERMES_EPHEMERAL_SYSTEM_PROMPT`)
- Prefill messages
- Gateway-derived session context overlays
- `pre_llm_call` plugin context (appended to current user message, not system prompt)
- Honcho/external recall injected mid-turn into user message

## Memory Snapshot Behavior

Memory snapshots are **frozen at session start**. Mid-session memory writes update disk but do NOT mutate the cached system prompt. The snapshot only refreshes on:
- New session start
- Explicit invalidation/rebuild (e.g., compression-triggered rebuild)

## Customization Hierarchy (in order of preference)

1. Edit `~/.hermes/SOUL.md` → change identity/persona
2. Edit `~/.hermes/MEMORY.md` / `USER.md` → persistent facts
3. Edit project context files (`.hermes.md`, `AGENTS.md`, etc.) → repo rules
4. Create/modify Skills → reusable workflows
5. Set `HERMES_EPHEMERAL_SYSTEM_PROMPT` → turn-scoped guidance
6. Edit `agent/prompt_builder.py` → fork-level global change (not recommended)

---

# 8. Agent Loop Internals (AIAgent)

Core file: `run_agent.py`

## Two Entry Points

```python
# Simple — returns final response string
response = agent.chat("Fix the bug in main.py")

# Full — returns dict with messages, metadata, usage stats
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,           # auto-built if omitted
    conversation_history=None,      # auto-loaded from session if omitted
    task_id="task_abc123"
)
```

`chat()` is a thin wrapper around `run_conversation()` extracting `result["final_response"]`.

## Turn Lifecycle (Step by Step)

```
run_conversation()
  1. Generate task_id if not provided
  2. Append user message to conversation history
  3. Build or reuse cached system prompt (prompt_builder.py)
  4. Check if preflight compression is needed (>50% context)
  5. Build API messages from conversation history
     - chat_completions: OpenAI format as-is
     - codex_responses: convert to Responses API input items
     - anthropic_messages: convert via anthropic_adapter.py
  6. Inject ephemeral prompt layers (budget warnings, context pressure)
  7. Apply prompt caching markers if on Anthropic
  8. Make interruptible API call (_interruptible_api_call)
  9. Parse response:
     - If tool_calls: execute them, append results, loop back to step 5
     - If text response: persist session, flush memory if needed, return
```

## API Modes

| API Mode | Used For | Client Type |
|----------|----------|-------------|
| `chat_completions` | OpenAI-compatible endpoints (OpenRouter, custom, most providers) | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` with Responses format |
| `anthropic_messages` | Native Anthropic Messages API | `anthropic.Anthropic` via adapter |

**Mode resolution order:**
1. Explicit `api_mode` constructor arg (highest priority)
2. Provider-specific detection (e.g., `anthropic` provider → `anthropic_messages`)
3. Base URL heuristics (e.g., `api.anthropic.com` → `anthropic_messages`)
4. Default: `chat_completions`

## Message Format (Internal)

All messages use OpenAI-compatible format internally:

```python
{"role": "system", "content": "..."}
{"role": "user", "content": "..."}
{"role": "assistant", "content": "...", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}
```

Reasoning content stored in `assistant_msg["reasoning"]`.

## Message Alternation Rules

- After system message: `User → Assistant → User → Assistant → ...`
- During tool calling: `Assistant (with tool_calls) → Tool → Tool → ... → Assistant`
- **NEVER** two assistant messages in a row
- **NEVER** two user messages in a row
- **ONLY** `tool` role can have consecutive entries (parallel tool results)

## Interruptible API Calls

```
┌────────────────────────────────────────────────────┐
│  Main thread                  API thread           │
│                                                    │
│   wait on:                     HTTP POST           │
│    - response ready     ───▶   to provider         │
│    - interrupt event                               │
│    - timeout                                       │
└────────────────────────────────────────────────────┘
```

When interrupted:
- API thread abandoned (response discarded)
- No partial response injected into history
- Agent can process new input or shut down cleanly

## Tool Execution

### Sequential vs Concurrent

- **Single tool call** → executed directly in main thread
- **Multiple tool calls** → executed concurrently via `ThreadPoolExecutor`
  - Interactive tools (e.g., `clarify`) force sequential execution
  - Results reinserted in original call order regardless of completion order

### Tool Execution Flow

```
for each tool_call in response.tool_calls:
    1. Resolve handler from tools/registry.py
    2. Fire pre_tool_call plugin hook
    3. Check if dangerous command (tools/approval.py)
       - If dangerous: invoke approval_callback, wait for user
    4. Execute handler with args + task_id
    5. Fire post_tool_call plugin hook
    6. Append {"role": "tool", "content": result} to history
```

### Agent-Level Tools (Intercepted Before Registry)

| Tool | Why Intercepted |
|------|----------------|
| `todo` | Reads/writes agent-local task state |
| `memory` | Writes to persistent memory files with character limits |
| `session_search` | Queries session history via agent's session DB |
| `delegate_task` | Spawns subagent(s) with isolated context |

## Callbacks

| Callback | When Fired | Used By |
|----------|-----------|---------|
| `tool_progress_callback` | Before/after each tool execution | CLI spinner, gateway progress |
| `thinking_callback` | When model starts/stops thinking | CLI "thinking..." indicator |
| `reasoning_callback` | When model returns reasoning content | CLI display, gateway blocks |
| `clarify_callback` | When `clarify` tool is called | CLI input prompt, gateway interactive |
| `step_callback` | After each complete agent turn | Gateway step tracking, ACP progress |
| `stream_delta_callback` | Each streaming token | CLI streaming display |
| `tool_gen_callback` | When tool call is parsed from stream | CLI tool preview in spinner |
| `status_callback` | State changes | ACP status updates |

## Iteration Budget

- Default: 90 iterations (`agent.max_turns`)
- Each agent gets its own budget
- Subagents: independent budgets capped at `delegation.max_iterations` (default 50)
- At 100%: agent stops and returns summary of work done

## Fallback Model Behavior

When primary model fails (429, 5xx, 401/403):
1. Check `fallback_providers` list in config
2. Try each fallback in order
3. On success, continue conversation with new provider
4. On 401/403, attempt credential refresh before failing over

Fallback is single-shot (`_fallback_activated = True` after first activation).

---

# 9. Tool System — Registry, Toolsets, Dispatch

## Tool Registration

Every tool file in `tools/` self-registers at import time:

```python
from tools.registry import registry

registry.register(
    name="terminal",               # Unique tool name (used in API schemas)
    toolset="terminal",            # Toolset this tool belongs to
    schema={...},                  # OpenAI function-calling schema
    handler=handle_terminal,       # Function called when tool executes
    check_fn=check_terminal,       # Optional: returns True/False for availability
    requires_env=["SOME_VAR"],     # Optional: env vars needed (for UI display)
    is_async=False,                # Whether handler is an async coroutine
    description="Run commands",    # Human-readable description
    emoji="💻",                    # Emoji for spinner/progress display
)
```

Each call creates a `ToolEntry` in singleton `ToolRegistry._tools` dict keyed by tool name.

## Auto-Discovery

`discover_builtin_tools()` scans `tools/*.py` via AST parsing for top-level `registry.register()` calls, then imports them. **No manual import list required.** New tool files are auto-discovered.

Exclusions from auto-discovery: `__init__.py`, `registry.py`, `mcp_tool.py`

After core discovery:
1. **MCP tools** — `tools.mcp_tool.discover_mcp_tools()`
2. **Plugin tools** — `hermes_cli.plugins.discover_plugins()`

## check_fn Behavior

```python
# Simplified from registry.py
if entry.check_fn:
    try:
        available = bool(entry.check_fn())
    except Exception:
        available = False   # Exceptions = unavailable
    if not available:
        continue            # Skip this tool entirely
```

- Results cached per-call
- Exceptions treated as "unavailable" (fail-safe)

## Dispatch Flow

```
Model response with tool_call
    ↓
run_agent.py agent loop
    ↓
model_tools.handle_function_call(name, args, task_id, user_task)
    ↓
[Agent-loop tools?] → handled directly (todo, memory, session_search, delegate_task)
    ↓
[Plugin pre-hook] → invoke_hook("pre_tool_call", ...)
    ↓
registry.dispatch(name, args, **kwargs)
    ↓
Look up ToolEntry by name
    ↓
[Async handler?] → bridge via _run_async()
[Sync handler?]  → call directly
    ↓
Return result string (or JSON error)
    ↓
[Plugin post-hook] → invoke_hook("post_tool_call", ...)
```

## Error Wrapping (Critical)

All handlers **MUST** return a JSON string. **NEVER** raise exceptions from handlers.

```python
# CORRECT
return json.dumps({"error": "WEATHER_API_KEY not configured"})

# WRONG — will be caught and wrapped anyway, but breaks schema
raise ValueError("API key missing")
```

Two-level error wrapping:
1. `registry.dispatch()` → catches handler exceptions → `{"error": "Tool execution failed: ExceptionType: message"}`
2. `handle_function_call()` → secondary catch → `{"error": "Error executing tool_name: message"}`

## Async Bridging

- **CLI path (no running loop)** → persistent event loop for cached async clients
- **Gateway path (running loop)** → disposable thread with `asyncio.run()`
- **Worker threads (parallel tools)** → per-thread persistent loops (thread-local storage)

## Dangerous Command Approval

`tools/approval.py` intercepts terminal commands matching `DANGEROUS_PATTERNS`:
- Recursive deletes (`rm -rf`)
- Filesystem formatting (`mkfs`, `dd`)
- SQL destructive operations
- System config overwrites (`> /etc/`)
- Service manipulation (`systemctl stop`)
- Remote code execution (`curl | sh`)
- Fork bombs, process kills

Approval flow:
1. Pattern detection before command execution
2. CLI mode: interactive prompt (approve / deny / allow permanently)
3. Gateway mode: async callback to messaging platform
4. Smart approval: auxiliary LLM can auto-approve low-risk pattern matches
5. Session state: approved patterns cached per-session
6. Permanent allowlist: written to `config.yaml` `command_allowlist`

---

# 10. Built-in Tools Reference (All ~71 Tools)

## `browser` toolset

| Tool | Description |
|------|-------------|
| `browser_navigate` | Navigate to a URL. Must be called before other browser tools. |
| `browser_snapshot` | Get accessibility tree snapshot. Returns ref IDs (like @e1, @e2) for interaction. |
| `browser_click` | Click element by ref ID from snapshot. |
| `browser_type` | Type text into input field by ref ID. Clears field first. |
| `browser_press` | Press a keyboard key (Enter, Tab, shortcuts). |
| `browser_scroll` | Scroll page in a direction. |
| `browser_back` | Navigate back in browser history. |
| `browser_get_images` | Get all images on current page with URLs and alt text. |
| `browser_console` | Get browser console output and JS errors. |
| `browser_vision` | Screenshot + vision AI analysis. For CAPTCHAs, visual verification, complex layouts. |

### CDP-Gated Browser Tools (require Chrome DevTools Protocol endpoint)
| Tool | Requires |
|------|----------|
| `browser_cdp` | CDP endpoint via `/browser connect`, `browser.cdp_url` config, Browserbase, or Camofox |
| `browser_dialog` | CDP endpoint; responds to JS dialogs (alert/confirm/prompt/beforeunload) |

## `clarify` toolset
| Tool | Description |
|------|-------------|
| `clarify` | Ask user for clarification. Supports multiple-choice (up to 4) or free-form input. |

## `code_execution` toolset
| Tool | Description |
|------|-------------|
| `execute_code` | Run Python scripts that call Hermes tools programmatically. For 3+ tool calls with processing logic, large output reduction, or conditional branching. |

## `cronjob` toolset
| Tool | Description |
|------|-------------|
| `cronjob` | Manage scheduled tasks. Actions: `create`, `list`, `update`, `pause`, `resume`, `run`, `remove`. |

## `delegation` toolset
| Tool | Description |
|------|-------------|
| `delegate_task` | Spawn subagents with isolated contexts. Only final summary returned to parent. |

## `feishu_doc` toolset (Feishu document-comment handler only)
| Tool | Description |
|------|-------------|
| `feishu_doc_read` | Read Feishu/Lark document content. |

## `feishu_drive` toolset (Feishu document-comment handler only)
| Tool | Description |
|------|-------------|
| `feishu_drive_add_comment` | Add top-level comment on Feishu file. |
| `feishu_drive_list_comments` | List whole-document comments, most recent first. |
| `feishu_drive_list_comment_replies` | List replies on a specific comment thread. |
| `feishu_drive_reply_comment` | Post reply on Feishu comment thread with optional @-mention. |

## `file` toolset
| Tool | Description |
|------|-------------|
| `read_file` | Read text file with line numbers and pagination. Use instead of `cat/head/tail`. Format: `LINE_NUM\|CONTENT`. |
| `write_file` | Write content to file, completely replacing existing. Creates parent dirs. Use for full rewrites. |
| `patch` | Targeted find-and-replace edits. Fuzzy matching (9 strategies). Returns unified diff. Auto-runs syntax checks. |
| `search_files` | Search file contents or find files by name. Ripgrep-backed. Use instead of `grep/rg/find`. |

## `homeassistant` toolset (requires `HASS_TOKEN`)
| Tool | Description |
|------|-------------|
| `ha_list_entities` | List HA entities, optionally filtered by domain or area. |
| `ha_get_state` | Get detailed state of a single HA entity including all attributes. |
| `ha_list_services` | List available HA services/actions for device control. |
| `ha_call_service` | Call an HA service to control a device. |

## `computer_use` toolset (macOS only, requires `cua-driver`)
| Tool | Description |
|------|-------------|
| `computer_use` | Background macOS desktop control (screenshots, click/drag/scroll/type/key/wait). Does NOT steal cursor/keyboard focus. |

## `image_gen` toolset (requires `FAL_KEY`)
| Tool | Description |
|------|-------------|
| `image_generate` | Generate images from text prompts via FAL.ai. Default model: FLUX 2 Klein 9B. Returns URL. |

## `kanban` toolset (dispatcher-spawned workers OR explicit `kanban` toolset)
| Tool | Access Level |
|------|-------------|
| `kanban_show` | Show active kanban task (title, description, comments, dependencies). |
| `kanban_list` | List board tasks with filters. **Orchestrator-only.** |
| `kanban_complete` | Mark task done with structured handoff (results, artifacts, follow-ups). |
| `kanban_block` | Block task on a user question. Dispatcher pauses; resumes after reply. |
| `kanban_heartbeat` | Send progress heartbeat for long-running operations. |
| `kanban_comment` | Add comment to task thread without changing state. |
| `kanban_create` | Fan out child tasks from current task. |
| `kanban_link` | Link tasks with parent→child dependency edge. |
| `kanban_unblock` | Return blocked task to `ready`. **Orchestrator-only.** |

**Note**: `kanban` toolset is NOT enabled by `all`/`*` wildcard. Must be listed explicitly or via `HERMES_KANBAN_TASK` env.

## `memory` toolset
| Tool | Description |
|------|-------------|
| `memory` | Save facts to persistent memory (MEMORY.md). Survives across sessions. Appears in system prompt at session start. |

## `messaging` toolset
| Tool | Description |
|------|-------------|
| `send_message` | Send message to connected messaging platform. Call with `action='list'` first when targeting specific channel/person. |

## `moa` toolset (requires `OPENROUTER_API_KEY`)
| Tool | Description |
|------|-------------|
| `mixture_of_agents` | Route problem through 4 reference models + 1 aggregator. Use sparingly (5 API calls). Best for complex math, algorithms. |

## `session_search` toolset
| Tool | Description |
|------|-------------|
| `session_search` | Search past sessions in local SQLite DB. FTS5-backed. Returns actual messages. Three shapes: discovery (query), scroll (session_id + around_message_id), browse (no args). |

## `skills` toolset
| Tool | Description |
|------|-------------|
| `skills_list` | List available skills (name + description). |
| `skill_view` | Load skill's full content or access linked files. Returns SKILL.md + skill directory path. |
| `skill_manage` | Create, update, delete skills. New skills go to `~/.hermes/skills/`. |

## `terminal` toolset
| Tool | Description |
|------|-------------|
| `terminal` | Execute shell commands. Filesystem persists between calls. `background=true` for servers. `notify_on_complete=true` for auto-notification on completion. Do NOT use `cat/head/tail` — use `read_file`. Do NOT use `grep/rg/find` — use `search_files`. |
| `process` | Manage background processes. Actions: `list`, `poll`, `log`, `wait`, `kill`, `write`. |

## `todo` toolset
| Tool | Description |
|------|-------------|
| `todo` | Manage task list for current session. No params = read current list. For 3+ step complex tasks. |

## `vision` toolset
| Tool | Description |
|------|-------------|
| `vision_analyze` | Analyze images with vision AI. On vision-capable models: returns raw pixels as multimodal result. On text-only models: falls back to auxiliary vision model description. |

## `video` toolset (opt-in)
| Tool | Description |
|------|-------------|
| `video_analyze` | Analyze video content from URL or file. Captions, scene breakdowns, key timestamps. |

## `video_gen` toolset (opt-in, requires backend plugin)
| Tool | Description |
|------|-------------|
| `video_generate` | Text-to-video or image-to-video. Pass `image_url` to animate; omit for text-only. Backends: xAI Grok-Imagine, FAL.ai (Veo 3.1, Pixverse v6, Kling O3). |

## `web` toolset
| Tool | Description |
|------|-------------|
| `web_search` | Search web. Up to 100 results. Supports `site:`, `filetype:`, `intitle:`, `-term`, `"exact phrase"` operators. Requires: EXA_API_KEY or PARALLEL_API_KEY or FIRECRAWL_API_KEY or TAVILY_API_KEY |
| `web_extract` | Extract page content as markdown. Also handles PDF URLs. Pages >5000 chars are LLM-summarized. Same key requirements. |

## `x_search` toolset (opt-in, requires xAI credentials)
| Tool | Description |
|------|-------------|
| `x_search` | Search X (Twitter) posts, profiles, threads via xAI's built-in tool. Off by default. |

## `tts` toolset
| Tool | Description |
|------|-------------|
| `text_to_speech` | Convert text to speech audio. Returns MEDIA: path. Telegram: voice bubble; Discord/WhatsApp: audio attachment; CLI: saves to `~/voice-memos/`. |

## `discord` toolset (gateway only)
| Tool | Description |
|------|-------------|
| `discord` | Read and participate in Discord. Actions: `search_members`, `fetch_messages`, `send_message`, `react`, `fetch_channel`, `list_channels`, etc. |

## `discord_admin` toolset (gateway only)
| Tool | Description |
|------|-------------|
| `discord_admin` | Manage Discord: list guilds/channels/roles, create/edit/delete channels, manage roles/timeouts/kicks/bans. |

## `spotify` toolset (registered by bundled `spotify` plugin)
| Tool | Description |
|------|-------------|
| `spotify_playback` | Control Spotify playback, inspect active state, fetch recently played. |
| `spotify_devices` | List Spotify Connect devices or transfer playback. |
| `spotify_queue` | Inspect queue or add items. |
| `spotify_search` | Search Spotify catalog (tracks, albums, artists, playlists, shows, episodes). |
| `spotify_playlists` | List, inspect, create, update, modify playlists. |
| `spotify_albums` | Fetch album metadata or tracks. |
| `spotify_library` | List, save, remove saved tracks/albums. |

## `hermes-yuanbao` toolset (Yuanbao platform only)
| Tool | Description |
|------|-------------|
| `yb_query_group_info` | Query basic Yuanbao group info. |
| `yb_query_group_members` | Query group members. |
| `yb_send_dm` | Send private/direct message with optional media. |
| `yb_search_sticker` | Search sticker catalogue by keyword. |
| `yb_send_sticker` | Send built-in sticker to current chat. |

---

# 11. Toolsets Reference (All Platform & Core Toolsets)

## Core Toolsets

| Toolset | Tools Included | Purpose |
|---------|---------------|---------|
| `browser` | All `browser_*` + `web_search` (fallback) | Browser automation |
| `clarify` | `clarify` | User clarification |
| `code_execution` | `execute_code` | Python sandbox execution |
| `cronjob` | `cronjob` | Scheduled tasks |
| `debugging` | `file` + `terminal` + `web` (composite) | Debug bundle |
| `delegation` | `delegate_task` | Subagent spawning |
| `discord` | `discord` | Discord read/write (gateway only) |
| `discord_admin` | `discord_admin` | Discord moderation (gateway only) |
| `feishu_doc` | `feishu_doc_read` | Feishu doc read |
| `feishu_drive` | `feishu_drive_*` (4 tools) | Feishu comment ops |
| `file` | `patch`, `read_file`, `search_files`, `write_file` | File operations |
| `homeassistant` | `ha_*` (4 tools) | Smart home control |
| `computer_use` | `computer_use` | macOS desktop automation |
| `context_engine` | (varies by plugin) | Active context engine tools |
| `image_gen` | `image_generate` | Image generation |
| `video_gen` | `video_generate` | Video generation |
| `kanban` | All `kanban_*` (9 tools) | Multi-agent coordination |
| `memory` | `memory` | Persistent memory |
| `messaging` | `send_message` | Cross-platform messaging |
| `moa` | `mixture_of_agents` | Multi-model consensus |
| `safe` | `image_generate`, `vision_analyze`, `web_extract`, `web_search` | Read-only research + media |
| `search` | `web_search` | Web search only |
| `session_search` | `session_search` | Past session search |
| `skills` | `skill_manage`, `skill_view`, `skills_list` | Skill management |
| `spotify` | All `spotify_*` (7 tools) | Spotify control |
| `terminal` | `process`, `terminal` | Shell execution |
| `todo` | `todo` | Session task list |
| `tts` | `text_to_speech` | TTS audio |
| `vision` | `vision_analyze` | Image analysis |
| `video` | `video_analyze` | Video analysis |
| `web` | `web_extract`, `web_search` | Web research |
| `x_search` | `x_search` | X/Twitter search |
| `yuanbao` | All `yb_*` (5 tools) | Yuanbao DM/group |

## Platform Toolsets

| Toolset | Differences from `hermes-cli` |
|---------|------------------------------|
| `hermes-cli` | **Full default toolset** — file, terminal, web, browser, memory, skills, vision, image_gen, todo, tts, delegation, code_execution, cronjob, session_search, clarify, safe, messaging |
| `hermes-acp` | Drops: `clarify`, `cronjob`, `image_generate`, `send_message`, `text_to_speech`, all 4 HA tools |
| `hermes-api-server` | Drops: `clarify`, `send_message`, `text_to_speech` |
| `hermes-cron` | Same as `hermes-cli` |
| `hermes-telegram` | Same as `hermes-cli` |
| `hermes-discord` | Adds `discord` + `discord_admin` on top of `hermes-cli` |
| `hermes-slack` | Same as `hermes-cli` |
| `hermes-whatsapp` | Same as `hermes-cli` |
| `hermes-signal` | Same as `hermes-cli` |
| `hermes-matrix` | Same as `hermes-cli` |
| `hermes-mattermost` | Same as `hermes-cli` |
| `hermes-email` | Same as `hermes-cli` |
| `hermes-sms` | Same as `hermes-cli` |
| `hermes-bluebubbles` | Same as `hermes-cli` |
| `hermes-dingtalk` | Same as `hermes-cli` |
| `hermes-feishu` | Adds 5 `feishu_doc_*`/`feishu_drive_*` tools |
| `hermes-qqbot` | Same as `hermes-cli` |
| `hermes-wecom` | Same as `hermes-cli` |
| `hermes-wecom-callback` | Same as `hermes-cli` |
| `hermes-weixin` | Same as `hermes-cli` |
| `hermes-yuanbao` | Adds 5 `yb_*` tools |
| `hermes-homeassistant` | Same as `hermes-cli` (HA tools activate when HASS_TOKEN set) |
| `hermes-webhook` | Same as `hermes-cli` |
| `hermes-gateway` | Union of every `hermes-<platform>` toolset |

## Configuring Toolsets

```bash
# Per-session (CLI)
hermes chat --toolsets web,file,terminal
hermes chat --toolsets debugging        # composite expands to file + terminal + web
hermes chat --toolsets all              # everything

# Per-platform (config.yaml)
toolsets:
  - hermes-cli

# Interactive management
hermes tools                            # curses UI
# In-session:
/tools list
/tools disable browser
/tools enable homeassistant
```

## Dynamic Toolsets

### MCP Server Toolsets
Each configured MCP server creates `mcp-<server>` toolset at runtime:
```yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
```
Creates `mcp-github` toolset with all tools the server exposes.

MCP tool naming: `mcp_<server>_<tool>` (hyphens/dots → underscores)
Examples: `mcp_github_create_issue`, `mcp_filesystem_read_file`

### Plugin Toolsets
Plugins register via `ctx.register_tool()` during initialization.

### Custom Toolsets (config.yaml)
```yaml
custom_toolsets:
  data-science:
    - file
    - terminal
    - code_execution
    - web
    - vision
```

## Wildcard Behavior
- `all` or `*` — all registered toolsets (built-in + dynamic + plugin)
- **NOT** enabled by `all`/`*`: `kanban`, and capability-gated tools requiring backends

## Tool-Level vs Toolset-Level Disable
`hermes tools` curses UI operates at the tool level (finer than toolsets). Disabled tools persist to `config.yaml` and are filtered even if their toolset is enabled.

---

# 12. Skills System — SKILL.md Format & Full Spec

## When to Use Skills vs Tools

| Make it a **Skill** when | Make it a **Tool** when |
|--------------------------|------------------------|
| Capability = instructions + shell commands + existing tools | Requires end-to-end API key / auth flow integration |
| Wraps external CLI or API via `terminal` or `web_extract` | Needs custom Python logic that must execute precisely |
| No custom Python integration needed | Handles binary data, streaming, or real-time events |
| Examples: arXiv search, git workflows, Docker management | Examples: browser automation, TTS, vision analysis |

## Skill Directory Structure

```
skills/
├── research/
│   └── arxiv/
│       ├── SKILL.md              # Required: main instructions
│       └── scripts/              # Optional: helper scripts
│           └── search_arxiv.py
├── productivity/
│   └── ocr-and-documents/
│       ├── SKILL.md
│       ├── scripts/
│       └── references/
└── ...

optional-skills/                  # Official optional (install explicitly)
~/.hermes/skills/                 # User-created custom skills
```

## SKILL.md Format — Complete Spec

```markdown
---
name: my-skill
description: Brief description (shown in skill search results)
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]          # Optional — restrict to specific OS
                                   # Valid: macos, linux, windows
                                   # Omit to load on all platforms (default)
metadata:
  hermes:
    tags: [Category, Subcategory, Keywords]
    related_skills: [other-skill-name]
    requires_toolsets: [web]            # Hide if this toolset is NOT active
    requires_tools: [web_search]        # Hide if this tool is NOT available
    fallback_for_toolsets: [browser]    # Hide if this toolset IS active
    fallback_for_tools: [browser_navigate]  # Hide if this tool IS available
    config:                              # Non-secret config.yaml settings
      - key: my.setting
        description: "What this setting controls"
        default: "sensible-default"
        prompt: "Display prompt for setup"
    blueprint:                           # Makes skill a runnable automation
      schedule: "0 9 * * *"             # cron expr / "every 2h" / ISO timestamp
      deliver: origin                   # optional (default: origin)
      prompt: "Task instruction for each run"  # optional
      no_agent: false                   # optional
required_environment_variables:          # Secrets — stored in .env
  - name: MY_API_KEY
    prompt: "Enter your API key"
    help: "Get one at https://example.com"
    required_for: "API access"
required_credential_files:              # OAuth/file-based credentials
  - path: google_token.json
    description: Google OAuth2 token (created by setup script)
---

# Skill Title

Brief intro.

## When to Use
Trigger conditions — when should the agent load this skill?

## Quick Reference
Table of common commands or API calls.

## Procedure
Step-by-step instructions the agent follows.

## Pitfalls
Known failure modes and how to handle them.

## Verification
How the agent confirms it worked.
```

## Conditional Skill Activation

| Field | Behavior |
|-------|----------|
| `requires_toolsets` | Skill is **hidden** when ANY listed toolset is **not** available |
| `requires_tools` | Skill is **hidden** when ANY listed tool is **not** available |
| `fallback_for_toolsets` | Skill is **hidden** when ANY listed toolset **is** available |
| `fallback_for_tools` | Skill is **hidden** when ANY listed tool **is** available |

**Use case for `fallback_for_*`**: Create a DuckDuckGo skill with `fallback_for_tools: [web_search]` — only shows when the web search tool isn't configured.

## Template Variables in SKILL.md

When a skill is loaded, two template tokens are substituted in the SKILL.md body:

| Token | Replaced With |
|-------|---------------|
| `${HERMES_SKILL_DIR}` | Absolute path to the skill's directory |
| `${HERMES_SESSION_ID}` | The active session ID (left in place if no session) |

Example usage:
```markdown
To analyse the input, run:

    node ${HERMES_SKILL_DIR}/scripts/analyse.js <input>
```

Disable globally with `skills.template_vars: false` in `config.yaml`.

## Inline Shell Snippets (opt-in)

```markdown
Current date: !`date -u +%Y-%m-%d`
Git branch: !`git -C ${HERMES_SKILL_DIR} rev-parse --abbrev-ref HEAD`
```

Enable in `config.yaml`:
```yaml
skills:
  inline_shell: true
  inline_shell_timeout: 10   # seconds per snippet
```

**Off by default** — snippets run on host without approval. Only enable for trusted sources.
Snippets run with skill directory as CWD. Output capped at 4000 chars.

## Secure Setup on Load

```yaml
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API key
    help: Get a key from https://developers.google.com/tenor
    required_for: full functionality
```

- Missing vars do NOT hide the skill from discovery
- Hermes prompts securely in local CLI
- Gateway/messaging sessions show local setup guidance
- Hermes NEVER exposes raw secret value to the model
- When skill loads: declared env vars that are set are **automatically passed through** to `execute_code` and `terminal` sandboxes

## Config Settings in Skills (non-secret)

```yaml
metadata:
  hermes:
    config:
      - key: myplugin.path
        description: Path to plugin data directory
        default: "~/myplugin-data"
        prompt: Plugin data directory path
```

Stored under `skills.config.<key>` in `config.yaml`. Injected into skill message at load time:
```
[Skill config (from ~/.hermes/config.yaml):
  myplugin.path = /home/user/my-data
]
```

Manual set: `hermes config set skills.config.myplugin.path ~/my-data`

## Delivering Media from Skills

Emit `[[as_document]]` anywhere in the response to deliver media as downloadable file attachments instead of inline image bubbles:
```markdown
Here is your chart.
[[as_document]]
```

## Blueprints (Skill + Automation)

A blueprint is a skill with a `metadata.hermes.blueprint` block. Installing a blueprint adds a **suggested** cron job (NOT auto-scheduled):

```bash
hermes skills install owner/morning-brief
# → Added to suggestions — run /suggestions to schedule or dismiss it.

/suggestions             # list pending
/suggestions accept 1    # creates the cron job
/suggestions dismiss 1   # never offer again
```

Blueprint suggestions table:

| Source | Trigger |
|--------|---------|
| `catalog` | Curated starters (`/suggestions catalog`) — daily briefing, mail monitor, weekly review |
| `blueprint` | You installed a skill with `blueprint:` block |
| `usage` | Background review noticed a recurring ask |
| `integration` | You connected an account (Gmail, GitHub, etc.) |

## Skill Trust Levels

| Level | Source | Behavior |
|-------|--------|----------|
| `builtin` | Ships with Hermes | Always trusted |
| `official` | `optional-skills/` in repo | Built-in trust, no third-party warning |
| `trusted` | From `openai/skills`, `anthropics/skills`, `huggingface/skills` | Trusted |
| `community` | Non-dangerous findings overridable with `--force` | `dangerous` verdicts remain blocked |

## Security Scanning (Hub-Installed Skills)

Checks for:
- Data exfiltration patterns
- Prompt injection attempts
- Destructive commands
- Shell injection

## Publishing Skills

```bash
hermes skills publish skills/my-skill --to github --repo owner/repo

# Add custom repository as tap:
hermes skills tap add owner/repo
```

## Skill Locations

| Location | Discovery | Trust |
|----------|-----------|-------|
| `skills/` | Ships with every install | builtin |
| `optional-skills/` | Install via `hermes skills browse` | official |
| `~/.hermes/skills/` | Personal user skills | local |
| External tap/hub | Install via `hermes skills install` | community |

---

# 13. Memory System — MEMORY.md, USER.md, Providers

## Built-in Memory Files

| File | Purpose | Who Maintains It |
|------|---------|-----------------|
| `~/.hermes/MEMORY.md` | Persistent facts about environment, preferences, project context | Agent (via `memory` tool) |
| `~/.hermes/USER.md` | User profile: name, preferences, GitHub handle, etc. | Agent (via `memory` tool) |

## Memory Snapshot in System Prompt

Both files are snapshot-frozen into the **volatile tier** of the system prompt at session start. They do NOT update mid-session until:
- New session starts
- Compression-triggered rebuild

## The `memory` Tool

```
Tool: memory
Toolset: memory
Description: Save important information to persistent memory that survives
across sessions. Memory appears in system prompt at session start.

WHEN TO SAVE:
- User preferences and working styles
- Environment details and tool quirks
- Project conventions and stable facts
- Information the agent would otherwise lose between sessions
```

## External Memory Providers

Only ONE external provider can be active at a time. Configured in `config.yaml`:
```yaml
memory:
  provider: "honcho"    # or any installed provider name
```

### Memory Provider Plugin Interface

See Section 21 for full plugin development guide.

Lifecycle hooks:
- `system_prompt_block()` — static provider info in system prompt
- `prefetch(query)` — before each API call, return recalled context
- `queue_prefetch(query)` — after each turn, pre-warm for next turn
- `sync_turn(user, assistant)` — after each completed turn (must be non-blocking)
- `on_session_end(messages)` — conversation ends
- `on_pre_compress(messages)` — before context compression
- `on_memory_write(action, target, content)` — on built-in memory writes
- `shutdown()` — process exit cleanup

## Memory Flush Lifecycle

Before context compression or session end:
1. Memory is flushed to disk first (preventing data loss)
2. `on_session_end()` fires for external provider
3. After flush, compression/session cleanup proceeds

---

# 14. Context Compression & Prompt Caching

## Dual Compression System

```
                 ┌──────────────────────────┐
Incoming message │   Gateway Session Hygiene │  Fires at 85% of context
─────────────────►   (pre-agent, rough est.) │  Safety net for large sessions
                 └─────────────┬────────────┘
                               │
                               ▼
                 ┌──────────────────────────┐
                 │   Agent ContextCompressor │  Fires at 50% of context (default)
                 │   (in-loop, real tokens)  │  Normal context management
                 └──────────────────────────┘
```

## Configuration

```yaml
compression:
  enabled: true
  threshold: 0.50             # Fraction of context window
  target_ratio: 0.20          # Tail token budget ratio
  protect_last_n: 20          # Min recent messages always preserved
  codex_gpt55_autoraise: true # Raise to 85% for gpt-5.5 on Codex OAuth

auxiliary:
  compression:
    model: null               # Summary model (null = auto-detect)
    provider: auto
    base_url: null
```

## Parameter Semantics

| Parameter | Default | Description |
|-----------|---------|-------------|
| `threshold` | `0.50` | Fires when prompt tokens ≥ threshold × context_length |
| `target_ratio` | `0.20` | Tail token budget = threshold_tokens × target_ratio |
| `protect_last_n` | `20` | Min messages preserved in tail |
| `protect_first_n` | `3` (hardcoded) | System prompt + first exchange always preserved |

## Computed Values (200K context model, defaults)

```
context_length       = 200,000
threshold_tokens     = 200,000 × 0.50 = 100,000
tail_token_budget    = 100,000 × 0.20 = 20,000
max_summary_tokens   = min(200,000 × 0.05, 12,000) = 10,000
```

## Compression Algorithm (4 Phases)

### Phase 1: Prune Old Tool Results
Old tool results (>200 chars) outside protected tail replaced with:
```
[Old tool output cleared to save context space]
```

### Phase 2: Determine Boundaries

```
[0..2]  ← protect_first_n (system + first exchange)
[3..N]  ← middle turns → SUMMARIZED
[N..end] ← tail (by token budget OR protect_last_n)
```

Boundaries aligned to avoid splitting tool_call/tool_result groups.

### Phase 3: Generate Structured Summary

Template:
```
## Goal
## Constraints & Preferences
## Progress
### Done
### In Progress
### Blocked
## Key Decisions
## Relevant Files
## Next Steps
## Critical Context
```

Summary budget: `content_tokens × 0.20` (min 2,000, max min(context × 0.05, 12,000))

### Phase 4: Assemble Compressed Messages

1. Head messages (system prompt gets compression note appended on first compression)
2. Summary message (role chosen to avoid same-role violations)
3. Tail messages (unmodified)

Orphaned tool_call/tool_result pairs cleaned by `_sanitize_tool_pairs()`.

### Iterative Re-compression

Previous summary passed to LLM with instructions to UPDATE rather than re-summarize. Items move from "In Progress" to "Done". `_previous_summary` field stores last summary.

## Pluggable Context Engine

```yaml
context:
  engine: "compressor"    # default built-in lossy summarization
  engine: "lcm"           # example plugin providing lossless context
```

Selection: `config.yaml` → `context.engine`. Resolution order:
1. `plugins/context_engine/<name>/` directory
2. General plugin system (`register_context_engine()`)
3. Fall back to built-in `ContextCompressor`

Plugin engines are **never auto-activated** — must explicitly set `context.engine`.

## Anthropic Prompt Caching

Source: `agent/prompt_caching.py`

Strategy: **system_and_3** (Anthropic max 4 breakpoints):
```
Breakpoint 1: System prompt (stable across all turns)
Breakpoint 2: 3rd-to-last non-system message ─┐
Breakpoint 3: 2nd-to-last non-system message  ├─ Rolling window
Breakpoint 4: Last non-system message          ─┘
```

Cache marker format:
```python
{"type": "ephemeral"}           # 5-minute TTL
{"type": "ephemeral", "ttl": "1h"}  # 1-hour TTL
```

Content type → marker placement:
| Content Type | Marker Placement |
|-------------|-----------------|
| String content | Convert to `[{"type": "text", "text": ..., "cache_control": ...}]` |
| List content | Added to last element's dict |
| None/empty | Added as `msg["cache_control"]` |
| Tool messages | Added as `msg["cache_control"]` (native Anthropic only) |

Configure TTL:
```yaml
prompt_caching:
  cache_ttl: "5m"    # "5m" | "1h"
```

Auto-enabled when:
- Model is Anthropic Claude (detected by model name)
- Provider supports `cache_control` (native Anthropic API or OpenRouter)

---

# 15. Session Storage — SQLite Schema

Database location: `~/.hermes/state.db` (WAL mode)
Source file: `hermes_state.py`

## Schema Overview

```
~/.hermes/state.db (SQLite, WAL mode)
├── sessions              — Session metadata, token counts, billing
├── messages              — Full message history per session
├── messages_fts          — FTS5 virtual table (content + tool_name + tool_calls)
├── messages_fts_trigram  — FTS5 with trigram tokenizer (CJK / substring search)
├── state_meta            — Key/value metadata table
└── schema_version        — Single-row table tracking migration state
```

Current schema version: **11**

## Sessions Table

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,           -- "cli", "telegram", "discord", etc.
    user_id TEXT,
    model TEXT,
    model_config TEXT,
    system_prompt TEXT,
    parent_session_id TEXT,
    started_at REAL NOT NULL,       -- Unix epoch float
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    api_call_count INTEGER DEFAULT 0,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE INDEX IF NOT EXISTS idx_sessions_source ON sessions(source);
CREATE INDEX IF NOT EXISTS idx_sessions_parent ON sessions(parent_session_id);
CREATE INDEX IF NOT EXISTS idx_sessions_started ON sessions(started_at DESC);
CREATE UNIQUE INDEX IF NOT EXISTS idx_sessions_title_unique
    ON sessions(title) WHERE title IS NOT NULL;
```

## Messages Table

```sql
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,             -- "system", "user", "assistant", "tool"
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,                -- JSON string of tool call objects
    tool_name TEXT,
    timestamp REAL NOT NULL,        -- Unix epoch float
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,         -- JSON string
    codex_reasoning_items TEXT,     -- JSON string
    codex_message_items TEXT        -- JSON string (Codex Responses phase replay)
);

CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, timestamp);
```

## FTS5 Full-Text Search

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);
-- Kept in sync via INSERT/UPDATE/DELETE triggers
```

FTS5 Query Syntax:
| Syntax | Example | Meaning |
|--------|---------|---------|
| Keywords | `docker deployment` | Both terms (implicit AND) |
| Quoted phrase | `"exact phrase"` | Exact phrase match |
| Boolean OR | `docker OR kubernetes` | Either term |
| Boolean NOT | `python NOT java` | Exclude term |
| Prefix | `deploy*` | Prefix match |

## Python API

```python
from hermes_state import SessionDB

db = SessionDB()                           # Default: ~/.hermes/state.db
db = SessionDB(db_path=Path("/tmp/test.db"))

# Create session
db.create_session(
    session_id="sess_abc123",
    source="cli",
    model="anthropic/claude-opus-4.6",
    user_id="user_1",
    parent_session_id=None,
)

# Append message
msg_id = db.append_message(
    session_id="sess_abc123",
    role="assistant",
    content="Here's the answer...",
    tool_calls=[{"id": "call_1", "function": {"name": "terminal", "arguments": "{}"}}],
    token_count=150,
    finish_reason="stop",
    reasoning="Let me think...",
)

# Retrieve conversation
conversation = db.get_messages_as_conversation("sess_abc123")
# Returns: [{"role": "user", "content": "..."}, {"role": "assistant", ...}]

# Search
results = db.search_messages("docker deployment")
results = db.search_messages("error", source_filter=["cli"])
results = db.search_messages("bug", exclude_sources=["telegram"])

# Session titles (must be unique among non-NULL)
db.set_session_title("sess_abc123", "Fix Docker Build")
session_id = db.resolve_session_by_title("Fix Docker Build")
next_title = db.get_next_title_in_lineage("Fix Docker Build")  # "Fix Docker Build #2"

# Lifecycle
db.end_session("sess_abc123", end_reason="user_exit")
db.reopen_session("sess_abc123")

# Cleanup
deleted_count = db.prune_sessions(older_than_days=90)
db.delete_session("sess_abc123")
db.export_session("sess_abc123")  # → dict with session + messages
```

## Write Contention Handling

```python
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020   # 20ms
_WRITE_RETRY_MAX_S = 0.150   # 150ms
_CHECKPOINT_EVERY_N_WRITES = 50
```

- Short SQLite timeout (1 second) not default 30s
- Application-level retry with random jitter
- `BEGIN IMMEDIATE` transactions
- WAL checkpoints every 50 writes (PASSIVE mode)

## Session Lineage (Compression-Triggered Splits)

```sql
-- Find all ancestors of a session
WITH RECURSIVE lineage AS (
    SELECT * FROM sessions WHERE id = ?
    UNION ALL
    SELECT s.* FROM sessions s
    JOIN lineage l ON s.id = l.parent_session_id
)
SELECT id, title, started_at, parent_session_id FROM lineage;
```

---

# 16. MCP (Model Context Protocol) Integration

## Config Structure

```yaml
mcp_servers:
  <server_name>:
    command: "..."         # stdio servers
    args: []
    env: {}

    # OR
    url: "..."             # HTTP servers
    headers: {}

    ssl_verify: true       # bool or CA bundle path
    client_cert: "..."     # mTLS cert (combined PEM, or [cert, key], or [cert, key, pass])
    client_key: "..."      # mTLS key (when cert is separate)

    enabled: true
    timeout: 120
    connect_timeout: 60
    supports_parallel_tool_calls: false
    auth: oauth            # OAuth 2.1 PKCE (HTTP only)
    tools:
      include: []          # Whitelist (wins over exclude if both set)
      exclude: []          # Blacklist
      resources: true      # Enable list_resources + read_resource
      prompts: true        # Enable list_prompts + get_prompt
```

## Tool Naming Convention

```
mcp_<server>_<tool>

Examples:
  mcp_github_create_issue
  mcp_filesystem_read_file
  mcp_my_api_query_data

Utility tools:
  mcp_<server>_list_resources
  mcp_<server>_read_resource
  mcp_<server>_list_prompts
  mcp_<server>_get_prompt
```

**Name sanitization**: Hyphens (`-`) and dots (`.`) in server names AND tool names → underscores (`_`).

IMPORTANT: Use **original** MCP tool names (with hyphens/dots) in `include`/`exclude` filters, not the sanitized version.

## Filtering Semantics

- `include` set → only those tools registered (whitelist wins)
- `exclude` set, no `include` → all except excluded
- Both set → `include` wins, `exclude` ignored
- Empty `include: []` with `resources: true` → resource-only server (no native tools)

## Capability-Aware Registration

Utility tools (`list_resources`, `list_prompts`) only register if the MCP session actually exposes that capability. Setting `resources: true` when server doesn't support it is a no-op.

## OAuth 2.1

```yaml
mcp_servers:
  protected_api:
    url: "https://mcp.example.com/mcp"
    auth: oauth
```

- Uses MCP SDK's OAuth 2.1 PKCE flow
- First connect: browser window opens for authorization
- Tokens persisted to `~/.hermes/mcp-tokens/<server>.json`
- Token refresh is automatic

## Reloading Config

```
/reload-mcp
```

## Example Configs

### GitHub Allowlist
```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "***"
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      resources: false
      prompts: false
```

### mTLS
```yaml
mcp_servers:
  internal_api:
    url: "https://mcp.internal.example.com/mcp"
    client_cert: "~/secrets/mcp-client.pem"

  partner_api:
    url: "https://mcp.partner.example.com/mcp"
    client_cert: "~/secrets/client.crt"
    client_key: "~/secrets/client.key"

  bank_api:
    url: "https://mcp.bank.example.com/mcp"
    client_cert: ["~/secrets/client.crt", "~/secrets/client.key", "my-passphrase"]

  lab_api:
    url: "https://mcp.lab.local/mcp"
    ssl_verify: "~/secrets/lab-ca.pem"
    client_cert: "~/secrets/lab-client.pem"
```

---

# 17. Provider Runtime Resolution

## Resolution Precedence

1. Explicit CLI/runtime request (highest priority)
2. `config.yaml` model/provider config
3. Environment variables
4. Provider-specific defaults / auto resolution

## Supported Providers

| Provider ID | Type |
|-------------|------|
| `openrouter` | OpenRouter API |
| `nous` | Nous Portal |
| `openai-codex` | OpenAI Codex / Responses API |
| `copilot` | GitHub Copilot |
| `anthropic` | Native Anthropic Messages API |
| `gemini`, `google-gemini-cli` | Google Gemini |
| `alibaba`, `alibaba-coding-plan` | Alibaba DashScope |
| `deepseek` | DeepSeek |
| `z.ai` | Z.AI |
| `kimi-coding`, `kimi-coding-cn` | Kimi / Moonshot |
| `minimax`, `minimax-cn`, `minimax-oauth` | MiniMax |
| `kilo-code` | Kilo Code |
| `huggingface` | Hugging Face |
| `opencode-zen`, `opencode-go` | OpenCode |
| `aws-bedrock` | AWS Bedrock |
| `azure-foundry` | Azure Foundry |
| `nvidia-nim` | NVIDIA NIM |
| `xai` | xAI (Grok) |
| `arcee` | Arcee |
| `gmi-cloud` | GMI Cloud |
| `stepfun` | StepFun |
| `qwen-oauth` | Qwen OAuth |
| `xiaomi` | Xiaomi |
| `ollama-cloud` | Ollama Cloud |
| `lm-studio` | LM Studio |
| `tencent-tokenhub` | Tencent TokenHub |
| `custom` | Any OpenAI-compatible endpoint |
| (named custom) | `custom_providers` list in config.yaml |

## Adding a Custom Provider

```yaml
# config.yaml
custom_providers:
  - name: "my-local"
    base_url: "http://localhost:8000/v1"
    model: "my-model"
    api_key: ""

# Or use the built-in custom provider:
provider: custom
model: my-model
```

## Resolution Output

```python
{
    "provider": "openrouter",
    "api_mode": "chat_completions",  # or "codex_responses" | "anthropic_messages"
    "base_url": "https://openrouter.ai/api/v1",
    "api_key": "sk-or-...",
    "source": "env"
}
```

## Anthropic Native Path

```
provider = "anthropic"
→ api_mode = anthropic_messages
→ client = anthropic.Anthropic (via anthropic_adapter.py)
→ Prefers refreshable Claude Code credentials over copied env tokens
→ Preflights credential refresh before API calls
→ Retries once on 401 after rebuilding Anthropic client
```

## Auxiliary Model Routing

Auxiliary tasks (vision, compression, web extraction, memory flushes) can use their own provider/model:

```yaml
auxiliary:
  compression:
    model: null       # null = auto-detect from main model
    provider: auto    # "auto" | "openrouter" | "nous" | "main" | etc.
  vision:
    model: null
    provider: auto
```

When set to `main`, resolves through same shared runtime path as normal chat.

## Fallback Model Config

```yaml
fallback_providers:
  - provider: "openrouter"
    model: "anthropic/claude-3.5-haiku"
  - provider: "openrouter"
    model: "openai/gpt-4o-mini"
```

Fallback triggers:
- After max retries on invalid API responses
- On non-retryable client errors (HTTP 401, 403, 404)
- After max retries on transient errors (HTTP 429, 500, 502, 503)

Fallback is **single-shot** — once activated, won't activate again.

---

# 18. Gateway Internals (Messaging Platforms)

## Key Files

| File | Purpose |
|------|---------|
| `gateway/run.py` | `GatewayRunner` — main loop, slash commands, message dispatch |
| `gateway/session.py` | `SessionStore` — conversation persistence + session key construction |
| `gateway/delivery.py` | Outbound message delivery |
| `gateway/pairing.py` | DM pairing authorization flow |
| `gateway/hooks.py` | Hook discovery, loading, lifecycle event dispatch |
| `gateway/mirror.py` | Cross-session message mirroring for `send_message` |
| `gateway/status.py` | Token lock management, profile-scoped process tracking |

## Platform Adapters (gateway/platforms/)

```
telegram.py        discord.py         slack.py
whatsapp.py        signal.py          matrix.py
mattermost.py      email.py           sms.py
dingtalk.py        feishu.py          wecom.py
weixin.py          bluebubbles.py     qqbot/
yuanbao.py         feishu_comment.py  msgraph_webhook.py
webhook.py         api_server.py      homeassistant.py
```

## Session Key Format

```
agent:main:{platform}:{chat_type}:{chat_id}

Examples:
  agent:main:telegram:private:123456789
  agent:main:discord:guild:987654321

Thread-aware (Telegram topics, Discord threads, Slack threads):
  chat_id may include thread ID
```

**NEVER construct session keys manually** — use `build_session_key()` from `gateway/session.py`.

## Message Flow

```
Platform event → Adapter.on_message() → MessageEvent
  → GatewayRunner._handle_message()
    → Resolve session key via _session_key_for_source()
    → Check authorization
    → Check if slash command → dispatch to command handler
    → Check if agent running → intercept /stop, /status
    → Otherwise → create AIAgent instance + run conversation
    → Deliver response through platform adapter
```

## Two-Level Message Guard

1. **Base adapter** (`gateway/platforms/base.py`): Checks `_active_sessions`. Queues message in `_pending_messages`, sets interrupt event.
2. **Gateway runner**: Checks `_running_agents`. Routes `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny` inline. Everything else → `running_agent.interrupt()`.

## Authorization Flow (Evaluated in Order)

1. Per-platform allow-all flag (e.g., `TELEGRAM_ALLOW_ALL_USERS`)
2. Platform allowlist (e.g., `TELEGRAM_ALLOWED_USERS`)
3. DM pairing codes
4. Global allow-all (`GATEWAY_ALLOW_ALL_USERS`)
5. **Default: deny**

DM Pairing:
```
Admin: /pair
Gateway: "Pairing code: ABC123. Share with the user."
New user: ABC123
Gateway: "Paired! You're now authorized."
```

## Gateway Hooks

```
Location: ~/.hermes/hooks/
Structure: <hook-name>/ directory with HOOK.yaml + handler.py
```

Events:
| Event | When Fired |
|-------|-----------|
| `gateway:startup` | Gateway process starts |
| `session:start` | New conversation session begins |
| `session:end` | Session completes or times out |
| `session:reset` | User resets with `/new` |
| `agent:start` | Agent begins processing |
| `agent:step` | Agent completes one tool-calling iteration |
| `agent:end` | Agent finishes |
| `command:*` | Any slash command executed |

## Process Management

```bash
hermes gateway start
hermes gateway stop              # stops current profile's gateway
hermes gateway stop --all        # kills all gateway processes

PID file: ~/.hermes/gateway.pid  # profile-scoped
```

## Delivery Targets

| Target | Syntax |
|--------|--------|
| Origin chat | `origin` |
| Local file | `local` |
| Telegram | `telegram` or `telegram:<chat_id>` or `telegram:<chat_id>:<thread_id>` |
| Discord | `discord` or `discord:#channel` |
| Slack | `slack` |
| WhatsApp | `whatsapp` |
| Signal | `signal` |
| Matrix | `matrix` |
| Mattermost | `mattermost` |
| Email | `email` |
| SMS | `sms` |

---

# 19. Cron Internals

Source files: `cron/jobs.py`, `cron/scheduler.py`, `tools/cronjob_tools.py`

## Schedule Formats

| Format | Example | Behavior |
|--------|---------|----------|
| Relative delay | `30m`, `2h`, `1d` | One-shot, fires after duration |
| Interval | `every 2h`, `every 30m` | Recurring at regular intervals |
| Cron expression | `0 9 * * *` | Standard 5-field cron syntax |
| ISO timestamp | `2025-01-15T09:00:00` | One-shot, fires at exact time |

## Job Storage (`~/.hermes/cron/jobs.json`)

```json
{
  "id": "a1b2c3d4e5f6",
  "name": "Daily briefing",
  "prompt": "Summarize today's AI news",
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "display": "0 9 * * *"
  },
  "skills": ["ai-funding-daily-report"],
  "deliver": "telegram:-1001234567890",
  "repeat": {
    "times": null,
    "completed": 42
  },
  "state": "scheduled",
  "enabled": true,
  "next_run_at": "2025-01-16T09:00:00Z",
  "last_run_at": "2025-01-15T09:00:00Z",
  "last_status": "ok",
  "created_at": "2025-01-01T00:00:00Z",
  "model": null,
  "provider": null,
  "script": null
}
```

## Job Lifecycle States

| State | Meaning |
|-------|---------|
| `scheduled` | Active, will fire at next scheduled time |
| `paused` | Suspended |
| `completed` | Repeat count exhausted or one-shot fired |
| `running` | Currently executing (transient) |

## Scheduler Tick Cycle (every 60 seconds)

```
tick()
  1. Acquire scheduler lock (prevents overlapping ticks)
  2. Load all jobs from jobs.json
  3. Filter due jobs (next_run <= now AND state == "scheduled")
  4. For each due job:
     a. Set state = "running"
     b. Create fresh AIAgent session (NO conversation history)
     c. Load attached skills (injected as user messages)
     d. Run job prompt through agent
     e. Deliver response to configured target
     f. Update run_count, compute next_run
     g. If repeat count exhausted → state = "completed"
     h. Otherwise → state = "scheduled"
  5. Write updated jobs to jobs.json
  6. Release scheduler lock
```

## Fresh Session Properties

- No conversation history from previous runs
- No memory of previous cron executions
- Prompt must be self-contained
- `cronjob` toolset disabled (recursion guard)

## Script-Backed Jobs

```json
{
  "script": "~/.hermes/scripts/check_competitors.py"
}
```

Script runs before each agent turn, stdout injected as prompt context.

Script timeout resolution (3-layer chain):
1. `HERMES_CRON_SCRIPT_TIMEOUT` env var
2. `cron.script_timeout_seconds` in `config.yaml`
3. Default: 120 seconds

## Response Wrapping

`[SILENT]` prefix in cron response suppresses delivery entirely.

```yaml
cron:
  wrap_response: true    # Wrap with job name/task header+footer (default)
```

## CLI Management

```bash
hermes cron list
hermes cron create                  # Interactive
hermes cron edit <job_id>
hermes cron pause <job_id>
hermes cron resume <job_id>
hermes cron run <job_id>            # Immediate execution
hermes cron remove <job_id>
```

---

# 20. Plugin System

## Discovery Sources (3 Sources)

1. `~/.hermes/plugins/` — user plugins
2. `.hermes/plugins/` — project plugins
3. pip entry points (`hermes_plugins` group)

## Plugin Types

| Type | Purpose | Selection |
|------|---------|-----------|
| General plugins | Register tools, hooks, CLI commands | Multiple can be active |
| Memory providers | `plugins/memory/<name>/` | Single-select only |
| Context engines | `plugins/context_engine/<name>/` | Single-select only |
| Model providers | `plugins/model-providers/<name>/` | One per provider ID |
| Video gen backends | `plugins/video_gen/<name>/` | One active backend |

## Plugin Context API

```python
def register(ctx) -> None:
    # Register a tool
    ctx.register_tool(name, schema, handler, toolset, check_fn=None)

    # Register a hook
    ctx.register_hook("pre_tool_call", my_hook_fn)
    ctx.register_hook("post_tool_call", my_hook_fn)
    ctx.register_hook("pre_llm_call", my_context_fn)

    # Register memory provider
    ctx.register_memory_provider(MyMemoryProvider())

    # Register context engine
    ctx.register_context_engine(MyContextEngine())

    # Register CLI commands
    ctx.register_cli_command("my-cmd", my_cmd_handler)
```

## plugin.yaml Structure

```yaml
name: my-plugin
version: 1.0.0
description: "What this plugin does"
hooks:
  - on_session_end
  - pre_llm_call
```

## Managing Plugins

```bash
hermes plugins                     # Curses UI
hermes plugins list
hermes plugins enable my-plugin
hermes plugins disable my-plugin
hermes memory setup                # Configure memory provider
hermes memory setup --provider honcho
```

## Pre-LLM Call Hook

`pre_llm_call` context lands in the **current turn's user message** (not the cached system prompt):

```python
def my_context_fn(session_id, query, **kwargs):
    return "Additional context: ..."
```

When multiple plugins return context, Hermes concatenates the context blocks.

---

# 21. Memory Provider Plugin Development

Source: `agent/memory_provider.py`

## Directory Structure

```
plugins/memory/my-provider/
├── __init__.py      # MemoryProvider implementation + register()
├── plugin.yaml      # Metadata
├── cli.py           # Optional: CLI subcommands (register_cli)
└── README.md        # Setup instructions
```

## MemoryProvider ABC

```python
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    @property
    def name(self) -> str:
        return "my-provider"

    def is_available(self) -> bool:
        """NO network calls here."""
        return bool(os.environ.get("MY_API_KEY"))

    def initialize(self, session_id: str, **kwargs) -> None:
        """kwargs always includes hermes_home (str)."""
        self._api_key = os.environ.get("MY_API_KEY", "")
        self._session_id = session_id

    def get_tool_schemas(self) -> list:
        """Return OpenAI function-calling schemas for your tools."""
        return [...]

    def handle_tool_call(self, tool_name: str, args: dict, **kwargs) -> str:
        """Handle tool calls. Must return JSON string."""
        ...

    def get_config_schema(self) -> list:
        """Declare config fields for hermes memory setup."""
        return [
            {
                "key": "api_key",
                "description": "My Provider API key",
                "secret": True,
                "required": True,
                "env_var": "MY_API_KEY",
                "url": "https://my-provider.com/keys",
            },
            {
                "key": "region",
                "description": "Server region",
                "default": "us-east",
                "choices": ["us-east", "eu-west", "ap-south"],
            },
        ]

    def save_config(self, values: dict, hermes_home: str) -> None:
        """Write non-secret config to native location."""
        config_path = Path(hermes_home) / "my-provider.json"
        config_path.write_text(json.dumps(values, indent=2))

def register(ctx) -> None:
    ctx.register_memory_provider(MyMemoryProvider())
```

## Optional Hook Methods

| Method | When Called |
|--------|-----------|
| `system_prompt_block()` | System prompt assembly |
| `prefetch(query, *, session_id="")` | Before each API call |
| `queue_prefetch(query)` | After each turn (pre-warm) |
| `sync_turn(user, assistant, *, session_id="", messages=None)` | After each turn |
| `on_session_end(messages)` | Conversation ends |
| `on_pre_compress(messages)` | Before context compression |
| `on_memory_write(action, target, content)` | On built-in memory writes |
| `shutdown()` | Process exit |

## Threading Contract

`sync_turn()` **MUST** be non-blocking:

```python
def sync_turn(self, user_content, assistant_content, *, session_id="", messages=None):
    def _sync():
        try:
            self._api.ingest(user_content, assistant_content, session_id=session_id)
        except Exception as e:
            logger.warning("Sync failed: %s", e)

    if self._sync_thread and self._sync_thread.is_alive():
        self._sync_thread.join(timeout=5.0)
    self._sync_thread = threading.Thread(target=_sync, daemon=True)
    self._sync_thread.start()
```

## Profile Isolation (CRITICAL)

```python
# CORRECT — profile-scoped
from hermes_constants import get_hermes_home
data_dir = get_hermes_home() / "my-provider"

# WRONG — breaks profile isolation
data_dir = Path("~/.hermes/my-provider").expanduser()
```

## Single Provider Rule

Only ONE external memory provider can be active at a time. Second registration rejected with warning.

## Adding CLI Commands

```python
# plugins/memory/my-provider/cli.py
def my_command(args):
    sub = getattr(args, "my_command", None)
    if sub == "status":
        print("Provider is active and connected.")

def register_cli(subparser) -> None:
    subs = subparser.add_subparsers(dest="my_command")
    subs.add_parser("status", help="Show provider status")
    subparser.set_defaults(func=my_command)
```

Commands only appear when provider is the active `memory.provider`.

---

# 22. Adding Custom Tools

## 2-File Process

1. `tools/your_tool.py` — handler, schema, check function, registration
2. `toolsets.py` — add tool name to `_HERMES_CORE_TOOLS` or new toolset

## Tool File Template

```python
# tools/weather_tool.py
import json
import os
import logging

logger = logging.getLogger(__name__)

def check_weather_requirements() -> bool:
    return bool(os.getenv("WEATHER_API_KEY"))

def weather_tool(location: str, units: str = "metric") -> str:
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return json.dumps({"error": "WEATHER_API_KEY not configured"})
    try:
        # ... call weather API ...
        return json.dumps({"location": location, "temp": 22, "units": units})
    except Exception as e:
        return json.dumps({"error": str(e)})

WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather for a location.",
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "City name or coordinates (e.g. 'London' or '51.5,-0.1')"
            },
            "units": {
                "type": "string",
                "enum": ["metric", "imperial"],
                "description": "Temperature units (default: metric)",
                "default": "metric"
            }
        },
        "required": ["location"]
    }
}

from tools.registry import registry

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool(
        location=args.get("location", ""),
        units=args.get("units", "metric")),
    check_fn=check_weather_requirements,
    requires_env=["WEATHER_API_KEY"],
)
```

## Critical Rules

- Handlers **MUST** return a JSON string (`json.dumps(...)`)
- Errors **MUST** be returned as `{"error": "message"}`, never raised
- `check_fn` returning `False` silently excludes the tool
- Handler receives `(args: dict, **kwargs)` where args is LLM's tool call arguments

## Async Handlers

```python
async def weather_tool_async(location: str) -> str:
    async with aiohttp.ClientSession() as session:
        ...
    return json.dumps(result)

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool_async(args.get("location", "")),
    check_fn=check_weather_requirements,
    is_async=True,   # Registry handles async bridging automatically
)
```

## Tools Needing task_id

```python
def _handle_weather(args, **kw):
    task_id = kw.get("task_id")
    return weather_tool(args.get("location", ""), task_id=task_id)
```

## Adding to OPTIONAL_ENV_VARS

```python
# hermes_cli/config.py
OPTIONAL_ENV_VARS = {
    ...
    "WEATHER_API_KEY": {
        "description": "Weather API key for weather lookup",
        "prompt": "Weather API key",
        "url": "https://weatherapi.com/",
        "tools": ["weather"],
        "password": True,
    },
}
```

## Checklist

- [ ] Tool file in `tools/` with handler, schema, check function, registration
- [ ] Added to toolset in `toolsets.py`
- [ ] Handler returns JSON strings; errors as `{"error": "..."}`
- [ ] Optional: API key added to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`
- [ ] Tested: `hermes chat -q "Use the weather tool for London"`

---

# 23. Programmatic Integration (ACP / TUI / API Server)

## Protocol Comparison

| Protocol | Transport | Best For |
|----------|-----------|---------|
| **ACP** | JSON-RPC over stdio | IDE clients (VS Code, Zed, JetBrains) |
| **TUI Gateway** | JSON-RPC over stdio / WebSocket | Custom hosts, full feature control |
| **API Server** | HTTP + SSE | OpenAI-compatible frontends, language-agnostic |

## ACP (Agent Client Protocol)

```bash
hermes acp              # serve ACP on stdio
hermes acp --bootstrap  # print IDE install snippet
```

Source: `acp_adapter/`

Capabilities: session creation, prompt submission, streaming chunks, tool-call events, permission requests, session fork, cancel, authentication.

## TUI Gateway JSON-RPC

Source: `tui_gateway/server.py`

### Method Catalog

```
prompt.submit           prompt.background       session.steer
session.create          session.list            session.active_list
session.activate        session.close           session.interrupt
session.history         session.compress        session.branch
session.title           session.usage           session.status
clarify.respond         sudo.respond            secret.respond
approval.respond        config.set / config.get commands.catalog
command.resolve         command.dispatch        cli.exec
reload.mcp              reload.env              process.stop
delegation.status       subagent.interrupt
terminal.resize         clipboard.paste         image.attach
```

### Events Streamed Back

```
message.delta           message.complete        tool.start
tool.progress           tool.complete           approval.request
clarify.request         sudo.request            secret.request
gateway.ready           (session lifecycle events)
```

## OpenAI-Compatible API Server

Source: `gateway/platforms/api_server.py`

```
POST /v1/chat/completions        OpenAI Chat Completions (SSE streaming)
POST /v1/responses               OpenAI Responses API (stateful)
POST /v1/runs                    Start a run → run_id (202)
GET  /v1/runs/{id}               Run status
GET  /v1/runs/{id}/events        SSE stream of lifecycle events
POST /v1/runs/{id}/approval      Resolve pending approval
POST /v1/runs/{id}/stop          Interrupt the run
GET  /v1/capabilities            Machine-readable feature flags
GET  /v1/models                  Lists hermes-agent
GET  /health, /health/detailed
```

Headers: `X-Hermes-Session-Id`, `X-Hermes-Session-Key`

## Python In-Process Embed

```python
from run_agent import AIAgent

agent = AIAgent(
    model="anthropic/claude-opus-4.6",
    provider="openrouter",
)
response = agent.chat("Fix the bug in main.py")

# Or full interface:
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    task_id="task_abc123"
)
```

## Model Hot-Swapping (All Surfaces)

```bash
# CLI/TUI:
/model claude-sonnet-4
/model openrouter:anthropic/claude-sonnet-4.6

# TUI gateway RPC:
command.dispatch → {"command": "/model claude-sonnet-4"}

# API server:
POST /v1/chat/completions with {"model": "claude-sonnet-4"}
# or header: X-Hermes-Model
```

---

# 24. Slash Commands Reference

All slash commands flow through `hermes_cli/commands.py` (`COMMAND_REGISTRY`).

## Core Session Commands

| Command | Description |
|---------|-------------|
| `/new` | Start a new session (resets conversation) |
| `/resume [title\|id]` | Resume a previous session |
| `/stop` | Interrupt current agent turn |
| `/status` | Show current agent status |
| `/history` | Show current session history |
| `/queue` | Show pending queued messages |
| `/clear` | Clear conversation display |

## Model & Provider

| Command | Description |
|---------|-------------|
| `/model [provider:model]` | Switch model mid-session |
| `/fallback` | Show or configure fallback model |
| `/providers` | List configured providers |

## Memory & Context

| Command | Description |
|---------|-------------|
| `/memory` | Show current memory state |
| `/memory edit` | Edit MEMORY.md directly |
| `/compress` | Manually trigger context compression |

## Skills

| Command | Description |
|---------|-------------|
| `/skills` | Open skills manager |
| `/skills browse` | Browse available skills |
| `/skills install <name>` | Install a skill |
| `/skills enable <name>` | Enable a skill |
| `/skills disable <name>` | Disable a skill |

## Tools

| Command | Description |
|---------|-------------|
| `/tools list` | List active tools |
| `/tools enable <toolset>` | Enable a toolset |
| `/tools disable <toolset>` | Disable a toolset |

## Browser

| Command | Description |
|---------|-------------|
| `/browser connect [url]` | Connect to Chrome DevTools Protocol endpoint |
| `/browser disconnect` | Disconnect from browser |
| `/browser status` | Show browser connection status |

## Cron & Suggestions

| Command | Description |
|---------|-------------|
| `/cron list` | List cron jobs |
| `/suggestions` | List pending automation suggestions |
| `/suggestions accept N` | Accept suggestion N (creates cron job) |
| `/suggestions dismiss N` | Dismiss suggestion N permanently |
| `/suggestions catalog` | Add curated starter automations |

## Approval Flow (Gateway)

| Command | Description |
|---------|-------------|
| `/approve` | Approve pending dangerous command |
| `/deny` | Deny pending dangerous command |
| `/pair` | Generate pairing code for new user |

## MCP

| Command | Description |
|---------|-------------|
| `/reload-mcp` | Reload MCP server connections after config change |

## Debug

| Command | Description |
|---------|-------------|
| `/debug` | Toggle debug output |
| `/config show` | Show current configuration |
| `/config get <key>` | Get config value |
| `/config set <key> <value>` | Set config value |

---

# 25. Security Model

## Secret Storage

- `~/.hermes/.env` — all API keys and secrets
- NEVER put secrets in `config.yaml`
- `config.yaml` is safe to commit (profile-specific settings only)
- The LLM model NEVER has access to raw secret values

## Sandbox Isolation

- `execute_code` runs in an isolated Python environment
- `terminal` supports remote backends (Docker, SSH, Modal, Daytona, Singularity)
- Env var passthrough is explicitly opt-in per-skill or per-config

## Dangerous Command Approval

`DANGEROUS_PATTERNS` in `tools/approval.py`:
- `rm -rf` — recursive deletes
- `mkfs`, `dd` — filesystem formatting
- `DROP TABLE`, `DELETE FROM` without `WHERE` — SQL destructive ops
- `> /etc/` — system config overwrites
- `systemctl stop` — service manipulation
- `curl | sh` — remote code execution
- Fork bombs, process kills

## Context File Security Scanning

All context files (SOUL.md, AGENTS.md, project context) are scanned for:
- Invisible unicode characters
- "Ignore previous instructions" patterns
- Credential exfiltration attempts

## Skill Security (Hub-Installed)

Scans for:
- Data exfiltration patterns
- Prompt injection attempts
- Destructive commands
- Shell injection

Dangerous verdicts remain blocked regardless of `--force`.

## Profile Isolation

Each profile is fully isolated:
- Separate config, secrets, memory, sessions
- Separate gateway process with its own PID file
- No cross-profile data leakage

## MCP Security

- `ssl_verify: false` explicitly disables certificate verification (do not use in production)
- mTLS supported for mutual authentication
- OAuth tokens stored in `~/.hermes/mcp-tokens/<server>.json`
- Tool filtering (`include`/`exclude`) limits exposure surface

## Network Exposure

- **NEVER** use Tailscale Funnel
- Tailnet-only for private deployments
- Gateway runs on local ports
- API server is local-only by default

---

# 26. File Dependency Chain & Import Order

```
tools/registry.py     ← No deps — imported by all tool files
       ↑
tools/*.py            ← Each calls registry.register() at import time
       ↑
model_tools.py        ← Imports tools/registry + triggers discovery
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

Tool registration happens at **import time**, before any agent instance is created. Any `tools/*.py` file with a top-level `registry.register()` call is auto-discovered.

## System Prompt Build Order

```
hermes_constants.get_hermes_home()
       ↓
agent/prompt_builder.py
  → load_soul_md()                    # SOUL.md → stable tier (identity)
  → build_tool_guidance()             # stable tier (tool behavior)
  → build_skills_prompt()             # stable tier (skills index)
  → load_system_message()             # context tier (override)
  → build_context_files_prompt()      # context tier (project files)
  → load_memory_snapshot()            # volatile tier (MEMORY.md)
  → load_user_profile()               # volatile tier (USER.md)
  → load_external_memory_block()      # volatile tier (provider)
  → build_timestamp_line()            # volatile tier (time/session)
```

## Config Load Order

```
hermes_cli/config.py (DEFAULT_CONFIG)
       ↓
~/.hermes/config.yaml (user overrides)
       ↓
~/.hermes/.env (secrets)
       ↓
Environment variables (shell overrides)
       ↓
CLI flags (highest priority)
```

---

# 27. Key Constants & Defaults Reference

## Agent Defaults

| Constant | Default | Location |
|----------|---------|----------|
| `max_turns` | 90 | `run_agent.py` |
| `delegation.max_iterations` | 50 | config |
| `compress_threshold` | 0.50 | `compression.threshold` |
| `compress_target_ratio` | 0.20 | `compression.target_ratio` |
| `compress_protect_last_n` | 20 | `compression.protect_last_n` |
| `compress_protect_first_n` | 3 | Hardcoded in `context_compressor.py` |
| Summary ratio | 0.20 (`_SUMMARY_RATIO`) | `context_compressor.py` |
| Min summary tokens | 2,000 | `context_compressor.py` |
| Max summary tokens formula | `min(context_length × 0.05, 12,000)` | `context_compressor.py` |
| Gateway hygiene threshold | 0.85 | `gateway/run.py` |
| Tool result prune threshold | 200 chars | `context_compressor.py` |

## SQLite Write Contention

| Constant | Value |
|----------|-------|
| `_WRITE_MAX_RETRIES` | 15 |
| `_WRITE_RETRY_MIN_S` | 0.020 (20ms) |
| `_WRITE_RETRY_MAX_S` | 0.150 (150ms) |
| `_CHECKPOINT_EVERY_N_WRITES` | 50 |
| SQLite timeout | 1 second |

## Prompt Size Limits

| Limit | Value |
|-------|-------|
| `SOUL.md` max chars | 20,000 |
| Context file max chars | 20,000 |
| Context file truncation ratio | 70/20 head/tail |
| Inline shell snippet output cap | 4,000 chars |
| Old tool result prune size | >200 chars (outside tail) |

## Cron Defaults

| Constant | Default |
|----------|---------|
| Tick interval | 60 seconds |
| Script timeout | 120 seconds |
| `cron.wrap_response` | `true` |

## Skill Template Variables

| Token | Value |
|-------|-------|
| `${HERMES_SKILL_DIR}` | Absolute path to skill directory |
| `${HERMES_SESSION_ID}` | Active session ID |

## File Paths

| Path | Content |
|------|---------|
| `~/.hermes/config.yaml` | All configuration (SSOT) |
| `~/.hermes/.env` | Secrets and API keys |
| `~/.hermes/SOUL.md` | Agent identity |
| `~/.hermes/MEMORY.md` | Persistent agent memory |
| `~/.hermes/USER.md` | User profile |
| `~/.hermes/state.db` | SQLite session storage |
| `~/.hermes/cron/jobs.json` | Cron job definitions |
| `~/.hermes/gateway.pid` | Gateway process PID |
| `~/.hermes/mcp-tokens/<server>.json` | MCP OAuth tokens |
| `~/.hermes/skills/` | User custom skills |
| `~/.hermes/plugins/` | User plugins |
| `~/.hermes/hooks/` | Gateway hooks |

---

*End of Hermes_Architecture.md*
*Source: NousResearch/hermes-agent GitHub repo + https://hermes-agent.nousresearch.com/docs/*
*Compiled: 2026-06-14 — offline reference only*
