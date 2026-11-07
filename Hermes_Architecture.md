# Hermes Agent — Master Architecture Reference
## `Hermes_Architecture.md`
### Version sourced from: NousResearch/hermes-agent (GitHub) + https://hermes-agent.nousresearch.com/docs/
### Compiled: 2026-06-14 | Updated: 2026-11-06 | Docs-only reference for offline agentic development
### Synced upstream SHA: `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (main, observed 2026-11-06) | Latest release at sync: v0.23.0 (published 2026-11-05)

> Sources: [Hermes Agent documentation home](https://hermes-agent.nousresearch.com/docs/) · [upstream release v0.23.0 notes](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0) · [README.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/README.md)

**How to read the citations.** Every section and subsection carries a `> Sources:` line. Docs links point at the official documentation (`hermes-agent.nousresearch.com/docs/...`, which is rendered from `website/docs/**` in the upstream repo). Repo links are pinned to the synced commit SHA above, so they are immutable. A source marked `[community/secondary]` denotes an upstream issue or merged PR (change context, not a behavioral spec) or a third-party page; those appear only in [Change context & community sources](#change-context--community-sources-secondary) and are never the sole support for a section. The v0.23.0 release page is upstream's own release note, cited only for dated change context and not as a behavioral spec.

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
10. [Built-in Tools Reference](#10-built-in-tools-reference)
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

Unnumbered front matter: [What changed since 2026-06-21](#what-changed-since-2026-06-21) · [Summary of material changes](#summary-of-material-changes) · [Change context & community sources](#change-context--community-sources-secondary) · [Verification limits](#verification-limits)

---

# What changed since 2026-06-21

> Sources: [Hermes Agent documentation home](https://hermes-agent.nousresearch.com/docs/) · [upstream release v0.23.0 notes](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0)

This file was last revised on 2026-06-21 and rebuilt on 2026-11-06 against upstream docs and source at the SHA in the header. The list below is a dated summary; each row points to the numbered section holding the verified detail and its citations. "Then" means the 2026-06-21 text of this file; "Now" means what the docs/source at the synced SHA state. Nothing here was executed.

## Summary of material changes

> Sources: [Gateway Session Lifecycle](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-session-lifecycle) · [Multiplexing Gateway](https://hermes-agent.nousresearch.com/docs/developer-guide/multiplexing-gateway) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [gateway/run.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run.py)

| Area | Then (2026-06-21 text) | Now (docs/source at the synced SHA) | See |
|------|------------------------|-------------------------------------|-----|
| Session ownership | Each surface (CLI, gateway, ACP, ...) was described as its own entry into `AIAgent` | The docs describe a gateway as the canonical session writer, with CLI/TUI/Desktop/ACP/API server/cron/bot chats attaching as clients | 1, 18, 23 |
| Webapp | Not covered | A Desktop-workspace-in-a-browser listener served by the gateway; documented as off until `webapp.enabled` is set | 3, 18 |
| Profiles | Isolated profiles, one gateway PID each | Default multiplexing gateway with per-profile routes, `hermes gateway attach`, profile distributions | 5, 18 |
| Managed / remote config | Not covered | Managed scope and remote-config pages describe a policy/config layer on top of `config.yaml` | 3 |
| Tool inventory | "~71 tools", ten `browser_*` tools | No count is asserted in this file; the old `browser_*` set was removed in favor of `browser_exec` (and CDP/dialog/vault tools) per the tools reference | 10, 11 |
| Tool-output budgets | Described as a 2026-06-21 addition with its own subsection | Constants still live in `tools/budget_config.py`; the defaults are listed in the constants table rather than as a separate feature section | 27 |
| Delegation | Single built-in `delegate_task` | Pluggable delegation backends; official `advanced-delegation` plugin; subagent lifecycle API | 9, 20 |
| Context management | Compressor plus 2026-06-21 flags | In-place compaction default, prepared compaction (opt-in), micro-compaction (opt-in), pluggable context engines | 14 |
| Memory files | `~/.hermes/MEMORY.md`, `USER.md` | Files live under the profile's `memories/` directory | 2, 13 |
| Session DB | Single `hermes_state.py` and retry counts | Facade over several `hermes_state_*` modules, schema version in `hermes_state_common.py`, time-budgeted write contention, recovery guide | 15 |
| MCP | `mcp_<server>_<tool>`, 120 s timeout, `hermes mcp serve` | `mcp__<server>__<tool>` naming per source, 300 s default tool timeout, `hermes mcp-serve` now a plugin, price/spend-limit keys for paid tools | 16 |
| Providers | Static provider list; single-shot fallback | Provider plugins under `plugins/model-providers/`; turn-scoped fallback; credential pools and vault | 17 |
| Cron | 120 s script timeout, PID/tick notes | 3600 s default script timeout, `hermes cron cancel`, catch-up/grace policy, Chronos managed-cron contract | 19 |
| Plugins | Three discovery sources, few types | Opt-in model, catalog tiers (official, verified, community), `hermes plugins update`, opt-in auto-update, public API contract | 20 |
| Security | `tools/approval.py` patterns | Hardline blocklist and patterns in `tools/approval_detection.py`; egress proxy; secret-source plugins; trust-model statement in `SECURITY.md` | 25 |
| Slash commands | Core list | `/rewind`, `/model reset` and other additions per the reference page | 24 |
| Default model | Examples used Opus-class IDs | Source constant `PREFERRED_SILENT_DEFAULT_MODEL` is `z-ai/glm-5.3` (used only when no model was ever chosen) | 17 |

## Items removed because they could not be verified

> Sources: [Hermes Agent documentation home](https://hermes-agent.nousresearch.com/docs/)

Claims from the earlier text that no current doc or source file supports were dropped rather than carried forward: exact tool/toolset/test counts, the old `/pair` command, a `HERMES_PROFILE` selector variable, the Tailscale Funnel rule, and several 2026-06-21 changelog bullets with PR numbers. The per-section reasons are kept in the verification log alongside the working files, not in this document.

## Change context & community sources (secondary)

> Sources: [upstream release v0.23.0 notes](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0) · [Hermes Agent documentation home](https://hermes-agent.nousresearch.com/docs/) · [CONTRIBUTING.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/CONTRIBUTING.md)

**Authority note.** Everything in this section is **[community/secondary]** change context: merged pull requests, issues and a hub listing. These items show when and why behavior changed, but they are not the specification. The numbered sections cite official docs and SHA-pinned source only. PR/issue titles below are quoted from the GitHub pages as fetched on 2026-11-06 (state `MERGED` or as noted; no code from them was run).

| Date merged | Merged PR [community/secondary] | Relevant sections |
|-------------|---------------------------------|-------------------|
| 2026-10-14 | [#106742](https://github.com/NousResearch/hermes-agent/pull/106742) Unified gateway runtime: one gateway owns every local session (classic CLI, TUI, Desktop, API, ACP, bots, cron) | 1, 18, 23 |
| 2026-10-17 | [#140334](https://github.com/NousResearch/hermes-agent/pull/140334) feat(gateway): steer and edit a canonical queued prompt from Desktop | 18, 23 |
| 2026-10-25 | [#141864](https://github.com/NousResearch/hermes-agent/pull/141864) feat(webapp): serve the Desktop workspace from the gateway | 3, 18 |
| 2026-10-26 | [#144684](https://github.com/NousResearch/hermes-agent/pull/144684) feat(compression): opt-in prepared compaction | 14 |
| 2026-10-28 | [#145806](https://github.com/NousResearch/hermes-agent/pull/145806) Manual /compress on gateway sessions uses the configured context engine again | 14, 18 |
| 2026-11-05 | [#148924](https://github.com/NousResearch/hermes-agent/pull/148924) fix(gateway): side tasks read the profile's config.yaml in existing chats | 5, 18 |
| 2026-11-06 | [#142007](https://github.com/NousResearch/hermes-agent/pull/142007) feat(plugins): pass a compression-stable conversation_id to plugin tools and tool hooks | 14, 20 |
| 2026-11-06 | [#149763](https://github.com/NousResearch/hermes-agent/pull/149763) fix(memory): let subagents compact and withhold the prune under checkpoint_required | 13, 14 |

Issues [community/secondary]: [#24770](https://github.com/NousResearch/hermes-agent/issues/24770) (open at observation) - feature discussion on native multi-provider memory routing, relevant to Section 13's single-external-provider description; [#7816](https://github.com/NousResearch/hermes-agent/issues/7816) (closed) - original proposal for skill lifecycle management, which the docs' Curator page links as its origin (Section 12).

**Hermes Hub.** The "Hermes Hub" the upstream Desktop app embeds is the Skills Hub picker on the official docs site, `https://hermes-agent.nousresearch.com/docs/skills` (HTTP 200 on 2026-11-06), with the GitHub Pages mirror `https://nousresearch.github.io` named as fallback origin in [apps/desktop/electron/hub-iframe-policy.ts](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/apps/desktop/electron/hub-iframe-policy.ts). The static catalog pages [Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) and [Optional Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/optional-skills-catalog) are official. Third-party sites named "HermesHub" (`hermeshub.xyz`, `hermeshub.com`) were **unavailable or non-resolving** when probed on 2026-11-06 and are not cited; this is a stated limitation. Community plugins listed in the plugin catalog are covered in Section 20 under the docs' trust tiers.

## Verification limits

> Sources: [Hermes Agent documentation home](https://hermes-agent.nousresearch.com/docs/) · [SECURITY.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/SECURITY.md)

- Docs-only sync. Every statement comes from reading upstream documentation and source at the pinned SHA. No Hermes command, gateway, plugin, tool or API was run, so nothing here is a claim about observed runtime behavior.
- Where two official pages disagree (examples: the default for `agent.max_turns`, MCP tool-name prefix form, the order in which gateway authorization is evaluated, whether `30m` is recurring or one-shot in the cron schedule table), the section says so and states which source it followed (normally source code over prose).
- Items the docs describe as opt-in, off by default, or experimental are labeled as such in the section text. If a feature is not labeled, that means the docs did not label it, not that it was tested.
- Counts of tools, toolsets, platforms, skills, plugins or tests are intentionally not asserted. Use the reference pages and CLI listing commands named in Sections 10, 11, 12 and 20 to enumerate them for your installed version.
- The upstream `main` branch moves. Values in this file are correct for the SHA in the header; re-verify before relying on them for a newer release.
- Community/secondary sources (merged PRs, issues, the Hermes Hub listing) are confined to the [Change context & community sources](#change-context--community-sources-secondary) section; no numbered section depends on them. Attempts to reach the third-party `hermeshub.xyz` and `hermeshub.com` domains on 2026-11-06 failed (HTTP 402 and a redirect to a host that did not resolve), so no claim here rests on those sites.

---

# 1. System Overview & Design Principles

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [developer-guide/architecture.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/architecture.md) · [run_agent.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/run_agent.py)

Everything below is derived from the upstream docs and source at commit `9f23c4311a4c4837078081db4f3cd8deab23c0fe`. Where this guide says "the docs state", it reports what the documentation says, not behavior that was executed.

## What Hermes Agent Is

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [developer-guide/architecture.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/architecture.md)

Hermes Agent (Nous Research) is a tool-using LLM agent with several front ends. The architecture page describes it as a set of subsystems around one agent class:

- **Agent loop** - `AIAgent`, exposed by the `run_agent.py` facade; the loop itself lives in `agent/conversation_loop.py` and `agent/turn_*.py`. It handles provider selection, prompt construction, tool execution, retries, fallback, callbacks, compression and persistence.
- **Prompt system** - ordered system-prompt tiers (`stable`, `context`, `volatile`) built by `agent/system_prompt.py` and `agent/prompt_builder.py`, plus prompt caching and context compression.
- **Provider resolution** - one runtime resolver shared by CLI, gateway, cron, ACP and auxiliary calls.
- **Tool system** - a central registry (`tools/registry.py`); each tool file self-registers at import time. Terminal tools run on pluggable backends.
- **Session persistence** - SQLite with FTS5 search and parent/child lineage across compressions.
- **Messaging gateway** - a long-running process with platform adapters, session routing, authorization, slash commands, hooks, cron ticking and background maintenance.
- **Plugins, cron, ACP, trajectories** - memory-provider and context-engine plugins, first-class scheduled agent tasks, an editor-native agent protocol, and ShareGPT-format trajectory generation.

The docs quote tool, toolset, backend, platform and test counts on the architecture page, but those numbers drift. To enumerate the current set, use the generated reference pages ([Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference), [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference)) or the interactive `hermes tools` command, rather than trusting any number printed in prose.

## Architecture Diagram: Entry Points to Storage and Tools

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions) · [developer-guide/session-storage.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/session-storage.md)

The architecture page draws three layers: entry points, the `AIAgent` core, and (below it) session storage and tool backends. The session-storage page adds the ownership rule that matters most in this release line: for a profile, **the gateway is the only writer of canonical sessions**, and the interactive surfaces attach to it as clients.

```text
 ENTRY POINTS (architecture page)
 +------------+  +-----------+  +-----+  +------------+  +------------+  +----------------+
 | CLI / TUI  |  | Messaging |  | ACP |  | API server |  | Batch      |  | Python library |
 | (hermes)   |  | platforms |  |     |  |            |  | runner     |  |                |
 +-----+------+  +-----+-----+  +--+--+  +-----+------+  +-----+------+  +--------+-------+
       |  clients of the profile's gateway (session-storage page):          |   direct
       |  hermes chat, TUI, Desktop (local), API server, ACP, cron,         |   callers
       |  Bot Chat, Kanban workers admit input over /api/ws                 |   of AIAgent
       v                                                                    |   (per the
 +-------------------------------------------------------------+            |   architecture
 | Gateway = session authority (one writer of canonical        |            |   page)
 | sessions for the profile): GatewayRunner, SessionStore,     |            |
 | authorization, slash commands, hooks, cron ticking          |            |
 +------------------------------+------------------------------+            |
                                v                                           v
 +--------------------------------------------------------------------------------------+
 | AIAgent (run_agent.py facade; loop in agent/conversation_loop.py + agent/turn_*.py)  |
 |   prompt builder      provider resolution        tool dispatch (model_tools.py)      |
 |   compression/caching  3 API modes               tool registry (tools/registry.py)   |
 +-------------------+--------------------------------------------+---------------------+
                     v                                            v
        +-------------------------+              +------------------------------------+
        | Session storage         |              | Tool backends                      |
        | SQLite + FTS5           |              | terminal backends, browser, web,   |
        | hermes_state.py (+      |              | file, vision, MCP (dynamic), ...   |
        | hermes_state_*.py)      |              | tools/ and tools/environments/     |
        | gateway/session.py      |              +------------------------------------+
        +-------------------------+
```

Notes on reading the diagram:

- The three API modes named on the architecture page are `chat_completions`, `codex_responses` and `anthropic_messages`.
- The classic CLI is documented as a *gateway client*: "`hermes --cli`, `hermes chat`, `hermes chat -q '...' -Q`, and top-level `hermes -z '...'` use the gateway authority" ([Classic gateway client](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-classic-cli)). The same page lists parity limits that are still outstanding, so it is described as an initial migration, not full parity.
- Fresh local CLI/TUI/GUI sessions can be created by the gateway listener without starting a separate agent runtime; the existing `GatewayRunner`/`TurnRunner` own execution, approval waiters, tool calls and storage ([Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions)).
- Out-of-process managed workers write through the owner (`RuntimeSessionStore` RPC); they do not open `state.db` for transcript writes themselves (Session Storage page).

### Data flow summaries

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [developer-guide/architecture.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/architecture.md)

The architecture page gives three canonical flows (names are the page's, simplified here):

| Flow | Path |
|---|---|
| CLI session | user input -> `HermesCLI.process_input()` -> `AIAgent.run_conversation()` -> prompt build -> provider resolution -> API call -> tool calls via `model_tools.handle_function_call()` (loop) -> display -> save to the session DB |
| Gateway message | platform event -> adapter -> `MessageEvent` -> `GatewayRunner._handle_message()` -> authorize user -> resolve session key -> create `AIAgent` with session history -> run -> deliver through the adapter |
| Cron job | scheduler tick -> load due jobs -> fresh `AIAgent` (no history) -> inject attached skills -> run job prompt -> deliver to target -> update job state and next run |

Because the gateway is the session authority for local surfaces (previous section), the CLI row is the page's high-level description of the agent path; the classic-CLI page documents that the process now hands off to the gateway before constructing a local agent.

## Design Principles

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [developer-guide/architecture.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/architecture.md) · [developer-guide/codebase-ownership.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/codebase-ownership.md)

As the Architecture page states them:

| Principle | What it means in practice (per the docs) |
|---|---|
| Prompt stability | The system prompt does not change mid-conversation; no cache-breaking mutations except explicit user actions such as `/model`. |
| Observable execution | Every tool call is visible to the user via callbacks; progress shows as a spinner in the CLI and as chat messages in the gateway. |
| Interruptible | API calls and tool execution can be cancelled mid-flight by user input or signals. |
| Platform-agnostic core | One `AIAgent` class serves CLI, gateway, ACP, batch and API server; platform differences live in the entry point, not the agent. |
| Loose coupling | Optional subsystems (MCP, plugins, memory providers, RL environments) use registry patterns and `check_fn` gating rather than hard dependencies. |
| Profile isolation | Each profile (`hermes -p <name>`) gets its own `HERMES_HOME`, config, memory, sessions and gateway state; multiple profiles can coexist. See section 5 for what the current multi-profile gateway does and does not isolate. |

Conventions from the Codebase Ownership Map that follow from the same design: changes stay inside their subsystem; a plugin that must edit core files is a design smell (widen the generic plugin surface instead); where two subsystems are involved the narrower one owns the change, because "the core is a narrow waist, and every addition there is paid for on every API call".

The architecture page also describes tool registration as import-time: `tools/registry.py` has no dependencies, each `tools/*.py` calls `registry.register()` at import, and `model_tools.py` triggers discovery before any agent instance exists.

## Where to Read Next

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership)

The architecture page's suggested reading order: Agent Loop Internals, Prompt Assembly, Provider Runtime Resolution, Adding Providers, Tools Runtime, Session Storage, Gateway Internals, Context Compression & Prompt Caching, ACP Internals. The Codebase Ownership Map is the lookup table from subsystem to source directory to docs entry point (agent core, prompt assembly, providers and transports, built-in tools, gateway, CLI, plugins, skills, cron, session storage, browser stack, egress firewall, ACP, Desktop app, TUI, docs site, tests).

---

# 2. Directory Structure

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [repo root tree](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/README.md)

Two different trees matter: the **source checkout** (this section's first half) and the **Hermes home** that holds a user's runtime data (second half). Line counts and test counts are deliberately omitted; they change constantly.

## Repository Top-Level Layout

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [developer-guide/codebase-ownership.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/codebase-ownership.md)

The list below comes from `git ls-tree` of the commit above (top-level entries), annotated with what the docs say. Entries marked "(not described in the docs)" were seen in the tree but the architecture and ownership pages do not explain them, so only their presence is reported.

| Path | Role |
|---|---|
| `run_agent.py` | `AIAgent` facade; the loop lives in `agent/conversation_loop.py` and `agent/turn_*.py` |
| `cli.py` | `HermesCLI` facade; mixins in `hermes_cli/cli_*_mixin.py` |
| `model_tools.py` | Tool discovery, schema collection, dispatch |
| `toolsets.py` | Tool groupings and platform presets |
| `hermes_state.py` + `hermes_state_*.py` | SQLite session/state database facade and its sibling modules (schema, FTS, search, compression, portability, gateway, ...). The state DB is no longer a single file. |
| `hermes_constants.py` | `HERMES_HOME` and profile-aware paths (`get_hermes_home()`) |
| `batch_runner.py` | Batch trajectory generation |
| `agent/` | Agent internals: prompt builder, context engine and compressor, prompt caching, auxiliary client, model metadata, Anthropic adapter, memory manager/provider ABC, trajectory helpers |
| `hermes_cli/` | CLI subcommands and setup; `main.py` is the `hermes` entry point; `config.py`/`config_defaults.py` hold `DEFAULT_CONFIG`; `commands.py` holds the slash-command registry; `auth.py`, `runtime_provider.py`, `plugins.py` (PluginManager) |
| `tools/` | Tool implementations (one file per tool family), `registry.py`, approvals, terminal orchestration, browser stack (`browser_use_cli.py` provides `browser_exec`), `mcp_tool.py`, `environments/` terminal backends |
| `gateway/` | Messaging gateway: `run.py` (`GatewayRunner` facade plus `run_*.py` phases), `session.py` (`SessionStore`), delivery, pairing, hooks, mirror, status; `platforms/` holds the core ingress adapters (API server, webhook, Microsoft Graph webhook) |
| `plugins/` | Bundled plugins. `plugins/platforms/` holds the bundled messaging-platform plugins; other subdirectories include `memory/`, `context_engine/`, `model-providers/`, `config_backends/` (remote config) and more |
| `acp_adapter/` | ACP server (VS Code / Zed / JetBrains) |
| `cron/` | Scheduler (`jobs.py`, `scheduler.py`) |
| `skills/` | Bundled skills, always available |
| `optional-skills/` | Official optional skills, installed explicitly |
| `ui-tui/`, `tui_gateway/` | TUI frontend and its gateway module (Codebase Ownership Map) |
| `apps/` | Contains `desktop/` (Desktop app, per the ownership map) plus other app packages (`bootstrap-installer`, `shared`, listed in the tree) |
| `website/` | Docusaurus documentation site; docs source lives in `website/docs/` |
| `tests/`, `tests-js/` | Test suites; tests mirror source (code in `tools/` is tested in `tests/tools/`) |
| `cli-config.yaml.example` | Annotated config template (see section 3) |
| `.env.example` | Annotated environment template (see section 4) |
| `providers/`, `pm/`, `hermes_platform/`, `native/`, `plugin-catalog/`, `optional-mcps/`, `web/`, `docker/`, `nix/`, `evals/`, `locales/`, `scripts/`, `contributors/`, `assets/` | Present in the tree (not described in the architecture or ownership pages beyond what other sections of this guide cite) |
| `Dockerfile`, `docker-compose*.yml`, `flake.nix`, `pyproject.toml`, `setup-hermes.sh`, `setup-hermes.ps1`, `uv.lock`, `AGENTS.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SOUL.md` | Packaging, install and project metadata files |

Mapping from the ownership page (subsystem -> source -> docs entry): agent core -> `agent/`, `run_agent.py`; model providers and transports -> `agent/transports/`, `plugins/model-providers/`, `hermes_cli/models.py`; built-in tools -> `tools/`; gateway -> `gateway/`, `plugins/platforms/`; CLI -> `hermes_cli/`; session storage -> `hermes_state.py`, `hermes_state_*.py`; browser stack -> `tools/browser_use_cli.py`, `tools/browser_session*.py`, `tools/browser_launcher.py`, `tools/browser_supervisor.py`, `tools/browser_cdp_tool.py`; egress firewall -> `agent/proxy_sources/iron_proxy.py`; ACP -> `acp_adapter/`; Desktop -> `apps/desktop/`; TUI -> `ui-tui/`, `tui_gateway/`.

## HERMES_HOME Layout

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Which File Does What?](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what) · [Nix Setup](https://hermes-agent.nousresearch.com/docs/getting-started/nix-setup) · [user-guide/configuration.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/configuration.md)

The Configuration page's own tree is intentionally short:

```text
~/.hermes/
+-- config.yaml     # Settings (model, terminal, TTS, compression, etc.)
+-- .env            # API keys and secrets
+-- auth.json       # OAuth provider credentials (Nous Portal, etc.)
+-- SOUL.md         # Primary agent identity (slot #1 in system prompt)
+-- memories/       # Persistent memory (MEMORY.md, USER.md)
+-- skills/         # Agent-created skills (managed via skill_manage tool)
+-- cron/           # Scheduled jobs
+-- sessions/       # Gateway sessions
+-- logs/           # Logs (errors.log, gateway.log; secrets auto-redacted)
```

Other entries that individual docs pages mention (the list is a union across pages, not a complete listing; "per page" says which one):

| Entry (under the Hermes home) | What the docs say |
|---|---|
| `state.db` (+ `-wal`/`-shm` in WAL mode) | SQLite session store; path is always `get_hermes_home() / "state.db"` (Session Storage) |
| `memories/MEMORY.md`, `memories/USER.md` | Curated memory files; **not** at the home root (Which File Does What?). The old layout that put `MEMORY.md`/`USER.md` at the root is obsolete. |
| `gateway.pid`, `mcp-tokens/` | Shown in the Nix setup layout (Nix Setup) |
| `plugins/` | User-installed plugins (Nix Home Manager layout; profile clone docs) |
| `profiles/<name>/` | Named profiles, each a full Hermes home (see section 5) |
| `active_profile` (in the default root) | Sticky profile selection read at CLI bootstrap (Session Storage, bootstrap description) |
| `profile.yaml` | Profile metadata such as description or display name (Profiles, Profile Commands) |
| `backups/`, `backups/config/` | Full backup zips and point-in-time `config.yaml` copies (Configuration, Update Behavior) |
| `state-snapshots/` | Quick pre-update snapshots of critical state (Configuration, Update Behavior) |
| `checkpoints/` | Excluded from `--clone-all` along with sessions and `state.db` (Profiles) |
| `home/` | Persistent tool-state HOME used by container backends (Profiles, "How it works") |
| `attachments/` | Default location for TUI/Desktop session attachments when `attachments.storage: hermes-home` (cli-config.yaml.example) |
| `gateway.parked`, `gateway_state.json` | Multiplexed-gateway markers/state (see section 5) |

On Windows the default home is `%LOCALAPPDATA%\hermes`; the official Docker image uses `/opt/data` (Environment Variables reference, `HERMES_HOME` row). Always resolve the home through `get_hermes_home()` rather than hard-coding `~/.hermes`: it checks a context-local override first, then `HERMES_HOME`, then the platform default.

---

# 3. Configuration — config.yaml SSOT

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Managed Scope](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope) · [Remote Config](https://hermes-agent.nousresearch.com/docs/user-guide/remote-config) · [Config Backend Seam](https://hermes-agent.nousresearch.com/docs/developer-guide/config-backend) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

`config.yaml` in the Hermes home is the primary store for **non-secret** settings; secrets go in `.env` (section 4). The built-in defaults are the `DEFAULT_CONFIG` dict in `hermes_cli/config_defaults.py`; the annotated template is `cli-config.yaml.example` at the repo root. Neither the number of keys nor their order is stable, so this section covers the structure and the keys the docs call out. For the exhaustive per-section reference, use the Configuration page (a long page; sections such as Terminal Backend, Compression, Auxiliary Models, Display, TTS/STT, Security, Delegation, Webapp and Dashboard each have their own heading).

## Managing and Inspecting Config

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [user-guide/configuration.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/configuration.md)

```bash
hermes config              # View current configuration
hermes config edit         # Open config.yaml in your editor
hermes config get KEY      # Print a resolved value
hermes config set KEY VAL  # Set a specific value
hermes config unset KEY    # Remove a user-set value
hermes config check        # Check for missing options and stale saved selections
hermes config migrate      # Interactively add missing options
```

Routing rules the docs state for `hermes config set`:

- Every `UPPER_SNAKE` name (for example `OPENROUTER_API_KEY`, `HERMES_TIMEZONE`) is an environment variable and is saved to `.env`, never to `config.yaml`; dotted settings go to `config.yaml`. Names on the env writer's denylist (`HERMES_YOLO_MODE`, `PATH`, and others) are refused.
- A known key written under the wrong prefix is refused with a did-you-mean (use `--force` to write anyway). An unknown path under a known section is written together with a notice, because the schema alone cannot tell a typo from a runtime-read key.
- `hermes config get` on an unknown path prints the stored value plus a stderr notice that Hermes may not read it.

## Core Structure (trimmed from the example template)

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

The snippet is trimmed from `cli-config.yaml.example`; comments note where the example's illustrative value differs from `DEFAULT_CONFIG`. Do not copy `_config_version` into another config: the template says Hermes manages it.

```yaml
# _config_version is managed by Hermes (52 at this commit); do not copy it.

model:
  default: "anthropic/claude-opus-4.6"   # example value; DEFAULT_CONFIG ships model "" (choose with `hermes model` / `hermes setup`)
  provider: "auto"                        # "auto" detects from credentials; see the example for the provider list
  base_url: "https://openrouter.ai/api/v1"
  # api_key: ...                          # prefer .env, or a ${VAR} reference

# Fallback chain (top-level list; the older single "fallback_model" shape is still read)
# fallback_providers:
#   - provider: openrouter
#     model: anthropic/claude-sonnet-4

toolsets: [hermes-cli]                    # DEFAULT_CONFIG default
# custom_toolsets:                        # documented in the Toolsets Reference
#   data-science: [file, terminal, code_execution, web, vision]
# platform_toolsets:
#   telegram: [web, terminal, file, todo] # per-platform override of the built-in preset

terminal:
  backend: "local"                        # local | docker | ssh | modal | daytona | vercel_sandbox | plugin backends
  cwd: "."                                # "." = launch directory (local backend); set an absolute path for a fixed workspace
  timeout: 180
  home_mode: "auto"                       # auto | real | profile (HOME policy for tool subprocesses)
  # env_passthrough: [MY_CUSTOM_VAR]      # names forwarded into sandboxes (never provider keys)

agent:
  max_turns: 500                          # example value; DEFAULT_CONFIG is null = unlimited (the Configuration page says unlimited by default)

compression:
  enabled: true
  threshold: 0.50
  target_ratio: 0.20
  protect_last_n: 20
  # in_place: true                        # DEFAULT_CONFIG / docs: compact on the same session id
# summarizer model/provider live under auxiliary.compression.*

prompt_caching:
  cache_ttl: "5m"                         # "1h" for sessions with long pauses

memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  nudge_interval: 10
  # provider: ""                          # empty = built-in only; one external provider may be set

context:
  engine: "compressor"                    # DEFAULT_CONFIG; plugin context engines are single-select

skills:
  # external_dirs: [~/.agents/skills]     # read-only shared skill directories

# mcp_servers:                            # user-defined; examples in the template
#   time:
#     command: uvx
#     args: ["mcp-server-time"]

cron:
  catch_up_missed: true

gateway:
  # multiplex_profiles: true              # see section 5
  # standalone: false                     # per-profile opt-out, named profiles only

webapp:                                    # off until enabled; restart the gateway after changing
  enabled: false
  host: 127.0.0.1
  port: 9120
  public_url: ""

updates:
  check: true
  pre_update_backup: quick                # quick | full | off (DEFAULT_CONFIG; the example file shows false, a legacy boolean form)
  backup_keep: 5

database:
  journal_mode: "wal"                     # "delete" for filesystems where WAL is unsafe
runtime:
  nofile_soft_limit: 4096
  malloc_arena_max: 4
```

Differences from the pre-existing guide worth knowing (see the log for the full list): the model is configured under `model.*` (not top-level `model`/`provider` scalars); context compression defaults are 0.50/0.20/20 with `in_place` true; `cron.script_timeout_seconds` exists but `DEFAULT_CONFIG` ships 3600 (the old guide said 120); the CLI theme key is `display.skin` (a `theme` key exists under `dashboard`, not `display`); and the old guide's `plugins.enabled` and `mcp_discovery_timeout: 1.5` are not restated here beyond what the docs and `DEFAULT_CONFIG` show (`mcp_discovery_timeout` is 1.5 in `DEFAULT_CONFIG`).

### Webapp (browser-hosted Desktop)

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop) · [Docker](https://hermes-agent.nousresearch.com/docs/user-guide/docker) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions)

The gateway can serve the Desktop workspace in a browser on its own listener. The Configuration page states it is **off until you turn it on** (`webapp.enabled: false` default; `hermes webapp` opens it; the gateway reads `enabled` at start, so restart after changing it). Key behaviors per the docs:

- `host`: `127.0.0.1`, `::1` or `localhost` keep it on this machine; `0.0.0.0` or `::` expose it to other machines and require a dashboard sign-in provider, otherwise the gateway serves native clients only and `hermes webapp --status` says why.
- `port` defaults to `9120`; a standalone profile's gateway needs a different port than the default gateway.
- `public_url` is for a reverse proxy or tunnel; it turns sign-in on even for a loopback host. In a container, `host: 0.0.0.0` is needed (Docker page).
- Native clients (CLI, TUI, Desktop app) never sign in through the Webapp's listener.

## Configuration Precedence

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Managed Scope](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope) · [user-guide/managed-scope.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/managed-scope.md)

The Configuration page's order (highest first):

1. CLI arguments (for example `hermes chat --model ...`) - per invocation.
2. `config.yaml` - all non-secret settings.
3. `.env` - fallback for env vars, and **required** for secrets.
4. Built-in defaults.

Rule of thumb from the docs: secrets in `.env`, everything else in `config.yaml`; when both are set, `config.yaml` wins for non-secret settings. Individual documented env vars can still override a config key (examples in the Environment Variables reference: `HERMES_INFERENCE_MODEL` for the session model, `HERMES_OPENROUTER_CACHE`, and `GATEWAY_ALLOW_ALL_USERS` over `gateway.allow_all_users`), so check the row for any variable you rely on.

**Managed scope inverts this for the keys it pins.** An administrator-owned directory (default `/etc/hermes`, relocatable with `HERMES_MANAGED_DIR`) can hold a `config.yaml` and a `.env`. The Managed Scope page's tiers:

| Tier | config.yaml | .env |
|---|---|---|
| 1 (wins) | managed `config.yaml` | managed `.env` |
| 2 | user `config.yaml` | user `.env` |
| 3 | built-in defaults | pre-existing shell environment |

Merging is leaf-level (pinning `model.default` does not freeze the rest of `model.*`). Enforcement is filesystem permissions only; the managed `.env` is world-readable by design, so use it for shared non-sensitive values. `hermes config set` on a pinned key refuses and names the source; `hermes config` and `hermes doctor` show the managed source and pinned keys. The page lists v1 limits (no hard boundary against the agent itself, POSIX-first, no drop-in fragments, no signing, no MDM delivery). A package-manager-managed install is a separate mechanism that blocks all config mutation.

## Remote Config

> Sources: [Remote Config](https://hermes-agent.nousresearch.com/docs/user-guide/remote-config) · [Config Backend Seam](https://hermes-agent.nousresearch.com/docs/developer-guide/config-backend) · [user-guide/remote-config.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/remote-config.md) · [hermes_cli/config_backend.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_backend.py)

For fleet deployments, `HERMES_CONFIG_BACKEND=remote` replaces `config.yaml` with a config plane that layers global, tenant, group, owner, agent and finally the profile's own settings, with administrators able to lock keys. The backend is chosen once per process from the environment (or `.env`, or the managed `.env`), never from a config value, because the backend is what serves config. Required variables: `HERMES_CONFIG_INSTANCE_ID`; `HERMES_CONFIG_REMOTE_URL` has a documented default. Behaviors the page states:

- Startup fetches config and **fails closed**: about 30 seconds of retries, then the process exits; it never falls back to a local file or to defaults.
- No local config file or cache; a running process keeps its in-memory copy and polls every 5 minutes.
- Writes go only to the profile's own level, with changed keys only and compare-and-swap protection; locked keys are refused naming the locking level.
- Secret-shaped keys accept only `${VAR}` references, not literal secrets.
- The managed `/etc/hermes/config.yaml` is ignored in remote mode (the managed `.env` still applies); `hermes config edit`, profile clone/rename/distribution install, `hermes backup`/`import` and `gateway --config` are refused.

## How Config Is Read and Written

> Sources: [Config Backend Seam](https://hermes-agent.nousresearch.com/docs/developer-guide/config-backend) · [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [developer-guide/config-backend.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/config-backend.md) · [hermes_cli/config_backend.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_backend.py)

- **One seam.** Every read, stat, existence check and write of a profile's user `config.yaml` goes through `hermes_cli/config_backend.py` (helpers such as `read_config_doc`, `read_config_doc_readonly`, `config_version`, `write_config_document`, `write_config_key`, `write_config_keys`). CI lints (`scripts/check_config_yaml_writers.py`, `scripts/check_config_yaml_readers.py`) reject direct YAML dumps or reads of a config path.
- **Related keys write together.** A model switch's `model.*` keys go through `write_config_keys`, which a remote backend sends as one compare-and-swap write.
- **Caches** key on a change signature (`config_cache_version`: the stat tuple plus a content digest while the file is younger than a racy-mtime window), so an in-place rewrite within the same timestamp tick is still noticed.
- **Atomic writes.** `hermes_cli/config.py` provides `atomic_config_write` / `atomic_config_replace`; `save_config` and `set_config_value` are the usual entry points.
- **Overlay order.** With the file backend, the managed scope is an overlay on top of the user layer; `${VAR}` references are resolved after loading (next subsection).
- **Backups and recovery.** Point-in-time copies of `config.yaml` go to `backups/config/` (before setup rewrites, before migrations, whenever it parses, and when it fails to parse). If `config.yaml` is broken, Hermes serves the newest `good` copy instead of built-in defaults and warns on every start; the broken file is never modified (Configuration, Update Behavior).
- **Schema migrations.** `_config_version` records the schema; `hermes update` and `hermes config migrate` apply one-time migrations (for example legacy `compression.summary_*` keys moved to `auxiliary.compression.*`).
- **Which process reads what.** The old claim that the CLI and gateway read config through different loaders is not restated here: the docs do not document it, and `cli.py` / `hermes_cli/cli_config_load.py` still contain a CLI-specific loader (`load_cli_config`). Treat per-surface differences as an implementation detail.

### Environment-Variable References in config.yaml

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) · [user-guide/configuration.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/configuration.md)

Config values can reference environment variables with `${VAR_NAME}` (several per value are allowed; bare `$VAR` is not expanded). An unset variable keeps the placeholder text and logs a warning. `${env:VAR}` is accepted as an equivalent spelling; other inline SecretRef sources (`${file:...}`, `${vault:...}`) are not resolved: use the `secrets:` block instead, which injects values into the environment at startup. Under a multiplexed gateway, references in a profile's `config.yaml` resolve against **that profile's** `.env`, not the shared process environment.

```yaml
auxiliary:
  vision:
    api_key: ${GOOGLE_API_KEY}
delegation:
  api_key: ${DELEGATION_KEY}
```

## What This Section Does Not Cover

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Environment Variables](https://hermes-agent.nousresearch.com/docs/reference/environment-variables)

Per-key reference for terminal backends, auxiliary models, reasoning effort, TTS/STT, display, security, approvals, delegation, checkpoints, dashboard auth and gateway tuning lives on the Configuration page (one heading per section) and, for provider setup, in the provider integration docs. Later sections of this guide cover compression, memory, MCP and gateway settings in their own contexts.

---

# 4. Environment Variables & .env

> Sources: [Environment Variables](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) · [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) · [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [.env.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/.env.example) · [hermes_cli/env_loader.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/env_loader.py)

The Environment Variables reference opens with the contract: Hermes reads variables from the process environment and, for user-managed secrets, from `~/.hermes/.env`. Keep API keys, bot tokens and OAuth secrets in `.env`; prefer `config.yaml` where a config key exists. Some variables are process-only overrides or internal bridge variables and should not be committed to `.env`. Every example value below is a placeholder.

## Variable Families

> Sources: [Environment Variables](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) · [reference/environment-variables.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/reference/environment-variables.md)

The reference page is organized into these groups; representative variables (all documented there) are listed per group. It is not a complete list.

| Family (page heading) | Examples |
|---|---|
| LLM providers | `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` + `OPENAI_BASE_URL` (custom endpoint), `GOOGLE_API_KEY`, `GEMINI_API_KEY`, `XAI_API_KEY`, `DEEPSEEK_API_KEY`, `LM_API_KEY`, `AI_GATEWAY_API_KEY`, `FIREWORKS_API_KEY`; `*_BASE_URL` overrides per provider |
| Provider auth (OAuth) | Nous Portal and related overrides (`HERMES_PORTAL_BASE_URL`, `NOUS_INFERENCE_BASE_URL`, ...); manual token overrides such as `ANTHROPIC_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`. OAuth credentials themselves are stored in `auth.json`. |
| Tool APIs | Web: `EXA_API_KEY`, `FIRECRAWL_API_KEY`, `TAVILY_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_SEARCH_API_KEY`, `SEARXNG_URL`; browser: `BROWSERBASE_API_KEY`, `BROWSER_USE_API_KEY`, `BROWSER_CDP_URL`; media: `FAL_KEY`, `GROQ_API_KEY`, `ELEVENLABS_API_KEY`; memory providers: `HONCHO_API_KEY`, `MEM0_API_KEY`, `SUPERMEMORY_API_KEY`, ...; sandboxes: `DAYTONA_API_KEY`, `VERCEL_TOKEN` |
| Terminal backend | `TERMINAL_ENV` (`local`, `docker`, `ssh`, `modal`, `daytona`, `vercel_sandbox`, or a plugin backend), `TERMINAL_DOCKER_IMAGE`, `TERMINAL_DOCKER_VOLUMES`, `TERMINAL_TIMEOUT`, `TERMINAL_LIFETIME_SECONDS`, `TERMINAL_CWD` (deprecated; use `terminal.cwd`), `SUDO_PASSWORD`; SSH: `TERMINAL_SSH_HOST`, `TERMINAL_SSH_USER`, `TERMINAL_SSH_PORT`, `TERMINAL_SSH_KEY` |
| Messaging | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, `DISCORD_BOT_TOKEN`, `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `HASS_TOKEN`/`HASS_URL`, `GATEWAY_ALLOW_ALL_USERS` (default `false`); many per-platform subsections |
| Web dashboard and Desktop | Dashboard auth/session variables (for example `HERMES_DASHBOARD_SESSION_TOKEN`) |
| Agent behavior | `HERMES_INFERENCE_MODEL`, `HERMES_YOLO_MODE`, `HERMES_IGNORE_USER_CONFIG`, `HERMES_IGNORE_RULES`, `HERMES_EPHEMERAL_SYSTEM_PROMPT`, `HERMES_REDACT_SECRETS` (default `true`), `HERMES_WRITE_SAFE_ROOT`, `HERMES_API_TIMEOUT`, `HERMES_AGENT_TIMEOUT`, `HERMES_KANBAN_TASK` (set by the kanban dispatcher; do not set manually) |
| Interface | `HERMES_TUI`, `HERMES_TUI_RESUME`, `HERMES_TUI_THEME` |
| Session settings | `HERMES_SESSION_ID`, `AI_AGENT`, `HERMES_AGENT` - exported by Hermes into tool subprocesses; do not set manually |
| Config-only topics | The page has headings for compression, fallback providers and provider routing explicitly stating that they have **no** env vars and live in `config.yaml` |

Treat the reference page as authoritative for defaults. Documentation inconsistencies to be aware of: the page lists `HERMES_MAX_ITERATIONS` with a default of 500, while the Configuration page and `DEFAULT_CONFIG` show `agent.max_turns` as unlimited by default.

## Home, Profile and Config-Backend Variables

> Sources: [Environment Variables](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) · [Managed Scope](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope) · [Remote Config](https://hermes-agent.nousresearch.com/docs/user-guide/remote-config) · [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_constants.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_constants.py)

| Variable | Documented behavior |
|---|---|
| `HERMES_HOME` | Selects the configuration and user-data home. A literal `~` or `$VAR` in the value is expanded. Defaults to `~/.hermes` (POSIX) or `%LOCALAPPDATA%\hermes` (Windows); the official Docker image uses `/opt/data`. Profile or runtime context can select a more specific home. The env writer refuses to set it through `hermes config set`. |
| `HERMES_CONFIG_BACKEND` | `file` (default) or `remote`. Read from the launch environment, the launch home's `.env` and the managed `.env`, never from `config.yaml`. Any other value stops Hermes at startup. |
| `HERMES_CONFIG_REMOTE_URL`, `HERMES_CONFIG_INSTANCE_ID` | Remote-config plane URL and this agent's id; the instance id is required with the remote backend. |
| `HERMES_MANAGED_DIR` | Relocates the managed-scope directory (default `/etc/hermes`); a deployment/bootstrap knob that Hermes never persists to a `.env`. Documented on the Managed Scope page. |
| `HERMES_TIMEZONE` | IANA timezone override. |
| `HERMES_KANBAN_HOME`, `HERMES_KANBAN_BOARD`, `HERMES_KANBAN_DB` | Anchor and select the kanban board storage. |

The old guide listed `HERMES_PROFILE` as a user-facing profile selector. The user docs do not document it as one: profiles are selected with `-p/--profile` or `hermes profile use` (section 5). The multiplexing design page does name `HERMES_PROFILE` in an internal allowlist of process-level variables, and other pages show it set for kanban workers, so treat it as internal.

## Loading Order and Where Secrets Come From

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) · [Managed Scope](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope) · [hermes_cli/env_loader.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/env_loader.py)

- The profile's `.env` is the normal place for secrets (`hermes config set OPENROUTER_API_KEY <your-api-key>` writes there). The setup flows write the same file the runtime reads.
- `hermes_cli/env_loader.py` (`load_hermes_dotenv`) documents the load behavior in its docstring: the user `~/.hermes/.env` overrides stale shell exports, and a project `.env` is a development fallback that only fills gaps when the user env exists. The managed `.env` is applied last and wins for the keys it sets. This is a source-docstring statement, not an executed check.
- Only credential-shaped names (`*_API_KEY`, `*_TOKEN`, `*_SECRET`, `*_KEY`) are sanitized on load (non-ASCII warnings, since they become HTTP header values); other variables are never silently altered (source comment).
- **External secret sources** (Bitwarden, 1Password, a command helper) can inject keys at startup instead of storing them in `.env`; only the bootstrap token lives in `.env`. Precedence: your `.env`/shell wins by default unless a source sets `override_existing`; mapped sources beat bulk sources; first source wins. `secrets.preserve_existing` and profile aliasing (`FOO_<PROFILE>` hydrates `FOO`) support shared vaults across profiles.
- Remote-config deployment variables (`HERMES_CONFIG_*`, `GATEWAY_RELAY_IDP_*`, `HERMES_PORTAL_BASE_URL`, `NOUS_PORTAL_BASE_URL`, `HERMES_SHARED_AUTH_DIR`) must come from `auth.json`, a `.env` or the process environment; if a `secrets:` source supplies one, Hermes refuses to start (Remote Config page).

## Secret Handling and Sandboxes

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) · [Environment Variables](https://hermes-agent.nousresearch.com/docs/reference/environment-variables) · [user-guide/security.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/security.md)

- **Stripped from children by default.** The Security page states that `execute_code` and `terminal` strip sensitive environment variables from child processes to prevent credential exfiltration by model-generated code. Terminal commands, `execute_code` sandboxes and `no_agent` cron scripts run with a sanitized environment (Secrets page).
- **Passthrough is explicit.** Variables reach children only if a loaded skill declares them in `required_environment_variables` or you list them under `terminal.env_passthrough`:

```yaml
terminal:
  env_passthrough:
    - MY_CUSTOM_KEY       # a name, never a value
```

  Infrastructure secrets (provider keys, gateway tokens) should not be added to `env_passthrough`; the Security page says such names are refused when declared. The old guide's claim that `.env` is "NEVER exposed to the LLM model" is not restated: the docs describe the stripping and redaction mechanisms above instead of an absolute guarantee. Managed-scope docs add that the agent's own tools are not hard-blocked from overriding a managed env value inside a subprocess.
- **Redaction.** `HERMES_REDACT_SECRETS` / `security.redact_secrets` (default true per the reference) controls secret redaction in tool output, logs and chat responses; logs are described as secrets-auto-redacted.
- **Write sandbox.** `HERMES_WRITE_SAFE_ROOT` hard-blocks `write_file`/`patch` outside the listed roots (the official Docker image sets it to the data volume). Do not set it in `.env` casually: Hermes state paths outside the listed roots then fail to write.
- **Config references.** In `config.yaml`, reference secrets as `${VAR}` instead of inlining them (section 3). In remote-config mode a secret-shaped key accepts only a `${VAR}` reference.
- **Multi-profile isolation.** Under a multiplexed gateway each profile's provider keys and bot tokens resolve from that profile's own `.env`, never from the default profile's (section 5).

Placeholder example of a minimal profile `.env` (never commit real values):

```bash
OPENROUTER_API_KEY=<your-api-key>
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_ALLOWED_USERS=<comma-separated-user-ids>
```

---

# 5. HERMES_HOME & Profile Isolation

> Sources: [Profiles: Running Multiple Agents](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) · [Running Many Gateways at Once](https://hermes-agent.nousresearch.com/docs/user-guide/multi-profile-gateways) · [Profile Distributions](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions) · [Profile Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/profile-commands) · [Multiplexing Gateway](https://hermes-agent.nousresearch.com/docs/developer-guide/multiplexing-gateway) · [hermes_constants.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_constants.py) · [hermes_cli/profiles.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/profiles.py)

A profile is a separate Hermes home. This section covers how the home is resolved, what a profile contains, the profile commands, how the gateway serves several profiles at once (the biggest change from older guides), and how profiles are shared.

## How the Home Is Resolved

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Profiles: Running Multiple Agents](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) · [hermes_constants.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_constants.py) · [hermes_cli/main.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/main.py)

```text
Default home:  ~/.hermes
Named profile: <default root>/profiles/<name>/
```

- `get_hermes_home()` is the authoritative resolver: context-local override first, then the `HERMES_HOME` environment variable, then the platform default (`~/.hermes` on macOS/Linux, `%LOCALAPPDATA%/hermes` on Windows). Never hard-code `~/.hermes`; use `display_hermes_home()` only for user-facing text.
- The CLI bootstrap (`_apply_profile_override()`) runs before the rest of Hermes is imported. An explicit `--profile`/`-p` resolves that profile and writes the directory to `HERMES_HOME`. Without a selector, a profile-specific `HERMES_HOME` is preserved; otherwise the default root's `active_profile` selection (set by `hermes profile use`) can apply.
- `HOME` only determines the platform default. Changing `HOME` is not a safe way to select a profile, and a subprocess that drops `HERMES_HOME` can fall back to the default profile, so spawners should pass `HERMES_HOME` explicitly.
- Hermes recognizes a directory under `profiles/` as a profile only if it carries an identity file (`config.yaml`, `.env`, `SOUL.md`, `profile.yaml`, `auth.json` or `state.db`).
- The default profile is `~/.hermes` itself; its internal id is always `default` (renaming it sets only a display name).

## What a Profile Contains and What It Does Not Isolate

> Sources: [Profiles: Running Multiple Agents](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) · [user-guide/profiles.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/profiles.md)

Each profile has its own `config.yaml`, `.env`, `SOUL.md`, memories, sessions, skills, cron jobs, logs, `state.db` and gateway state. Points the docs stress:

- **Never point two agent processes at the same profile.** Both write memory automatically and each loads the other's writes at session start. Agents that need shared memory should use an external memory provider.
- **A profile is not a sandbox or a workspace.** Terminal commands start from `terminal.cwd`, not the profile directory; on the local backend the agent has the same filesystem access as your user. Set an absolute `terminal.cwd` for a fixed project directory.
- **`HERMES_HOME` vs `HOME`.** On host installs tool subprocesses keep the real OS-user `HOME` by default so tools like `git`, `ssh` and `gh` find their normal credentials. Container backends use `{HERMES_HOME}/home`. Set `terminal.home_mode: profile` for strict per-profile CLI identities; `HERMES_REAL_HOME` is exposed so scripts can still find the real home in that mode.
- **OAuth logins are shared by default.** Single-use refresh-token logins (Anthropic, OpenAI Codex, xAI) are not copied by `--clone-all`; the new profile reads the root `auth.json`, and refreshes are written back to root. Run `hermes -p <name> auth add <provider>` to give a profile its own login.

## Profile Commands

> Sources: [Profile Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/profile-commands) · [Profiles: Running Multiple Agents](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) · [reference/profile-commands.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/reference/profile-commands.md)

```bash
hermes profile create coder          # blank profile (bundled skills seeded) + "coder" command alias
coder setup                          # alias is `hermes -p coder ...`
hermes -p coder chat                 # explicit flag; works in any position
hermes profile use coder             # sticky default (like kubectl config use-context)
hermes profile list                  # also: show <name>, describe <name>
hermes profile rename coder dev-bot
hermes profile delete coder          # stops gateway, removes service and alias, deletes data
hermes profile export coder          # coder.tar.gz, API keys stripped
hermes profile import coder.tar.gz
```

| Subcommand | Purpose (per the reference page) |
|---|---|
| `list`, `show`, `use`, `create`, `describe`, `delete`, `alias`, `rename` | Core management; `describe` stores a description used by the kanban orchestrator for routing |
| `migrate-identity`, `purge-identity` | Retry a rename's or delete's session-identity step when the live gateway could not do it |
| `export`, `import` | Single `.tar.gz` file; in chat, `/export` and `/import` |
| `install`, `update`, `info` | Profile distributions (below) |

`hermes profile create` options: `--clone` (config, `.env`, `SOUL.md`, skills, `memories/MEMORY.md`, `memories/USER.md`, the active memory provider's config; not sessions, `state.db` or cron), `--clone-all` (everything except per-profile history, and cron jobs, which stay with the source so jobs do not fire twice), `--clone-from <profile>`, `--no-alias`, `--description`, `--no-skills`. **Messaging channels are never cloned** (bot tokens, allowlists, `platforms:` sections, `gateway.multiplex_profiles`/`profile_routes`) unless `--clone-channels` is passed, because a bot can belong to only one profile. The Profiles page also covers a corner case: if the sticky profile's directory is deleted by hand, ordinary commands refuse to run rather than silently use the default profile.

## One Gateway for All Profiles (Multiplexing)

> Sources: [Running Many Gateways at Once](https://hermes-agent.nousresearch.com/docs/user-guide/multi-profile-gateways) · [Multiplexing Gateway](https://hermes-agent.nousresearch.com/docs/developer-guide/multiplexing-gateway) · [Hosted rooms across local profile owners](https://hermes-agent.nousresearch.com/docs/developer-guide/hosted-profile-owners) · [user-guide/multi-profile-gateways.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/multi-profile-gateways.md) · [hermes_cli/gateway_multiplex_mode.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/gateway_multiplex_mode.py)

The older model (one gateway process per profile, each with its own bot token) is no longer the default. Per the Multi-profile Gateways page:

- **Multiplexing is on by default** (`gateway.multiplex_profiles` defaults to `true`): one gateway process, launched from the default profile, serves the default profile plus every live named profile. Each profile keeps its own adapters, secrets, sessions and cron ticks; the process shares one event loop, HTTP listener, process lock and status surface.
- **An unset flag is settled at boot** by a preflight (the same one as `hermes gateway migrate --multiplex`): it multiplexes only when the fold is safe (two or more profiles, no secondary still running its own gateway, no duplicate bot credential, no port-binding platform without a `/p/<profile>/` ingress), writing `true` into the default profile's config when it does. Otherwise it serves the default profile only and warns loudly. `false` is retired and rewritten to `true`.
- **Named profiles attach rather than start.** `hermes -p <name> gateway run` while a host gateway is live attaches (exit 0, or 75 under a service supervisor so the unit retries). `gateway install`/`start` for a served named profile refuses with exit 78 unless `--force`. A profile that needs its own process sets `gateway.standalone: true` in its own `config.yaml` (documented as a temporary compatibility shim; ignored on the default profile).
- **Park instead of stop.** `hermes -p coder gateway stop` writes a `gateway.parked` marker and unserves just that profile; `start` removes it; `restart` re-reads config. The host rescans every 30 seconds. `gateway start --all` / `restart --all` mean the one host multiplexer.
- **Live served set.** Profiles created, deleted or renamed while the multiplexer runs are served or unserved without a restart. A hot-added profile that reuses another profile's bot token is parked with a `duplicate_credential` error: one credential, one poller.
- **Session keys are namespaced** `agent:<profile>:...`; the default profile keeps the historical `agent:main:...` namespace unchanged. Each profile's rows land in its own `state.db`.
- **HTTP-inbound platforms** are reached under a `/p/<profile>/` URL prefix.
- **`profile_routes`** lets several chats on one shared bot token route to different profiles (by guild, channel, thread, chat id and similar), declared on the default profile.
- **Single status surface.** One PID/lock and one `gateway_state.json` live under the default home (with `served_profiles`); `hermes status`, `hermes -p <name> gateway status` and the dashboard report the multiplexer as that profile's running gateway.

How isolation is implemented (Multiplexing Gateway page): every inbound event composes two context-local scopes (a `HERMES_HOME` override and a secret scope built from the profile's `.env` and secret sources) before any profile-owned code runs. They are `contextvars`, propagate into worker threads, unwind deterministically, and nothing is written to `os.environ`. With multiplexing armed, a credential read with no scope raises rather than falling back to the process environment. The docs' stated invariant: profile A's turns must never observe profile B's state. The Multi-profile Gateways page has a per-concern table (provider keys, authorization, slash-command gating, HTTP endpoints, adapter settings, MCP child environments, egress, session namespace, logs, terminal settings, command approvals and more) of what each profile resolves from its own home; known limitations are listed on the design page and are not repeated here. Hosted rooms across local profile owners are also described (each profile owns only its own database and is reached over private control sockets); the docs say execution between two profiles served by the same multiplexer is covered by in-process tests and is not yet certified on a live multi-profile install.

### Sessions Are Owned by the Gateway

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Classic gateway client](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-classic-cli) · [Gateway Session Lifecycle](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-session-lifecycle) · [developer-guide/gateway-session-lifecycle.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/gateway-session-lifecycle.md)

Profile isolation interacts with the unified session-ownership model from section 1: a profile's gateway is the only writer of that profile's canonical sessions, and `hermes chat`, the TUI, local Desktop, the API server, ACP, cron, Bot Chat and Kanban workers are its clients. The gateway session-lifecycle page documents the data model (`SessionSource`, `SessionEntry`, `SessionStore`, session keys and the suspended/resume-pending state machine); the isolation consequence is that each profile's session store, state DB and key namespace are separate even when one process serves many profiles.

## Profile Distributions and Export Files

> Sources: [Profile Distributions](https://hermes-agent.nousresearch.com/docs/user-guide/profile-distributions) · [Profile Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/profile-commands) · [user-guide/profile-distributions.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/profile-distributions.md)

Two ways to share a profile:

| | Distribution (git repo) | Export file (`.tar.gz`) |
|---|---|---|
| Ship by | `hermes profile install <repo>` | Send the file; `/export`, `/import` or `hermes profile export` / `import` |
| Updates | `hermes profile update <name>` | Re-send |
| Carries | SOUL, config, skills, cron, MCP, plugins | The same, plus Desktop theme and layout when exported from the desktop app |
| Credentials, memories, sessions | Never shipped | API keys stripped (read the "what an export contains" section before sending one) |

Distribution essentials (from the Profile Commands and Profile Distributions pages):

```yaml
# distribution.yaml at the repo root
name: research-bot
version: 1.0.0
description: "Autonomous research assistant"
hermes_requires: ">=0.12.0"
env_requires:
  - name: OPENAI_API_KEY
    description: "API key for model access"
    required: true
# distribution_owned: [SOUL.md, skills/, cron/]   # optional; defaults apply when omitted
```

- `hermes profile install <source> [--name NAME] [--alias] [--force] [--yes]` accepts a git URL (`github.com/user/repo`, HTTPS, SSH) or a local directory containing `distribution.yaml`. Private repos work through your normal git authentication. It shows the manifest, lists required env vars (written to `.env.EXAMPLE`) and warns about cron jobs before confirming.
- `hermes profile update <name> [--force-config] [--yes]` re-clones from the recorded source. Distribution-owned files (SOUL, `mcp.json`, shipped skills) are replaced; cron definitions merge by job id, with newly shipped jobs arriving paused; the installer's `config.yaml` is preserved unless `--force-config`. Memories, sessions, auth and `.env` are never touched.
- `hermes profile info <name>` prints the stored manifest, source and install timestamp.
- Credentials (`auth.json`, `.env` and other credential stores) and machine-specific package-manager state are excluded from distributions and exports. The docs warn that Hermes does not control git: the exclusions apply when installing, not when you `git add`, so write a `.gitignore` before the first commit.
- In remote-config mode, profile clone, rename and distribution install are refused (section 3).

## Related Operational Pages

> Sources: [Running Many Gateways at Once](https://hermes-agent.nousresearch.com/docs/user-guide/multi-profile-gateways) · [Connecting Desktop to Many Hermes Instances](https://hermes-agent.nousresearch.com/docs/user-guide/multi-connection-desktop)

The Multi-profile Gateways page also covers service files (LaunchAgent, systemd user/system service, Windows Scheduled Task, s6/Docker), log locations per profile, keeping the host awake, token-conflict safety, the `hermes update` automatic migration from per-profile gateways to the multiplexer (with opt-out and blocker lists) and troubleshooting. Connecting one desktop app to Hermes instances on different machines is covered separately. These are not reproduced here.

---

# 6. Agent Identity — SOUL.md

> Sources: [Personality & SOUL.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) · [Use SOUL.md with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes) · [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

`SOUL.md` is the agent's primary identity. Per the docs it occupies slot #1 of the system prompt and replaces the built-in default identity text; it is not a context file and is not duplicated in the project-context block. This section describes what the docs and source state at the pinned SHA. Nothing here was executed.

## Where the file lives and how it is created

> Sources: [Personality & SOUL.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) · [Use SOUL.md with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

| Fact | What the docs state |
|------|---------------------|
| Path | `~/.hermes/SOUL.md`, or `$HERMES_HOME/SOUL.md` when a custom home (or profile) is in use. |
| Search scope | `HERMES_HOME` only. Hermes does not look in the working directory for a `SOUL.md`. |
| Seeding | Hermes seeds a starter `SOUL.md` if none exists; an existing file is never overwritten. |
| Empty / unreadable | An empty, whitespace-only, or unloadable file means the built-in default identity is used instead. |
| Wrapper text | None. The file content is injected verbatim after scanning and truncation. |
| Duplication | `build_context_files_prompt(skip_soul=True)` prevents a second copy in the `# Project Context` block. |

Because the file is read from `HERMES_HOME`, identity belongs to the Hermes instance (or profile), not to whichever directory you launched from. See section 5 for profile isolation: each profile has its own `SOUL.md`.

## Loading logic (docs and source)

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py)

The developer guide shows this simplified shape (the real function takes a `context_length` and an optional `home_override`):

```python
# simplified, per the Prompt Assembly page
def load_soul_md() -> Optional[str]:
    soul_path = get_hermes_home() / "SOUL.md"
    if not soul_path.exists():
        return None
    content = soul_path.read_text(encoding="utf-8").strip()
    content = _scan_context_content(content, "SOUL.md", user_authored=True)  # warn + load, never block
    content = _truncate_content(content, "SOUL.md")   # cap scales with model context window
    return content
```

Details confirmed in `agent/prompt_builder.py` (`load_soul_md`):

- The read is scoped to the agent's own profile home when a `home_override` is supplied (a guard against a thread that lost the `HERMES_HOME` context variable reading another profile's file).
- A legacy generated Bot Mode protocol block, if present, is stripped before the text is scanned and truncated.
- If a profile distribution manifest owns `SOUL.md` (installed by `hermes profile install <git-url>` / updated by `hermes profile update`), the file is treated as not user-authored and a scanner hit still blocks it.
- In `agent/system_prompt.py`, `_identity_parts` calls `load_soul_md` when `load_soul_identity` is set or `skip_context_files` is not; otherwise (for example subagent delegation, where `skip_context_files` is set) the hard-coded `DEFAULT_AGENT_IDENTITY` is used. The docs state the same for subagents: SOUL.md is excluded from the project context embedded in a child's prompt.

## Security scan and truncation

> Sources: [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [Use SOUL.md with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

- **Scan.** Context files are checked for prompt-injection patterns (instruction overrides, deception phrases, hidden HTML comments or `display:none` divs, credential-exfiltration shapes, secret-file reads, invisible characters). A hit in a *project* file replaces it with a `[BLOCKED: ...]` marker.
- **SOUL.md is treated differently.** It is your own file (agent writes to it need approval), so a scanner hit logs a warning naming the pattern, still loads the file, and `/context` marks it with a review warning. The exception is a distribution-owned `SOUL.md` (above), where a hit still blocks.
- **Size cap.** The old guide said "cap at 20k chars". Current docs say the cap scales with the model's context window (floor 20,000 characters, ceiling 500,000); an explicit `context_file_max_chars` in `config.yaml` always wins. Truncation keeps a head/tail split (70% head, 20% tail, marker in the middle). `context_file_read_timeout` (default 5 seconds) skips files that take too long to read.

## Default identity fallback

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [Personality & SOUL.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

When no usable `SOUL.md` is present, the constant `DEFAULT_AGENT_IDENTITY` is used. The docs quote it as opening with "You are Hermes Agent, built by Nous Research. Be direct: match the length of your reply to the weight of the ask ..." followed by brevity and anti-filler guidance. Treat the exact wording as version-specific; read `DEFAULT_AGENT_IDENTITY` in `agent/prompt_builder.py` for the current text.

## SOUL.md vs AGENTS.md vs /personality

> Sources: [Use SOUL.md with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes) · [Personality & SOUL.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) · [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)

| Mechanism | Scope | Use for |
|-----------|-------|---------|
| `SOUL.md` | Whole Hermes instance / profile, every session | Tone, voice, directness, how to handle uncertainty and disagreement. |
| `AGENTS.md` (and `.hermes.md`, `CLAUDE.md`, `.cursorrules`) | The project / directory | Architecture, conventions, commands, ports, repo workflow. |
| `/personality <name>` | Session-level overlay | A temporary mode switch on top of `SOUL.md`. |
| `agent.system_prompt` | Manual system prompt you write | Applies only when no personality is selected. |

Rule of thumb from the docs: if it should follow you everywhere it belongs in `SOUL.md`; if it belongs to one project it belongs in `AGENTS.md`.

## Built-in and custom personalities

> Sources: [Personality & SOUL.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) · [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

Built-in presets listed on the personality page: `helpful`, `concise`, `technical`, `creative`, `teacher`, `kawaii`, `catgirl`, `pirate`, `shakespeare`, `surfer`, `noir`, `uwu`, `philosopher`, `hype`. They are available on every surface (CLI, messaging, TUI, desktop). Switch with `/personality <name>`.

Define your own (or override a built-in by reusing its name) under `agent.personalities` in `config.yaml` (a top-level `personalities:` block also works; `agent.personalities` wins on conflict):

```yaml
agent:
  personalities:
    codereviewer: >
      You are a meticulous code reviewer. Identify bugs, security issues,
      performance concerns, and unclear design choices. Be precise and constructive.
```

The selected name is stored in `display.personality`. To return to base behavior (your `SOUL.md`, plus `agent.system_prompt` if set) use `/personality none`, `/personality default`, or `/personality neutral`. The docs also describe a one-time reset of saved personality selections on the first run after an upgrade from older versions.

Conversational persona is separate from terminal appearance: `display.skin` and `/skin` change how Hermes looks, not how it speaks.

## Picking up edits

> Sources: [Which file does what](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Use SOUL.md with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes)

The system prompt is assembled at session start, so an edited `SOUL.md` normally takes effect in a new session. The CLI reference documents `hermes sessions refresh-prompts` (optionally `--session ID`, `--json`, `-p/--profile`), which makes continuing sessions rebuild their stored system prompt from the current `SOUL.md`, skills and context files on their next turn, keeping history and session ids at the cost of one prompt-cache miss per refreshed session.

---

# 7. Prompt Assembly — System Prompt Architecture

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

Hermes separates a **cached system prompt** (built once per session and reused) from **ephemeral, API-call-time additions**. The split protects token usage, provider prompt caching, session continuity and memory correctness. The original architecture note described a flat ten-layer list; the developer guide now describes a three-tier model, and this section follows it.

## Three cached tiers

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py)

`build_system_prompt_parts()` in `agent/system_prompt.py` returns three strings, joined as `stable` then `context` then `volatile`:

| Tier | Contents (per the docs and source) |
|------|------------------------------------|
| **stable** | Identity (`SOUL.md` or `DEFAULT_AGENT_IDENTITY`), tool-aware and model-gated guidance, the hermes-agent help pointer, pinned/auto-loaded skills, the coding operating brief, platform hints (including `platform_hints` overrides). |
| **context** | Caller-supplied `system_message`, project context files (`.hermes.md` / `AGENTS.md` / `CLAUDE.md` / `.cursorrules`), then the worktree-dependent git workspace snapshot and related workspace guidance. A session with no workspace snapshot keeps its trailing guidance in the stable tier. |
| **volatile** | Skills index, built-in memory snapshot (`MEMORY.md`), user profile snapshot (`USER.md`), external memory-provider block, plugin prompt sections, the active-profile line, the timestamp/session/model/provider line, then the runtime environment hints (host, home, current working directory). The runtime block always ends the volatile tier. |

```text
 system prompt = stable  +  context  +  volatile            (cached per session)
                 |           |            |
   identity, guidance,   project files,   skills index, MEMORY/USER snapshots,
   platform hints,       workspace        memory provider block, timestamp,
   coding brief          snapshot         runtime environment (cwd) -- last

 per API call (NOT cached): ephemeral_system_prompt, prefill messages,
   gateway context overlays, pre_llm_call plugin context, recall injected
   into the current user message
```

Consequences stated in the docs:

- Skills and memory/profile snapshots live in the *cached* prompt (in the volatile tier), not as mid-turn overlays. The prose on the Prompt Assembly page once calls skills part of the stable tier, but its own tier list and `build_system_prompt_parts()` place the skills index first in the volatile tier; the code is followed here.
- Project context files precede anything that names the current worktree, so sessions of one project in different git worktrees share a longer common prompt prefix for a longest-prefix provider cache.
- Persisted prompts are validated by `_stored_prompt_matches_runtime()` (`agent/conversation_loop.py`) against the rendered `# Hermes runtime environment` boundary; legacy prompts persisted before the reorder still validate.
- `Platform:` is deliberately not an identity field. A surface switch (for example desktop to TUI) keeps the stored bytes and delivers the new surface's guidance as a one-shot note on the per-turn user message (`agent/surface_switch.py`).
- The `<available_skills>` index is likewise compared each turn against what the session would build now (`agent/skills_index_delta.py`); a difference rides the same one-shot note channel, and the index itself converges at the next rebuild boundary (compaction).
- When `skip_context_files` is set (for example subagent delegation) `SOUL.md` is not loaded and the default identity is used.

## Illustrative assembled prompt

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py)

The developer guide shows a simplified, numbered sketch of the assembled prompt. It is illustrative only; the authoritative order is the three-tier code above. Sections that appear in it:

1. Agent identity from `SOUL.md`.
2. Tool-aware behavior guidance (task-learned procedures belong in skills; memory is for facts that apply to every session; use `session_search` for cross-session recall). Skill-writing instructions appear only when `skill_manage` is available.
3. Tool-use enforcement text for GPT/Codex-family models only (model-gated).
4. External memory-provider static block, when active.
5. Optional system message (config or API supplied).
6. Frozen `MEMORY.md` snapshot ("Persistent Memory").
7. Frozen `USER.md` snapshot ("User Profile").
8. Skills index (`<available_skills>` grouped by category, with the instruction to load a matching skill via `skill_view`).
9. Project context under a `# Project Context` header.
10. Timestamp and session line, then the platform hint (for example a CLI hint to prefer plain text).

Because the sketch lists memory and skills before project context while the code puts project context in the earlier *context* tier, do not treat the numbering as a precedence order.

## Context files: discovery and limits

> Sources: [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py) · [agent/subdirectory_hints.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/subdirectory_hints.py)

Only one project-context type loads per session (first match wins):

| Priority | File(s) | Scope |
|----------|---------|-------|
| 1 | `.hermes.md` / `HERMES.md` | Walks from the cwd up to the git root. YAML frontmatter is stripped (reserved for future config overrides). |
| 2 | `AGENTS.md` chain (with `AGENTS.override.md`) | Merged chain from git root down to the cwd; deeper files come later and take precedence; identical copies are de-duplicated. Per directory the first non-empty of `AGENTS.override.md` / `AGENTS.md` / `agents.md` wins, so a personal (usually gitignored) override replaces the committed file. Outside a git repo only the cwd is checked. |
| 3 | `CLAUDE.md` | cwd only. |
| 4 | `.cursorrules`, `.cursor/rules/*.mdc` | cwd only. |

The Context Files page lists the priority as `.hermes.md` then `AGENTS.override.md` then `AGENTS.md` then `CLAUDE.md` then `.cursorrules`; the source (`_agents_md_candidates`) matches the override behavior. The Prompt Assembly page's shorter four-row list omits the override file.

All context files are security-scanned (project files are blocked on a hit), and truncated at `context_file_max_chars` (or the dynamic cap scaled to the model context window, floor 20,000 and ceiling 500,000 characters, 70/20 head/tail with a marker).

**Progressive subdirectory discovery.** After the startup load, `SubdirectoryHintTracker` (`agent/subdirectory_hints.py`) watches tool arguments (`path`, `workdir`, shell commands). When the agent touches a new directory it walks that directory and up to five ancestors, loads the first match of `AGENTS.override.md` / `AGENTS.md` / `CLAUDE.md` / `.cursorrules` there, scans it, caps it at 32,000 characters, and appends it to the *tool result*, so the system prompt stays stable. Each directory is checked at most once per session.

## Platform hints and `platform_hints`

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py)

Built-in per-surface guidance lives in `PLATFORM_HINTS`; plugin platforms supply theirs through the platform registry. An administrator can adjust one platform's hint with the top-level `platform_hints` key:

```yaml
platform_hints:
  whatsapp:
    append: >
      When tabular output would be useful, invoke the table_formatting
      skill instead of emitting a Markdown table.
  slack:
    replace: "You are on Slack. Keep responses tight and avoid wide tables."
  telegram: "Prefer short messages; split long answers."   # shorthand = append
```

`append` keeps the built-in hint and adds text; `replace` substitutes it; a bare string is shorthand for `append`; `replace` wins if both are present; malformed entries are ignored. Cron jobs run as platform `cron` but also carry the hint of their `deliver` channel under a `Delivery destination (<channel>):` line. The override is resolved when the prompt is built (session start and compaction rebuild) and is byte-stable for a fixed config, so it sits in the stable tier without breaking caching.

## API-call-time-only layers

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [Observer Hooks](https://hermes-agent.nousresearch.com/docs/developer-guide/observer-hooks)

Not persisted in the cached prompt (per the docs):

- `ephemeral_system_prompt` (the docs name `HERMES_EPHEMERAL_SYSTEM_PROMPT` as an environment-level way to supply turn-scoped guidance).
- Prefill messages.
- Gateway-derived session-context overlays.
- Later-turn external recall injected into the current user message.
- `pre_llm_call` plugin context, appended to the current turn's user message (multiple plugins' blocks are concatenated). Per the observer-hook contract, `pre_llm_call` may also return `{"action": "block", ...}` to end the turn before any inference request.

## Memory snapshots and rebuilds

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [Context Compression & Prompt Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py)

`MEMORY.md` and `USER.md` are captured in the volatile tier at build time. Mid-session memory writes update disk but do not mutate the cached prompt. `invalidate_system_prompt()` clears the cache after compression, reloads memory from disk, and clears the frozen plugin-section snapshot so the next turn rebuilds everything at the compaction boundary. See sections 13 and 14 for memory and compression details.

## Supported customization surfaces

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [Which file does what](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what)

The docs treat `agent/prompt_builder.py` as implementation, not a configuration surface. Use instead: `SOUL.md` (identity), `MEMORY.md`/`USER.md` (durable facts), project context files, skills, optional system-prompt config/API overrides, `platform_hints`, `agent.*` guidance toggles, and ephemeral overlays. Edit the Python only when maintaining a fork or contributing a global change.

## Reviewing prompt-surface changes (contributor workflow)

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [scripts/ci/prompt_surface.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/scripts/ci/prompt_surface.py)

The rendered "prompt surface" (the system prompt plus the `tools[]` array a stock install sends) is committed and reviewed like code. `scripts/ci/prompt_surface.py render` builds it through the production assembly code and writes snapshots under `tests/fixtures/prompt_surface/`; `tests/ci/test_prompt_surface.py` fails when the snapshot is stale; a CI job posts per-surface diffs to the PR; and changes to the snapshot need approval from the core owners group. User-side customization never touches the snapshot. The docs say the render covers a fixed set of surfaces (CLI with several model families, a code workspace, desktop, Telegram, Discord and cron).

---

# 8. Agent Loop Internals (AIAgent)

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [run_agent.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/run_agent.py) · [agent/conversation_loop.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/conversation_loop.py)

`AIAgent` is the orchestration engine behind CLI, gateway, ACP, TUI and API-server sessions. The key structural change since the original note: `run_agent.py` is now a thin facade. The loop body lives in `agent/conversation_loop.py`, each turn phase in `agent/turn_*.py`, constructor wiring in `agent/agent_init.py`, and prompt assembly, tool dispatch and provider failover in focused `agent/*.py` modules mixed into `AIAgent`.

## Responsibilities

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [agent/agent_init.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/agent_init.py)

- Assemble the effective system prompt and tool schemas (section 7, section 9).
- Select the provider/API mode.
- Make interruptible model calls with cancellation.
- Execute tool calls sequentially or concurrently.
- Keep conversation history in OpenAI message format.
- Handle compression, retries and fallback model switching.
- Track iteration budgets across parent and child agents.
- Flush persistent memory before context is lost.

## Entry points

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [run_agent.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/run_agent.py)

```python
# Simple interface: returns the final response string
response = agent.chat("Fix the bug in main.py")

# Full interface: returns a dict with messages, metadata, usage stats
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,           # auto-built if omitted
    conversation_history=None,     # auto-loaded from session if omitted
    task_id="task_abc123",
)
```

`chat()` wraps `run_conversation()` and returns the `final_response` field.

## API modes

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime)

| API mode | Used for | Client |
|----------|----------|--------|
| `chat_completions` | OpenAI-compatible endpoints (OpenRouter, custom endpoints, most providers) | `openai.OpenAI` |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` with Responses format |
| `anthropic_messages` | Native Anthropic Messages API | `anthropic.Anthropic` via adapter |

Resolution order per the docs: explicit `api_mode` constructor argument, then provider-specific detection, then base-URL heuristics (for example an Anthropic-style endpoint), then the default `chat_completions`. All modes converge on the same internal OpenAI-style message format before and after the API call. Provider selection itself is covered in section 17.

## Turn lifecycle

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [agent/conversation_loop.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/conversation_loop.py)

```text
run_conversation()
  1. Generate task_id if not provided
  2. Append user message to conversation history
  3. Build or reuse the cached system prompt        (agent/system_prompt.py)
  4. Preflight compression check (over 50% of context per the docs)
  5. Build API messages from history
       chat_completions  -> OpenAI format as-is
       codex_responses   -> Responses API input items
       anthropic_messages-> converted via the Anthropic adapter
  6. Inject ephemeral prompt layers (budget warnings, context pressure)
  7. Apply prompt-caching markers (Anthropic)
  8. Make an interruptible API call
  9. Parse response
       tool_calls -> execute, append results, loop to step 5
       text       -> persist session, flush memory if needed, return
```

The 50% preflight figure is the docs' description of the default compressor trigger (`compression.threshold` default 0.50 in `hermes_cli/config_defaults.py`). The Context Compression page describes several compaction gates (turn-start preflight, idle, pre-API pressure, post-tool) and a separate gateway session-hygiene pass at a fixed 85% of context; see section 14.

### Message format and alternation

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop)

Internal messages use `system`, `user`, `assistant` (with optional `tool_calls`) and `tool` roles. Reasoning content from thinking models is kept in `assistant_msg["reasoning"]` and shown via the `reasoning_callback` when configured. The loop enforces strict alternation: user and assistant turns alternate, an assistant message with tool calls is followed by tool results, never two assistant or two user messages in a row, and only `tool` messages may appear consecutively (parallel results). Providers reject malformed histories.

## Interruptible API calls

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [agent/turn_api_call.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/turn_api_call.py)

The HTTP request runs in a background thread while the main thread waits on the response, an interrupt event, or a timeout. On interrupt (new user message, `/stop`, signal) the API thread is abandoned, its response is discarded, and no partial response enters history.

## Tool execution

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [agent/tool_executor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/tool_executor.py) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py)

- One tool call runs directly; several run concurrently via a thread pool, with results reinserted in the original call order. Interactive tools (for example `clarify`) force sequential execution.
- Per call: resolve the handler from the registry, fire `pre_tool_call` plugin hooks, run the dangerous-command approval check for terminal commands, execute with `args` and `task_id`, fire `post_tool_call`, append the tool message. Middleware wraps this path (section 9).
- **Agent-level tools** are intercepted by `agent/tool_executor.py` before `handle_function_call()` because they need per-session agent state. `model_tools.py` defines `_AGENT_LOOP_TOOLS = {"todo_list", "memory", "session_search", "delegate_task"}`. The Agent Loop page's table names the first of these `todo`; the registered tool name per the tools reference and source is `todo_list`.

## Callback surfaces

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [agent/agent_init_params.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/agent_init_params.py)

| Callback | When fired | Used by |
|----------|-----------|---------|
| `tool_progress_callback` | Before/after each tool | CLI spinner, gateway progress |
| `thinking_callback` | Model starts/stops thinking | CLI indicator |
| `reasoning_callback` | Reasoning content returned | CLI/gateway reasoning display |
| `clarify_callback` | `clarify` tool called | CLI prompt, gateway interactive message |
| `step_callback` | After each complete agent turn | Gateway step tracking, ACP progress |
| `stream_delta_callback` | Each streamed token (when enabled) | CLI streaming display |
| `tool_gen_callback` | Tool call parsed from stream | CLI tool preview |
| `status_callback` | State changes | ACP status updates |

`agent_init_params.py` lists additional callbacks beyond this table (for example event, reaction and setup-beat callbacks); the page documents the eight above.

## Iteration budget and limits

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [agent/iteration_budget.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/iteration_budget.py) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

`IterationBudget` (`agent/iteration_budget.py`) is a thread-safe counter. `execute_code` iterations are refunded so programmatic tool calling does not consume budget. Each agent has its own budget; subagents get independent budgets, so total iterations across a parent and its children can exceed the parent's cap.

Defaults, as documented and as found in `hermes_cli/config_defaults.py` at this SHA:

| Setting | Value | Note |
|---------|-------|------|
| `agent.max_turns` | `null` = unlimited | The Configuration page and `config_defaults.py` agree. The Agent Loop page and the `cli-config.yaml.example` comment still say 500; treat those as stale. Positive integers cap; `none`/`unlimited`/`inf`/`0`/`-1` also mean unlimited. |
| `agent.budget_warning_ratio` | off | When set strictly between 0 and 1 with a finite `max_turns`, one model-visible checkpoint notice is appended to the latest tool result. |
| `delegation.max_iterations` | 250 | Per-child cap (older docs say 50; `config_defaults.py` and the delegation page say 250). |

## Fallback providers

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [agent/agent_init_fallback.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/agent_init_fallback.py)

When the primary model fails (429, 5xx, 401/403), the agent tries the `fallback_providers` list in config in order and continues the conversation on success; on 401/403 it first attempts a credential refresh. The docs also describe a Codex Responses turn that stalls on reasoning-only output (three consecutive continuations) failing over with reason `incomplete_response`. Auxiliary tasks (vision, compression, title generation) have their own fallback chains under the `auxiliary.*` config section.

## Compression and persistence

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Context Compression & Prompt Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage)

Compression (detail in section 14): memory is flushed first, middle turns are summarized, the last N messages are kept (`compression.protect_last_n`, default 20 in `config_defaults.py`), tool-call/result pairs are not split, and a new lineage session is created. After each turn messages are saved to the session store (SQLite, `hermes_state.py` plus split `hermes_state_*.py` modules; section 15), memory changes are flushed to `MEMORY.md`/`USER.md`, and the session can be resumed with `/resume` or `hermes chat --resume`.

## Key source files

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Architecture Overview](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)

| File | Purpose |
|------|---------|
| `run_agent.py` | `AIAgent` facade and public entry points |
| `agent/conversation_loop.py` | `run_conversation()` body |
| `agent/turn_*.py` | Turn phases (iteration prep, api call, api error, overflow, truncation, recovery) |
| `agent/tool_executor.py` | Tool execution and agent-level interception |
| `agent/system_prompt.py`, `agent/prompt_builder.py` | Prompt assembly |
| `agent/context_engine.py`, `agent/context_compressor.py` | Pluggable context engine and default compressor |
| `agent/prompt_caching.py` | Anthropic cache markers and metrics |
| `agent/auxiliary_client.py` | Auxiliary LLM client |
| `model_tools.py` | Tool schema collection and `handle_function_call()` |

---

# 9. Tool System — Registry, Toolsets, Dispatch

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

Hermes tools are self-registering functions grouped into toolsets and executed through a central registry and dispatch path. Primary files: `tools/registry.py`, `model_tools.py`, `toolsets.py`, `tools/terminal_tool.py`, `tools/environments/*`.

## Registration model

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

Each tool module calls `registry.register(...)` at import time. The docs' example signature (trimmed):

```python
registry.register(
    name="terminal",             # unique tool name used in API schemas
    toolset="terminal",          # toolset the tool belongs to
    schema={...},                # model-facing schema
    handler=handle_terminal,     # executes the call
    check_fn=check_terminal,     # optional availability check
    requires_env=["SOME_VAR"],   # optional, for UI display
    is_async=False,
    description="Run commands",  # ToolEntry metadata only
    emoji="💻",
)
```

`tools/registry.py` at this SHA also accepts `max_result_size_chars`, `dynamic_schema_overrides`, `override`, `scope` and `execution_timeout`. Key behaviors:

- Each call creates a `ToolEntry` in the singleton `ToolRegistry`, keyed by tool name.
- Registering a name that would shadow a tool in a *different* toolset is rejected unless the caller passes `override=True`. Plugin overrides of built-in tools additionally require the operator opt-in `plugins.entries.<plugin_id>.allow_tool_override: true` in `config.yaml`.
- `schema["description"]` is what the model sees; the separate `description=` argument only fills registry metadata.
- A non-dict schema or non-object `parameters` is rejected at registration time rather than at request time.

## Discovery

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py) · [tools/mcp_tool_discovery.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_discovery.py)

Importing `model_tools.py` calls `discover_builtin_tools()`, which scans `tools/*.py` using AST parsing for modules with a *top-level* `registry.register()` call and imports them (skipping `__init__.py`, `registry.py` and `mcp_tool.py`). New tool files are therefore picked up without a manual import list. Optional-tool import errors are logged without blocking other tools. After built-ins, two more sources are added:

1. **MCP tools** via `tools.mcp_tool_discovery.discover_mcp_tools()` (re-exported by the `tools.mcp_tool` facade), from the `mcp_servers` config (section 16).
2. **Plugin tools** via `hermes_cli.plugins.discover_plugins()` (section 20).

## Availability: `check_fn`

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

`registry.get_definitions()` runs each tool's optional `check_fn` and silently drops tools whose check returns false or raises. Typical checks: an API key present, a service reachable, a dependency importable (for example the browser harness for `browser_exec`). Results are cached per call so a shared check runs once. Handlers must return JSON strings and report problems as `{"error": "..."}` rather than raising (per Adding Tools).

## Toolset resolution

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py)

`model_tools.get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode)` produces the schema list:

1. If `enabled_toolsets` is given, only those toolsets (expanded via `resolve_toolset()`) are included.
2. If `disabled_toolsets` is given, start from everything and subtract.
3. If neither, all known toolsets.
4. The resolved names go to `registry.get_definitions()`, which applies `check_fn` filtering and returns OpenAI-format schemas.
5. Dynamic schema patching adjusts schemas such as `execute_code`, `browser_cdp` and `delegate_task` so they only reference tools that survived filtering.

Old `_tools`-suffixed names (`web_tools`, `terminal_tools`, `file_tools`, and similar) map to modern tool names through `_LEGACY_TOOLSET_MAP` in `model_tools.py`. Section 11 covers toolset kinds, platform presets, configuration and wildcards.

## Dispatch flow

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

```text
model response with tool_call
   -> agent loop (agent/conversation_loop.py via the AIAgent facade)
   -> agent-loop tools (todo_list, memory, session_search, delegate_task)
        handled directly by the loop, never by registry.dispatch
   -> model_tools.handle_function_call(name, args, task_id, ...)
        tool_request middleware -> pre_tool_call hooks / guardrails / approval
   -> registry.dispatch(name, args, **kwargs)
        look up ToolEntry; async handler bridged via _run_async(); sync called directly
        tool_execution middleware wraps the actual call
   -> JSON result string (or JSON error)
   -> post_tool_call hooks -> transform_tool_result hooks -> appended to history
```

Error wrapping happens at two levels (docs): `registry.dispatch()` converts handler exceptions into a JSON error ("Tool execution failed: ..."), and `handle_function_call()` adds an outer catch so the model always receives well-formed JSON. The four agent-loop tools keep their schemas in the registry, but `handle_function_call` returns an error if a call reaches it directly.

**Async bridging** (`_run_async()`): with no running loop (CLI) a persistent event loop is used so cached async clients stay alive; with a running loop (gateway) a disposable thread runs `asyncio.run()`; parallel worker threads use per-thread persistent loops.

## Hooks and middleware on the tool path

> Sources: [Observer Hooks](https://hermes-agent.nousresearch.com/docs/developer-guide/observer-hooks) · [Middleware](https://hermes-agent.nousresearch.com/docs/developer-guide/middleware) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py)

Plugins register in `register(ctx)`.

- **Observer hooks** (`ctx.register_hook`) are read-only telemetry: session lifecycle, turn-scoped `pre_llm_call`/`post_llm_call`, request-scoped `pre_api_request`/`post_api_request`/`api_request_error`, auxiliary-call hooks, tool hooks `pre_tool_call`/`post_tool_call`/`transform_tool_result`, human-input and approval hooks, and `subagent_start`/`subagent_stop`. Callbacks are fail-open and receive keyword arguments (accept `**kwargs`); each payload carries `telemetry_schema_version = "hermes.observer.v1"`. A few older hooks may change behavior: `pre_tool_call` may return a block or modify directive, `transform_tool_result` may replace a result, `transform_llm_output` may replace final text.
- **Middleware** (`ctx.register_middleware`) can change behavior. Kinds: `llm_request`, `llm_execution`, `tool_request`, `tool_execution` (schema version `hermes.middleware.v1`). Request kinds rewrite provider kwargs or tool args; execution kinds wrap the real call via `next_call`. Multiple execution middlewares nest in registration order; failures are fail-open.
- Order for a tool call (Middleware page): parse and coerce arguments, `tool_request` middleware, then availability checks, observer block directives, guardrails and approval against the effective arguments, then `tool_execution` middleware, then `post_tool_call`, then `transform_tool_result`.

## Dangerous-command approval

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/approval.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval.py)

The terminal tool checks commands against `DANGEROUS_PATTERNS` (regex/description pairs for recursive deletes, filesystem formatting, destructive SQL, system config overwrites, service manipulation, remote code piped to a shell, fork bombs, and similar) via `detect_dangerous_command()`. On a match, CLI mode prompts (approve, deny, or allow permanently), gateway mode sends an async approval request to the messaging platform, and "smart approval" can let an auxiliary model auto-approve low-risk matches. Approvals are tracked per session; "allow permanently" writes the pattern to `command_allowlist` in `config.yaml`. `tools/approval.py` is now split across several `approval_*.py` modules. Security detail is in section 25.

## Terminal and runtime environments

> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [Terminal Environment Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/terminal-environment-plugin)

Backends named in the docs: `local` (default), `docker`, `ssh`, `modal`, `daytona`, `vercel_sandbox`, and plugin backends registered via `ctx.register_terminal_environment_provider` (the user guide cites `singularity` and `sprites`). Select with `terminal.backend` in `config.yaml`; the user guide also documents `terminal.cwd`, `terminal.timeout` and container resource keys. Also supported: per-task cwd overrides, background processes, PTY mode and approval callbacks. Completed background process output is retained as redacted receipts under the profile's `logs/process-results/`, readable only by the owning session (docs: newest 64 results, up to 7 days).

## Tool Search (progressive disclosure)

> Sources: [Tool Search](https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-search) · [tools/tool_search.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/tool_search.py)

With many MCP servers or plugin tools, schemas can dominate context. Tool Search replaces deferred tools in the model-visible array with three bridge tools: `tool_search(queries, limit?)`, `tool_describe(names)` and `tool_call(calls)`. Hermes unwraps `tool_call` and dispatches the real tool, so hooks, guardrails and approvals see the underlying name.

```yaml
tools:
  tool_search:
    enabled: auto        # auto (default, currently an alias of on), on, or off
    threshold_pct: 5
    search_default_limit: 5
    max_search_limit: 25
    listing: auto
    listing_max_tokens: 4000
    defer: [computer_use, session_search, image_generate, todo_list, process_manage, cronjob_manage, manage_catalog]   # replaces the curated default; [] = nothing deferred
```

MCP and plugin tools are eligible automatically; core working-set tools stay eager unless named in `defer` (the shipped curated list includes cold built-ins and desktop helpers). Tiered disclosure (docs): tier 0 no deferred tools; tier 1 bridge plus a manifest of deferred tools; tier 2 bare bridge plus per-server summary for very large catalogs. The catalog is scoped to the session's granted toolsets, so a restricted subagent cannot discover tools outside its subset. The page also documents connector (remote gateway) tools reachable through the same bridge.

## Delegation: the `delegate_task` tool

> Sources: [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) · [Delegation & Parallel Work](https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns) · [Delegation Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/delegation-provider-plugin) · [tools/delegate_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/delegate_tool.py) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

`delegate_task` spawns child `AIAgent` instances with a fresh conversation, their own terminal session and inherited tool access; only the final summary returns to the parent. Per the docs, top-level model calls run in the **background** automatically: a handle comes back immediately and the result re-enters the conversation as a new message; orchestrator subagents wait for their own workers.

Key documented behavior:

- **Context.** Children know nothing of the parent conversation; the `goal` and `context` fields are their only input. A workspace's project context files (not `SOUL.md`) are embedded in the child's prompt. Optional `images` (up to 8) can be forwarded; optional `output_schema` makes the parent validate the child's JSON answer with one bounded correction turn.
- **Batch.** `tasks=[...]` runs children in parallel. By default one consolidated message is returned; `delegation.independent_completions: true` returns per completion unit (and advertises the `group` field).
- **Toolsets.** There is no model-facing `toolsets` parameter: children inherit the parent's enabled toolsets. Blocked for leaf children: `delegate_task`, `clarify`, `memory`, `send_message`, `cronjob`. Both roles keep `execute_code`.
- **Roles and depth.** `role="leaf"` (default) cannot delegate; `role="orchestrator"` retains delegation, gated by `delegation.max_spawn_depth` and `delegation.orchestrator_enabled`.
- **Control.** The parent can `list`, `steer` or `stop` its running children with the same tool (`action` argument, scoped to its own spawn tree). Live per-task transcripts are written under the Hermes home `cache/delegation/live/`.
- **Cancellation and durability.** `/stop`, or closing/resetting the owning session, ends background children. A process restart does not resume a running child; its attempt becomes `unknown`. Completion events are stored durably so undelivered results are restored. For work that must survive restarts the docs point to `cronjob_manage` or `terminal(background=True, notify_on_complete=True)`.
- **Finite runs.** In `-Q`/`--oneshot`/non-TTY chat the call joins its children in the same turn; messaging chats follow `delegation.gateway_mode` (`auto`, `inline`, `background`).
- **Worktrees.** `delegation.worktree_isolation: true` (off by default) gives each child its own git worktree on the local terminal backend.

Documented defaults and limits at this SHA (from the Delegation page and `config_defaults.py`):

| Key | Default | Meaning |
|-----|---------|---------|
| `delegation.max_concurrent_children` | 10 | Parallel children per batch and concurrent background units; floor 1, no ceiling. Over-limit batches return a tool error. |
| `delegation.max_spawn_depth` | 1 (flat) | Floor 1; the docs and source state no ceiling (a comment in `cli-config.yaml.example` says range 1 to 3; treat as stale). |
| `delegation.orchestrator_enabled` | true | Kill switch forcing every child to `leaf`. |
| `delegation.max_iterations` | 250 | Per-child tool-calling turns; exhausted children return `exit_reason: max_iterations`. |
| `delegation.child_timeout_seconds` | 0 | No wall-clock cap; a positive value bounds inactivity (floor 30s). A heartbeat staleness monitor interrupts frozen children regardless (docs: 450s idle between turns, 1200s inside a tool). |
| `delegation.oneshot_max_children` | 2 | Total children for a finite one-shot run; 0 = unlimited. |
| `delegation.independent_completions` | false | Per-unit result delivery. |
| `delegation.gateway_mode` | `auto` | `auto` / `inline` / `background`. |
| `delegation.worktree_isolation` | false | Per-child git worktree. |
| `delegation.model`, `provider`, `base_url`, `api_key`, `api_mode`, `request_overrides`, `reasoning_effort`, `fallback_providers` | empty (inherit parent) | Route children to a cheaper or different model. |

### Pluggable delegation: `delegation.backend`

> Sources: [Delegation Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/delegation-provider-plugin) · [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) · [agent/delegation_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/delegation_provider.py) · [plugins/delegation/default/plugin.yaml](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/delegation/default/plugin.yaml) · [plugin-catalog/advanced-delegation.yaml](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugin-catalog/advanced-delegation.yaml)

Release notes for v0.23.0 introduced pluggable delegation: `delegation.backend` selects how each task becomes a child run. Core keeps the subagent lifecycle (spawn, isolation, roles, depth, concurrency, background completion, stop fan-out, result assembly, `output_schema` validation, progress); a provider only answers "which route does this task take".

- `default` (the default value) follows the `delegation` section keys above, or inherits the parent's route. It is itself the bundled, always-loaded plugin `plugins/delegation/default/` and is the smallest complete example of the provider interface.
- The **official `advanced-delegation` plugin** (catalog entry in `plugin-catalog/advanced-delegation.yaml`, `requires_hermes: ">=0.22.5"`) adds two backends: `model_map` (named model routes the model picks per task) and `profile_routes` (each child runs as one of your Hermes profiles, with that profile's model, credentials, skills, `SOUL.md` and terminal policy). It also adds `hermes delegation config` and the `/delegation` slash command. While `model_map` or `profile_routes` is active, `delegation.model`/`delegation.provider` are inactive.
- Backend settings live under `delegation.backends.<backend name>`:

```yaml
delegation:
  backend: model_map
  backends:
    model_map:
      options:
        - name: fast
          description: Cheap and quick; lookups, summaries, simple edits.
          provider: openrouter
          model: google/gemini-3-flash-preview
        - name: deep
          description: Slow and careful; hard debugging and design work.
          provider: anthropic
          model: <your-model-id>
          reasoning_effort: high
      default: fast
    profile_routes:
      profiles:
        - name: coder
          description: Writes and refactors code; runs tests.
```

- **Fail closed.** A configured backend that is missing, disabled or unavailable makes `delegate_task` return a tool error naming the backend and the fix; it never silently runs on the default route. A task its backend cannot route is refused alone (`status: "error"`, `route.refused: true`) while sibling tasks run.
- **Switching.** `/delegation default` saves `delegation.backend` and applies from the next session (`/new`) because the per-task choices are part of the cached tool list; `/delegation <backend> --now` rebuilds the current chat's tools at the next turn (one prompt-cache miss). Without the plugin, `hermes config set delegation.backend default` restores the default.
- **Security note (docs).** Listing a profile under `profile_routes` lets this profile's subagents use that profile's credentials, credential pool, `SOUL.md` and terminal policy.
- **Writing a provider.** Subclass `DelegationProvider` (`agent/delegation_provider.py`): required `name` and `route_task(task, config)`; optional `task_parameters`, `description`, `is_available`. A route dict may carry `model`, `provider`, `base_url`, `api_key`, `api_mode`, `request_overrides`, `reasoning_effort`, `fallback_providers`, `service_tier`, `profile`, `label`. `task_parameters` and `description` must be deterministic for a given config because the `delegate_task` definition is part of the cached prompt. Register with `ctx.register_delegation_provider(...)` and enable via `hermes plugins enable <name>`. Providers also run under `plugins.isolation: host`.

### Public subagent lifecycle API

> Sources: [Public Subagent Lifecycle API](https://hermes-agent.nousresearch.com/docs/developer-guide/subagent-lifecycle-api) · [agent/subagent_lifecycle.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/subagent_lifecycle.py)

Plugins can launch and supervise fresh child sessions without importing `tools.delegate_tool` via `ctx.subagent_lifecycle` (`launch`, `status`, `wait`, `cancel`, `steer`, `result`, `reconnect`). The parent is resolved from the current agent turn; launching outside an active turn fails closed ("No active Hermes parent session"). States: `PENDING`, `STARTING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `INTERRUPTED`, `CANCEL_REQUESTED`, `CANCELLED`, `UNKNOWN`. `cancel` is cooperative, results are immutable and bounded to 32k characters, and after a process restart `reconnect` returns `RECONNECT_UNAVAILABLE` without starting a replacement. Requests are fail-closed: parent-broadening toolsets, per-tool blocks, working-directory overrides and per-launch timeouts are rejected. This API does not consult `delegation.backend`.

## Adding a built-in tool

> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/)

Docs position: default to plugins for custom tools; follow the built-in path only when shipping a core tool. A built-in tool touches two files: `tools/your_tool.py` (handler, schema, check function, `registry.register()`) and `toolsets.py` (add the tool name to `_HERMES_CORE_TOOLS` or define a toolset). Handlers receive `(args: dict, **kwargs)` (with `task_id` available through `kwargs`), return JSON strings, and may be `is_async=True`. The old "Step 3: add a discovery import" is no longer needed. A tool that needs an API key can be surfaced in the setup wizard through `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`. Section 22 covers custom tools in full.

---

# 10. Built-in Tools Reference

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

This section summarizes the tools the Built-in Tools Reference lists, organized by toolset. It deliberately gives **no total tool count**: the registry changes between releases, availability depends on platform, credentials and enabled toolsets, and plugins and MCP servers add more. The reference page is the authoritative, code-derived listing (including full descriptions and parameters); treat the tables below as an orientation aid. Descriptions are condensed paraphrases of that page at the pinned SHA.

## How to enumerate the tools you actually have

> Sources: [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [hermes_cli/subcommands/tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/subcommands/tools.py)

| Method | What it does |
|--------|--------------|
| [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) page | Everything the registry can offer, grouped by toolset. |
| `hermes tools` | Interactive per-platform tool configuration UI. `hermes tools --summary` prints enabled tools per platform and exits (documented). |
| `hermes tools list [--platform P]`, `hermes tools enable|disable NAME... [--platform P]` | Present in the CLI parser at this SHA (the CLI reference page documents only `--summary`). MCP tools use `server:tool` notation. |
| `/tools [list|disable|enable] [name...]` | In-session (CLI-only slash command). Disabling a tool removes it from the toolset and triggers a session reset. |
| `/toolsets` | CLI-only slash command listing available toolsets. |
| `hermes chat --list-tools` / `--list-toolsets` | Documented on the classic-CLI page as printing the catalog and exiting without a session. |

## Removed and relocated tools

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Browser Automation](https://hermes-agent.nousresearch.com/docs/user-guide/features/browser) · [Hermes Agent v0.23.0 release notes](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0)

- **Browser.** The old set of ten `browser_*` tools (`browser_navigate`, `browser_click`, and the rest) and the agent-browser integration were removed in v0.23.0. Browser automation now runs on `browser_exec`, with three companion tools (below). Homes that used Camofox get the `camofox` catalog plugin; prompts, skills or toolset lists naming the old tools should switch to `browser_exec`.
- **Moved to plugins.** The reference page states that Honcho tools (`honcho_*`), Home Assistant tools (`ha_*`, toolset `homeassistant`) and Spotify tools (toolset `spotify`) are no longer built in; they come from catalog plugins.
- **No agent-callable messaging send tool.** `toolsets.py` notes there is deliberately no `send_message` tool; outbound delivery is via cron delivery, the `hermes send` CLI and the gateway notifier (Tools & Toolsets page).

## `browser` toolset

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Browser Automation](https://hermes-agent.nousresearch.com/docs/user-guide/features/browser) · [tools/browser_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/browser_tool.py)

| Tool | Summary | Requires |
|------|---------|----------|
| `browser_exec` | Drives a real browser by running Python with pre-imported helpers (`new_tab`, `goto_url`, `wait_for_load`, `page_info`, `js`, `fill_input`, `click_at_xy`, `capture_screenshot`, `cdp`, `ensure_real_tab`). Browser state persists across calls; Python variables do not. `session` isolates concurrent work; `timeout_s` defaults to 300 (max 1800). | `terminal` toolset |
| `browser_cdp` | One raw Chrome DevTools Protocol command; escape hatch beyond the helpers. | CDP endpoint |
| `browser_dialog` | Accept or dismiss a native JavaScript dialog (`alert`/`confirm`/`prompt`/`beforeunload`). | CDP endpoint |
| `browser_vault` | Use saved logins, cards, addresses and one-time codes on the page without secrets entering the conversation (`list`, `fill`, `save_login`, `enter_code`, `unlock`). Deferred behind tool search by default. | none listed |

`browser_exec` executes model-written Python on your machine and disappears from sessions without the `terminal` toolset (Browser page). Browser providers (local Chromium by default, cloud providers, Lightpanda, your own browser via CDP) are chosen in `hermes tools` under Browser Automation.

## `terminal` and `file` toolsets

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [tools/file_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/file_tools.py)

| Tool | Toolset | Summary |
|------|---------|---------|
| `terminal` | `terminal` | Execute shell commands; filesystem persists between calls. `background=true` for long runs; `notify_on_complete=true` for an automatic completion notice; `heartbeat=N` (seconds, minimum 60) for periodic output notices. |
| `process_manage` | `terminal` | Manage background processes: `list`, `poll`, `log`, `wait`, `kill`, `write`, and more. (The user guide's examples call this `process`; the registered name in the reference and source is `process_manage`.) |
| `read_file` | `file` | Read text with line numbers and pagination; large reads are truncated on a line boundary and return a next offset; notebooks, Word and Excel files are supported per the page. |
| `write_file` | `file` | Replace a whole file; creates parent directories; for an existing file it requires a current full read first and refuses otherwise. Auto-runs syntax checks on linted languages. |
| `patch` | `file` | Targeted find-and-replace with fuzzy matching; returns a unified diff; runs syntax checks. |
| `search_files` | `file` | Ripgrep-backed content search or file-name search. |

## Web, media and analysis toolsets

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Tool Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway)

| Tool | Toolset | Summary | Requires (per page) |
|------|---------|---------|---------------------|
| `web_search` | `web` (also `search`) | Web search; default 5 results, optional `limit` 1 to 100. | One of the listed search-backend API keys (Exa, Parallel, Firecrawl, Tavily, Perplexity, Keenable). |
| `web_extract` | `web` | Page or PDF content as markdown/text; max 5 URLs per call; large pages return a head+tail window with the full text saved on disk. | Same backend keys. |
| `x_search` | `x_search` | Read-only public X discovery via xAI's built-in search. Off by default; opt in via `hermes tools`; schema registered only when xAI credentials exist. | xAI API key or xAI OAuth login |
| `vision_analyze` | `vision` | Image analysis; native pixels for vision-capable main models, auxiliary vision model otherwise. | none listed |
| `video_analyze` | `video` | Video analysis from a URL or path. Opt-in toolset. | none listed |
| `image_generate` | `image_gen` | Text-to-image or image edit via the user-configured backend (FAL.ai, OpenAI, OpenAI Codex auth, xAI, Krea); model is not agent-selectable. | Backend credential |
| `video_generate`, `xai_video_edit`, `xai_video_extend` | `video_gen` | Text-to-video and image-to-video through plugin backends (the page names xAI, FAL.ai, OpenRouter, DeepInfra); xAI edit/extend are provider-specific. Opt-in toolset (`--toolsets video_gen` or `hermes tools` Video Generation). | Active backend plugin plus credential |
| `text_to_speech` | `tts` | Text to speech; returns a `MEDIA:` path the platform delivers as audio. | none listed |

## Agent-orchestration, memory and scheduling toolsets

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)

| Tool | Toolset | Summary |
|------|---------|---------|
| `todo_list` | `todo` | Session task list; no parameters reads the list; items may nest via `parent`. Agent-loop tool. |
| `clarify` | `clarify` | Ask the user 1 to 5 questions in one call (single-select, multi-select or open-ended); returns a `responses` array and an `outcome` (`submitted`, `cancelled`, `timed_out`, `undelivered`). Subagents cannot use it. |
| `execute_code` | `code_execution` | Run a Python script that calls Hermes tools programmatically (for multi-call pipelines and filtering large outputs). |
| `delegate_task` | `delegation` | Spawn subagents (single `goal` or a `tasks` batch). See section 9. |
| `memory` | `memory` | Persistent cross-session memory; shows in the system prompt at the next session start. Agent-loop tool. |
| `session_search` | `session_search` | FTS5-backed search of past sessions or scrolling within one; four shapes (discovery, scroll, read, browse) with `after`/`before` bounds and `exclude_session_ids`. Agent-loop tool. |
| `cronjob_manage` | `cronjob` | Scheduled-task manager with actions `create`, `list`, `update`, `pause`, `resume`, `run`, `remove`; supports skill-backed jobs. Runs happen in fresh sessions. |
| `skills_list`, `skill_view`, `skill_manage` | `skills` | Browse, load and create/update/delete skills (new skills go under the Hermes home `skills/` directory). |

## `kanban` toolset

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Kanban Multi-Agent](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) · [tools/kanban_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/kanban_tools.py)

Per the reference page the kanban tools register when the agent is spawned by the kanban dispatcher (`HERMES_KANBAN_TASK` set) or when a profile/platform explicitly enables the `kanban` toolset. The `all`/`*` wildcard does **not** enable it.

| Tool | Summary |
|------|---------|
| `kanban_show` | Show the active task (title, description, comments, dependencies). |
| `kanban_complete` | Mark the task done with a structured handoff. |
| `kanban_block` | Block the task on a question for the user. |
| `kanban_request_review` | Hand implementation to a reviewer; moves the task to `review`; not a block. |
| `kanban_request_changes` | Reviewer verdict; routes the task back to the implementer. |
| `kanban_heartbeat` | Progress heartbeat during long operations. |
| `kanban_comment` | Add a comment without changing state. |
| `kanban_create` | Fan out child tasks. |
| `kanban_link` | Add a parent-to-child dependency edge. |
| `kanban_attach`, `kanban_attach_url`, `kanban_attachments` | Attach a file (inline bytes or URL, 25 MB cap) and list attachments. |
| `kanban_list`, `kanban_unblock` | Orchestrator-only board-routing tools; the page states they are hidden from dispatcher-spawned task workers. |

`delegate_task` children are not kanban run owners: their schema strips this toolset and runtime guards reject direct board mutation, even if parent `HERMES_KANBAN_*` variables are present (docs).

## Integration and desktop toolsets

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference) · [Tool Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-gateway) · [Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop)

| Toolset | Tools | Notes (per the reference page) |
|---------|-------|--------------------------------|
| `connections` | `manage_connections` | Managed connector accounts via the Nous gateway and local MCP catalog servers; present only when the portal has enabled connectors for the account. |
| `computer_use` | `computer_use` | Background desktop control via `cua-driver` on `$PATH` (macOS, Windows, Linux); does not steal cursor or focus. |
| `catalog` | `manage_catalog` | Search the plugin/skills catalog and install through an approval card; desktop-app sessions only; not included in `all`; deferred behind tool search by default. |
| `project` | `desktop_project` | Create, list or switch desktop Projects (desktop/GUI sessions only). |
| `desktop_ui` | `read_terminal`, `close_terminal`, `desktop_preview`, `drive_preview`, `annotate_preview`, `read_window_below`, `focus_pane`, `react_to_message`, `gui_tour`, `show_tip`, `set_theme` | Affordances acting on the Hermes desktop app; enabled for desktop-sourced sessions, absent from CLI, TUI, messaging and cron. |
| `discord`, `discord_admin` | `discord`, `discord_admin` | Registered on the `hermes-discord` toolset; need `DISCORD_BOT_TOKEN` (admin also needs bot permissions). |
| `feishu_doc`, `feishu_drive` | `feishu_doc_read`, `feishu_drive_add_comment`, `feishu_drive_list_comments`, `feishu_drive_list_comment_replies`, `feishu_drive_reply_comment` | Scoped to the Feishu document-comment handler. |
| `yuanbao` | `yb_query_group_info`, `yb_query_group_members`, `yb_send_dm`, `yb_search_sticker`, `yb_send_sticker` | Registered only on `hermes-yuanbao`. |
| (bridge) | `tool_search`, `tool_describe`, `tool_call` | Tool Search bridge tools (section 9); not listed in the reference page. |
| MCP | `mcp__<server>__<tool>` | Dynamic tools from MCP servers; naming per the reference page tip. Section 16. |

## Not covered here

> Sources: [Built-in Tools Reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference)

Full parameter descriptions, the long-form `clarify` per-surface behavior, desktop tour/tip details, and per-backend environment variables are on the reference page. Tools registered by plugins (for example memory-provider or video-gen backends) are documented with their plugins.

---

# 11. Toolsets Reference (All Platform & Core Toolsets)

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [Tools & Toolsets](https://hermes-agent.nousresearch.com/docs/user-guide/features/tools) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

Toolsets are named bundles of tools; they are the main way to control tool availability per platform, session or task. The docs describe three kinds: **core** (one logical group, such as `file`), **composite** (several toolsets combined for a scenario, such as `debugging`) and **platform** (a full configuration for a deployment context, such as `hermes-cli`). Every tool belongs to exactly one toolset. The list changes with releases; the Toolsets Reference page and `toolsets.py` are the sources of truth. No counts are asserted here.

## Core toolsets

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

| Toolset | Tools (see section 10) |
|---------|-------------------------|
| `web` | `web_search`, `web_extract` |
| `search` | `web_search` only |
| `browser` | `browser_exec`, `browser_cdp`, `browser_dialog`, `browser_vault` (`web_search` is deliberately not a member, so disabling `browser` cannot strip it) |
| `terminal` | `terminal`, `process_manage` |
| `file` | `read_file`, `write_file`, `patch`, `search_files` |
| `vision`, `video` | `vision_analyze`; `video_analyze` (video is opt-in) |
| `image_gen`, `video_gen` | `image_generate`; `video_generate`, `xai_video_edit`, `xai_video_extend` (video_gen is opt-in) |
| `tts` | `text_to_speech` |
| `skills` | `skills_list`, `skill_view`, `skill_manage` |
| `todo`, `memory`, `session_search` | `todo_list`; `memory`; `session_search` |
| `clarify`, `code_execution`, `delegation`, `cronjob` | `clarify`; `execute_code`; `delegate_task`; `cronjob_manage` |
| `computer_use`, `connections` | `computer_use`; `manage_connections` |
| `x_search` | `x_search` (off by default) |
| `kanban` | `kanban_*` tools (workflow-gated; see section 10) |
| `discord`, `discord_admin`, `feishu_doc`, `feishu_drive`, `yuanbao` | Platform-specific tools |
| `context_engine` | Runtime tools exposed by the active context-engine plugin (empty until a plugin populates it) |
| `desktop_ui`, `project`, `catalog` | Desktop-session affordances, enabled by the client for desktop-sourced sessions |

`toolsets.py` also defines `onboarding` and `bot_room` toolsets (first-run onboarding cards and Group Chat turn capabilities) that the Toolsets Reference page does not list.

## Composite toolsets

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

| Toolset | Composition (per the docs) |
|---------|----------------------------|
| `debugging` | `file` + `terminal` + `web` |
| `safe` | `web` + `vision` + `image_gen`, read-only research and media; no file writes, terminal or code execution |
| `coding` | A coding posture: file, terminal, search, web docs, skills, browser, todo, memory, session search, clarify, code execution, delegation and vision. In `toolsets.py` it is marked `posture=True` (a per-session posture selected for code workspaces, not auto-recovered into platform tool config). |

## Platform toolsets

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

Most messaging platforms use the same tool set as `hermes-cli` (the shared `_HERMES_CORE_TOOLS` list in `toolsets.py`; individual tools are still gated at runtime by their `check_fn`).

| Toolset | Difference from `hermes-cli` (per the docs) |
|---------|---------------------------------------------|
| `hermes-cli` | The default for interactive CLI sessions. |
| `hermes-acp` | Drops `clarify`, `cronjob`, `image_generate`, `text_to_speech`, `computer_use` and the kanban tools; coding focus for IDE integrations. |
| `hermes-api-server` | Drops `clarify`, `text_to_speech`, `computer_use` and the kanban tools. |
| `hermes-cron` | Same as `hermes-cli` (filtered by `hermes tools` platform config). |
| `hermes-telegram`, `-slack`, `-whatsapp`, `-signal`, `-matrix`, `-mattermost`, `-email`, `-sms`, `-bluebubbles`, `-dingtalk`, `-qqbot`, `-wecom`, `-wecom-callback`, `-weixin` | Same as `hermes-cli`. |
| `hermes-discord` | Adds `discord` and `discord_admin`. |
| `hermes-feishu` | Adds the five Feishu document and drive tools (used by the document-comment handler). |
| `hermes-yuanbao` | Adds the five `yb_*` tools. |
| `hermes-webhook` | Restricted safe subset: `web_search`, `web_extract`, `vision_analyze`, `clarify`. No terminal, file or browser access (webhook payloads are untrusted). |
| `hermes-gateway` | Internal orchestrator toolset: the union (via `includes`) of the `hermes-<platform>` toolsets, used when the gateway accepts any message source. |

Platforms provided by plugins get an implicit `hermes-<platform>` bundle (the shared core tools plus whatever the plugin registers under the platform name) that is not part of `hermes-gateway`. For example `hermes-homeassistant` resolves once the `homeassistant` catalog plugin is installed. Whether a given plugin platform exists depends on which plugins are loaded; see section 18 and section 20.

## Configuring toolsets

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

**Per session (CLI).**

```bash
hermes chat --toolsets web,file,terminal
hermes chat --toolsets debugging        # composite: expands to file + terminal + web
hermes chat --toolsets all              # everything (subject to the gating notes below)
```

**Per platform (`config.yaml`).** The example config documents the `platform_toolsets` key, accepting either a preset or a list of individual toolsets per platform. Unlisted platforms use their built-in default.

```yaml
platform_toolsets:
  cli: [hermes-cli]
  telegram: [web, terminal, file, todo]     # custom composition
  discord: [web, vision, skills, todo]      # restrictive example
```

The example file states that the old top-level `toolsets:` key is deprecated and ignored and that per-platform configuration lives in `platform_toolsets`. The Toolsets Reference page still shows a top-level `toolsets:` snippet, so prefer `platform_toolsets` (or `hermes tools`) and verify against your installed version.

**Interactive.** `hermes tools` (curses UI, per platform; persists to `config.yaml`; operates at the tool level, finer than toolsets), `hermes tools --summary`, and in-session `/tools list|enable|disable` and `/toolsets`. Disabled tools are filtered out even if their toolset is enabled.

**Global disable.** `agent.disabled_toolsets` removes toolsets everywhere after per-platform config is applied:

```yaml
agent:
  disabled_toolsets:
    - memory       # hide memory tools and the memory guidance
    - web
```

Removal is per tool, so disabling `search` also strips `web_search` from an enabled `web`. `hermes tools list` and `hermes doctor` flag an enabled toolset that loses tools this way.

**Default-off toolsets.** `hermes_cli/tools_config.py` defines `_DEFAULT_OFF_TOOLSETS` (at this SHA: `spotify`, `discord`, `discord_admin`, `video`, `video_gen`, `x_search`, `a2a`, `kanban`). When a platform's toolsets are inferred from a preset, these are dropped unless explicitly enabled. This matches the docs' statements that `video`, `video_gen`, `x_search` and `kanban` are opt-in.

## Dynamic toolsets

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [MCP](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

- **MCP server toolsets.** Each configured MCP server produces an `mcp-<server>` toolset at runtime containing the tools that server exposes; the bare server name also works as an alias. If a server is named like a built-in toolset (for example `browser` or `web`), that name resolves to the built-in tools plus the server's `mcp__<server>__*` tools; neither shadows the other. Per-tool toggling uses `server:tool` notation in `hermes tools enable|disable`.

```yaml
mcp_servers:
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
# creates the mcp-github toolset
```

- **Plugin toolsets.** Plugins can register their own toolsets via `ctx.register_tool()` during initialization; they appear alongside built-ins and can be enabled or disabled the same way. Delegation's public lifecycle API counts plugin-registered toolsets as known.
- **Custom toolsets.** The Toolsets Reference documents a `custom_toolsets:` block in `config.yaml` to define project bundles of existing toolsets. I did not find a reader for a `custom_toolsets` key in the Python source at this SHA (only `toolsets.create_custom_toolset()`, which tests call), so treat that config key as documented but not source-verified.

## Wildcards and gated tools

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

- `all` or `*` expands to every registered toolset (built-in, dynamic and plugin). In `toolsets.py`, toolsets that declare session `platforms` (such as `catalog` and `onboarding`) are excluded from `all`; the docs say a profile gets those only when its config names them.
- Some tools have an extra availability check beyond toolset membership and are **not** turned on by `all` alone:
  - *Capability-gated* tools (browser, `computer_use`, `code_execution`, Feishu, cronjob) appear only when their backend or credential prerequisite is configured.
  - *Workflow-gated* tools: the `kanban` toolset is deliberately opt-in. List it explicitly (or be a dispatcher-spawned worker with `HERMES_KANBAN_TASK` set); `hermes tools enable kanban --platform <p>` is the documented way to save it for a platform.
- Subagents inherit the parent's toolsets; a child's `delegation.inherit_mcp_toolsets` (default true per `cli-config.yaml.example`) keeps the parent's MCP toolsets when child toolsets are narrowed.

## Not covered here

> Sources: [Toolsets Reference](https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference)

Per-platform setup (tokens, allowlists) is in section 18 and the messaging docs. The mechanics of `hermes tools` provider selection (browser, web search, image and video backends) are on each feature's page. Always cross-check unusual toolset names against `toolsets.py` for your installed version.

---

# 12. Skills System — SKILL.md Format & Full Spec

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Working with Skills](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills) · [Curator](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator) · [tools/skill_manager_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skill_manager_tool.py) · [agent/skill_utils.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/skill_utils.py)

Skills are on-demand knowledge documents the agent loads when needed. Per the docs they follow a progressive-disclosure pattern to minimise token use and are compatible with the agentskills.io open standard. This section is a reference for the file format, discovery, loading, management and sharing of skills at upstream SHA `9f23c43`. Everything below is derived from the docs (and spot-checked against source constants); nothing here was executed. Skill counts are intentionally not asserted: use the catalog pages (linked below) or `hermes skills list` to enumerate what is installed in a given profile.

## 12.1 Skill or Tool? (decision guide)

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills)

| Make it a **Skill** when | Make it a **Tool** when |
|---|---|
| The capability is instructions plus shell commands plus existing tools | It needs end-to-end integration with API keys, auth flows or multi-component configuration |
| It wraps an external CLI or API the agent can call through `terminal` or `web_extract` | It needs custom processing logic that must execute precisely every time |
| No custom Python integration or baked-in key management is needed | It handles binary data, streaming or real-time events |
| Examples (per docs): arXiv search, git workflows, Docker management, PDF processing, email via CLI tools | Examples (per docs): browser automation, TTS, vision analysis |

Skills are described by the docs as the preferred way to add capabilities: no agent code changes, easy to share.

## 12.2 Where skills live

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Bundled Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) · [Official Optional Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/optional-skills-catalog) · [tools/skills_sync.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skills_sync.py)

`~/.hermes/skills/` is the primary directory and source of truth (profile-scoped: each profile has its own under its `HERMES_HOME`). On a fresh install, bundled skills are copied there from the repo's `skills/` tree; hub-installed and agent-created skills are also written there. The agent may modify or delete any skill (subject to locks and the write-approval gate described later).

```text
~/.hermes/skills/                  # single source of truth
├── mlops/                         # category directory
│   ├── axolotl/
│   │   ├── SKILL.md               # main instructions (required)
│   │   ├── references/            # additional docs
│   │   ├── templates/             # output formats
│   │   ├── scripts/               # helper scripts callable from the skill
│   │   ├── examples/              # referenced example outputs
│   │   └── assets/                # supplementary files
│   └── vllm/SKILL.md
├── devops/deploy-k8s/             # agent-created skill (SKILL.md + references/)
├── .hub/                          # Skills Hub state: lock.json, quarantine/, audit.log
└── .bundled_manifest              # tracks seeded bundled skills (name -> origin hash)
```

Related locations, all per the docs:

| Location | Role |
|---|---|
| `skills/` in the repo | Bundled skills, copied to the profile on install and on every `hermes update` (manifest-guarded) |
| `optional-skills/` in the repo | Official optional skills: ship with the repo, not active by default, installed with `hermes skills install official/<category>/<name>` |
| `~/.hermes/skills/.hub/` | Hub lock file (`lock.json`: source URL, content hash, scanner version, findings, timestamp), quarantine area and audit log |
| `~/.hermes/skills/.usage.json` | Sidecar holding usage telemetry and the per-skill `pinned` / `locked` flags (see Curator below) |
| `~/.hermes/skills/.archive/` | Where the curator archives long-unused skills (recoverable) |
| `~/.hermes/skill-bundles/<slug>.yaml` | Skill bundles (one slash command, several skills) |
| `~/.hermes/pending/skills/` | Staged skill writes when `skills.write_approval: true` |
| `<project-root>/.hermes/skills/`, `<project-root>/.agents/skills/` | Project-local skills (highest precedence; trust required) |

To see what ships, use the generated catalogs: the [Bundled Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) and the [Official Optional Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/optional-skills-catalog). The v0.23.0 release notes describe the built-in set as consolidated (for example, Word, PowerPoint and Excel skills merged into a single `microsoft-office` skill, and a number of niche skills moved to optional installs); old copies stay on disk until removed. The bundled catalog page notes that if a catalogued skill is missing from your profile it is still shipped, and can be restored with `hermes skills reset <name> --restore`.

## 12.3 SKILL.md format

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [tools/skill_manager_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skill_manager_tool.py)

A skill is a markdown file with YAML frontmatter. The documented full frontmatter (all optional keys shown) is:

```markdown
---
name: my-skill
description: Brief description (shown in skill search results)
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]          # optional; valid: macos, linux, windows; omit = all platforms
metadata:
  hermes:
    tags: [Category, Subcategory, Keywords]
    category: devops               # used in the work-with-skills guide example
    related_skills: [other-skill-name]
    requires_toolsets: [web]            # show only when these toolsets are active
    requires_tools: [web_search]        # show only when these tools are available
    fallback_for_toolsets: [browser]    # hide when these toolsets are active
    fallback_for_tools: [browser_exec]  # hide when these tools exist
    config:                             # non-secret config.yaml settings
      - key: my.setting
        description: "What this setting controls"
        default: "sensible-default"
        prompt: "Display prompt for setup"
    blueprint:                          # marks the skill a runnable automation
      schedule: "0 9 * * *"             # cron expr / "every 2h" / ISO timestamp
      deliver: origin                   # optional (default origin)
      prompt: "Task instruction for each run"   # optional
      no_agent: false                   # optional
required_environment_variables:         # secrets the skill needs
  - name: MY_API_KEY
    prompt: "Enter your API key"
    help: "Get one at https://example.com"
    required_for: "API access"
required_credential_files:              # file-based credentials (OAuth tokens etc.)
  - path: google_token.json             # relative to ~/.hermes/
    description: Google OAuth2 token (created by setup script)
---

# Skill Title

## When to Use
Trigger conditions: when should the agent load this skill?

## Quick Reference
Table of common commands or API calls.

## Procedure
Step-by-step instructions the agent follows.

## Pitfalls
Known failure modes and how to handle them.

## Verification
How the agent confirms it worked.
```

Notes from the docs and source:

- The recommended body order is `When to Use`, `Quick Reference`, `Procedure`, `Pitfalls`, `Verification`. The `/learn` flow is documented as authoring skills to "house standards" including a description of at most 60 characters; that is guidance for authored skills, not a documented hard limit on frontmatter.
- Hard limits enforced by `skill_manage` (source constants at this SHA): name up to 64 characters matching `^[a-z0-9][a-z0-9._-]*$`; description up to 1024 characters; `SKILL.md` content up to 100,000 characters; a supporting file up to 1 MiB; supporting-file writes are restricted to the `references`, `templates`, `scripts` and `assets` subdirectories.
- Legacy `prerequisites.env_vars` remains supported as a backward-compatible alias for `required_environment_variables`.
- `skill_manage` also runs an advisory linter (warn-only, never blocks): `incident-log-shape` (body dense in PR/issue numbers), `references-sprawl` (more than 60 reference files) and `oversized-body` (a `SKILL.md` body past roughly 24k characters, because `skill_view` loads the whole file into context for the rest of the session).
- Skill content is meant to capture "lessons, not logs": a generalizable rule plus a clause of why, attached to the step it affects, with always-on rules in `SKILL.md` and bulky detail under `references/`.

### 12.3.1 Platform restriction

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills)

`platforms: [macos | linux | windows]` restricts a skill by operating system. On an incompatible platform the skill is hidden from the system prompt, `skills_list()` and slash commands. Omitted or empty means it loads everywhere.

### 12.3.2 Conditional activation

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)

| Field | Behaviour (per docs) |
|---|---|
| `requires_toolsets` | Hidden when any listed toolset is not available |
| `requires_tools` | Hidden when any listed tool is not available |
| `fallback_for_toolsets` | Hidden when any listed toolset is available (shown only when it is missing) |
| `fallback_for_tools` | Hidden when any listed tool is available |

Typical use: a curl-based search skill with `fallback_for_tools: [web_search]` appears only when `web_search` is disabled, so the agent still has a search path. Skills with no conditional fields are always shown. Note the two docs pages phrase the toolset example differently (one uses `fallback_for_toolsets: [web]`, the other `[browser]`); both describe the same semantics.

### 12.3.3 Secrets and config on load

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

- **Secrets** (`required_environment_variables`): a missing value does not hide the skill. In the local CLI, Hermes prompts securely when the skill is loaded; you can skip. Messaging surfaces never collect secrets in chat and instead point to `hermes setup` or `~/.hermes/.env`. Declared variables that are set are passed through to `execute_code` and `terminal` sandboxes (including remote backends such as Docker and Modal); the raw value is never exposed to the model. For non-skill variables use `terminal.env_passthrough`.
- **Credential files** (`required_credential_files`): paths relative to `~/.hermes/`. Missing files trigger `setup_needed`; existing files are mounted read-only into Docker, synced into Modal sandboxes, and available directly on the local backend.
- **Config settings** (`metadata.hermes.config`): non-secret settings stored in `config.yaml` under `skills.config.<key>`. `hermes config migrate` prompts for unconfigured ones, `hermes config show` lists them, and on load the resolved values are appended to the skill message as a `[Skill config (from ~/.hermes/config.yaml): ...]` block. They can be set by hand: `hermes config set skills.config.myplugin.path ~/my-data`. Each entry takes `key` and `description` (required) plus optional `default` and `prompt`.

### 12.3.4 Template variables and inline shell

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

When a skill loads, the activation message exposes `[Skill directory: /abs/path]` and substitutes two tokens anywhere in the body: `${HERMES_SKILL_DIR}` (absolute skill directory) and `${HERMES_SESSION_ID}` (active session id, left as-is if none). Disable globally with `skills.template_vars: false` (default `true`).

Inline shell snippets written as `` !`cmd` `` are **off by default** (`skills.inline_shell: false`). When enabled, each snippet runs on the host with the skill directory as working directory and its stdout is inlined (output capped at 4000 characters, per-snippet timeout `skills.inline_shell_timeout`, default 10 seconds; failures become a short `[inline-shell error: ...]` marker). Community hub installs never auto-execute snippets even with the flag on; bundled, trusted and your own skills do.

### 12.3.5 Media delivery directives

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)

A bare absolute media path in a response is auto-detected by the gateway, stripped from visible text and delivered natively. `[[audio_as_voice]]` promotes audio to voice-message bubbles on platforms that support them. `[[as_document]]` (all-or-nothing per response) forces every extracted media path to be delivered as a document attachment instead of a recompressed inline image; the directive is stripped before delivery.

## 12.4 Loading model: progressive disclosure

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Working with Skills](https://hermes-agent.nousresearch.com/docs/guides/work-with-skills) · [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly)

```text
Level 0: skills_list()           -> [{name, description, category}, ...]   (docs: ~3k tokens)
Level 1: skill_view(name)        -> full SKILL.md content + metadata
Level 2: skill_view(name, path)  -> one reference/template/script file
```

The compact skills index is part of the system prompt's **volatile** tier (the prompt-assembly docs list "skills index" there). Because the system prompt is held fixed to keep the provider prompt cache valid, skills installed, created or removed mid-conversation do not edit it: the next user message carries a one-time note naming the added and removed skills, and the system prompt itself picks up the change at the next compaction. `/reload-skills` refreshes `/skill-name` slash commands. `hermes sessions refresh-prompts` (v0.23.0 release notes) lets continuing sessions pick up edited skills without `/new`, at the cost of one cache miss per session.

Skills can also be pinned fully loaded in every new session with `skills.auto_load` (a list of names, resolved once when the prompt is built; `-s <name>` preloads on the command line). Such skills live in the system prompt and are never compacted.

### 12.4.1 Slash commands, stacking and bundles

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [agent/skill_commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/skill_commands.py)

Every installed skill is a slash command (`/excalidraw sketch the request flow`); the bare name loads it and lets the agent ask what you need. Leading `/skill` tokens stack (up to 5 are loaded; parsing stops at the first token that is not an installed skill, so file paths are not swallowed). Plan mode is now a built-in `/plan` command, not a skill.

**Skill bundles** are YAML files in `~/.hermes/skill-bundles/<slug>.yaml` that group skills under one command:

```yaml
name: backend-dev
description: Backend feature work — review, test, PR workflow.
skills:
  - github-code-review
  - test-driven-development
instruction: |
  Always start by writing failing tests, then implement.
```

`skills` is required and non-empty; `name`, `description` and `instruction` are optional. Manage with `hermes bundles list|show|create|delete|reload` (and `/bundles` in chat). A bundle wins over a same-named skill; missing skills are skipped with a note to the agent; bundles generate a user message at invocation and do not mutate the system prompt, so the prompt cache is not invalidated.

### 12.4.2 `/learn`

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)

`/learn <anything you can describe>` (directory, URL, the workflow just demonstrated, pasted notes, or a large document) builds a standards-guided prompt and hands it to the agent as a normal turn; the agent gathers material with its existing tools and saves the result with `skill_manage`. Large sources become an expansive knowledge-base skill: a lean `SKILL.md` with core mental models and an index, plus one distilled file per chapter or topic under `references/`. The write-approval gate (below) applies. The dashboard Skills page has a "Learn a skill" panel that composes the same request.

## 12.5 Discovery: directories, precedence, project skills

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

```yaml
skills:
  external_dirs:            # extra read/scan dirs; ~ and ${VAR} supported; missing paths skipped silently
    - ~/.agents/skills
    - ${SKILLS_REPO}/skills
  create_dir: ""            # where skill_manage creates NEW skills (empty = profile-local dir)
  project_discovery: true   # scan <repo>/.hermes/skills and <repo>/.agents/skills (trusted repos only)
  trusted_project_dirs: []  # managed by `hermes skills trust` / `untrust`
  auto_load: []
  template_vars: true
  inline_shell: false
  inline_shell_timeout: 10
  write_approval: false
```

**Precedence** (same on every surface: index, `skills_list`, `skill_view`, slash commands, `-s` preload, cron): `project -> local (~/.hermes/skills/) -> skills.create_dir -> external_dirs`. A shadowed copy is hidden and logged. Inside one directory, two different skills sharing a name are never guessed between: both are listed under their relative paths and loading the bare name fails with an "Ambiguous skill name" error. Plugin skills use `plugin:skill` names and never collide (and are not listed in the system prompt).

**Update-in-place vs create-local:** new agent-created skills go to the local dir (or `skills.create_dir`); existing skills are modified where they are found, including under `external_dirs`. External directories are not a write-protection boundary: use filesystem permissions if they must stay read-only.

**Project-local skills** are the highest tier but load only after you run `hermes skills trust` in (or pass the path of) the repo; the banner shows an untrusted-skills notice otherwise. Every project skill is scanned with the hub scanner before it enters the index; a `dangerous` verdict quarantines it (hidden from index, `skills_list` and slash commands, and refused by name). The curator never modifies project skills. Cron and other non-interactive surfaces inherit the interactive trust decision and never prompt; the project root comes from the surface's working directory.

## 12.6 Agent-managed skills (`skill_manage`)

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [tools/skill_manager_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skill_manager_tool.py)

The agent can create, update and delete its own skills: its "procedural memory". The system prompt asks it to record non-trivial workflows (multi-step procedures worth repeating, error recoveries, user corrections). Memory holds small durable facts always in context; skills hold longer procedures loaded on demand.

| Action | Use for | Key params |
|---|---|---|
| `create` | New skill from scratch | `name`, `content` (full SKILL.md), optional `category` |
| `patch` | Targeted fix (preferred; most token-efficient) | `name`, `old_string`, `new_string` |
| `patch` with `content` | Whole-file structural rewrite (`edit` is the legacy alias) | `name`, `content` |
| `delete` | Remove a skill | `name` |
| `write_file` | Add/update a supporting file | `name`, `file_path`, `file_content` |
| `remove_file` | Remove a supporting file | `name`, `file_path` |

Each action's text slot is exclusive (`content` / `new_string` / `file_content`); a batch carrying the wrong slot for its action is rejected before any operation in the batch is applied.

### 12.6.1 Write gating: approval, guard, pin and lock

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Curator](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

Four independent controls exist:

| Control | What it does |
|---|---|
| `skills.write_approval: true` | Every `skill_manage` write (foreground or background review) is **staged** under `~/.hermes/pending/skills/` and reviewed with `/skills pending`, `/skills diff <id>`, `/skills approve <id\|all>`, `/skills reject <id\|all>`, `/skills approval on\|off`. Default `false` |
| `skills.guard_agent_created` | A content scanner (dangerous-pattern heuristics) over agent-written skills; not an approval gate (the Configuration page documents its default and semantics) |
| `hermes curator pin <skill>` | Agent-created skills only: skips curator auto-transitions and makes `skill_manage delete` refuse; edits/patches still work |
| `hermes curator lock <skill>` | Stops the agent changing the skill at all while it stays readable and auto-loadable. Refused for `skill_manage` (patch/edit/write_file/remove_file/delete, single and batch), for `write_file`/`patch` on any path inside the skill dir, and for `terminal` commands that write there; the curator will not archive it; the agent cannot unlock it; applies in cron, subagents and background review, and even under `--yolo` |

The lock is stored as `"locked": true` in `~/.hermes/skills/.usage.json` (per profile) and is a policy check, not a sandbox: a script writing through its own logic or `execute_code` is not inspected. It does not block `hermes update` / `hermes skills update` from refreshing an unmodified bundled or hub skill, and your own editor edits always work. v0.23.0 release notes introduce the lock.

## 12.7 Skills Hub and trust

> Sources: [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [tools/skills_hub.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skills_hub.py) · [tools/skills_guard.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/skills_guard.py)

```text
hermes skills browse [--source official]      # official first
hermes skills search <query> [--source skills-sh|well-known|github|...]
hermes skills inspect <id>                    # preview, upstream metadata
hermes skills install <id> [--force] [--name N] [--category C]
hermes skills list [--source hub] | check | update [name] [--force] | audit | uninstall <name>
hermes skills reset <name> [--restore] [--yes]
hermes skills publish skills/my-skill --to github --repo owner/repo
hermes skills snapshot export setup.json
hermes skills tap add|list|remove owner/repo
hermes skills trust|untrust [path]            # project-local skills
hermes skills opt-out [--remove] | opt-in [--sync]   # bundled-skill seeding
```

Every one of these also works in chat as `/skills ...`.

| Source id | Example identifier | Notes |
|---|---|---|
| `official` | `official/security/1password` | Optional skills shipped in the repo |
| `skills-sh` | `skills-sh/<owner>/<repo>/<skill>` | skills.sh directory |
| `well-known` | `well-known:https://<site>/.well-known/skills/<name>` | Sites publishing `/.well-known/skills/index.json` |
| `url` | `https://example.com/SKILL.md` | Direct URL plus explicitly referenced files under `references/`, `templates/`, `scripts/`, `assets/`, `examples/`; name from frontmatter, then URL slug, then prompt, then `--name` |
| `github` | `owner/repo/path` | Direct repo/path installs and custom taps; several vendor repos are default taps |
| `clawhub`, `lobehub`, `browse-sh` | source-specific | Community/marketplace integrations; `browse-sh` is `community` trust |

**Trust levels** (docs table): `builtin` (ships with Hermes, always trusted), `official` (`optional-skills/`, built-in trust, no third-party warning), `trusted` (a listed set of vendor/registry repos, more permissive policy), `community` (everything else; non-dangerous findings can be overridden with `--force`, `dangerous` verdicts stay blocked). `--force` never overrides a `dangerous` verdict. Taps added with `hermes skills tap add` default to `community` trust; raising trust requires a core change to `TRUSTED_REPOS` in `tools/skills_guard.py`.

**Security scanning:** every hub install runs a scanner (data exfiltration, prompt injection, destructive commands, supply-chain signals). Third-party URL/GitHub installs scan the complete quarantined bundle and record source URL, content hash, scanner version, findings and timestamp in `skills/.hub/lock.json`. An advisory NVIDIA SkillEvaluator Tier 1 pass can additionally run when its optional binaries are on PATH (off with `skills.tier1_advisory: false`); it never blocks.

**Update lifecycle:** `check`/`update` compare the stored source identifier and upstream bundle hash. Locally edited hub skills are skipped unless `--force`. GitHub API calls are rate limited unauthenticated; the docs recommend setting `GITHUB_TOKEN`.

**Bundled-skill sync:** each `hermes update` copies bundled skills in, comparing your copy's hash to the manifest's origin hash. Unchanged copies are updated; changed copies are treated as user-modified and skipped. `hermes skills reset <name>` re-baselines the manifest entry (your copy kept); `--restore` also replaces it with the bundled version. Cache directories such as `__pycache__` do not count as modification.

**Blank slate:** `hermes profile create <name> --no-skills`, or `hermes skills opt-out [--remove]` / `opt-in --sync` on an existing profile, write or remove a `.no-bundled-skills` marker that stops installer/update/sync seeding. `--remove` deletes only unmodified bundled skills.

**Publishing a tap:** a GitHub repo with `skills/<name>/SKILL.md` directories (path configurable per tap in `~/.hermes/skills/.hub/taps.json`; names starting with `.` or `_` are ignored). A tap may ship `skills.sh.json` `groupings` to label categories in the public Skills Hub. A single skill can also be installed straight from any public repo with `hermes skills install owner/repo/skills/<name>`.

## 12.8 Curator (background maintenance)

> Sources: [Curator](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator) · [agent/curator.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/curator.py)

The curator tracks per-skill usage and maintains **agent-created** skills so narrow near-duplicates do not pile up. It is triggered by an inactivity check (CLI start, gateway housekeeping, the `hermes serve` maintenance timer), not by cron. It spawns a background fork of the agent with its own prompt cache. It never auto-deletes: the worst outcome is archival to `~/.hermes/skills/.archive/`. Hub-installed skills are always exempt; bundled skills are only archived (never patched) when you set `curator.prune_builtins: true`.

```yaml
curator:
  enabled: true
  interval_hours: 168          # 7 days between runs
  min_idle_hours: 2
  stale_after_days: 14         # active -> stale
  archive_after_days: 30       # stale -> archived
  consolidate: false           # LLM umbrella-building pass: opt-in (prune-only by default)
  prune_builtins: false        # opt in to archiving unused bundled skills
```

The first run is deferred by one full interval on a fresh install. A run has two phases: (1) deterministic transitions with no LLM (skills named in any cron job's `skills:` list and pinned skills are skipped; never-used skills get a grace floor), and (2) an optional LLM consolidation pass (`consolidate: true` or `hermes curator run --consolidate`) that can patch, merge into umbrellas or archive, treating a skill as a whole package including `references/`/`scripts/`. The review model is the `auxiliary.curator` slot (configurable through `hermes model`). A tar.gz snapshot of the skill tree is taken before consolidation (`hermes curator rollback`). Notable CLI verbs: `status`, `run [--dry-run|--background|--consolidate]`, `pause|resume`, `pin|unpin`, `lock|unlock`, `adopt`, `archive|restore|list-archived`, `prune`, `ledger`, `backup|rollback`, `purge`. See the Curator page for the full list.

## 12.9 Automation blueprints (skills that are also automations)

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Automation Blueprints Catalog](https://hermes-agent.nousresearch.com/docs/reference/automation-blueprints-catalog) · [Automation Blueprints (guide)](https://hermes-agent.nousresearch.com/docs/guides/automation-blueprints) · [Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/)

A **blueprint** is an ordinary skill that adds a `metadata.hermes.blueprint` block (see the frontmatter above). Because it is a skill, search, install, scanning, taps and `hermes skills publish` apply unchanged. Installing a blueprint skill registers a **suggested cron job** rather than scheduling it; scheduling is always opt-in via `/suggestions` (`/suggestions accept N`, `/suggestions dismiss N`, `/suggestions catalog` for curated starters). Suggestions come from four sources (`catalog`, `blueprint`, `usage`, `integration`); accepting one calls the same cron job-creation path as the `cronjob_manage` tool, so there is no second engine. A blueprint loaded by a cron job (`hermes cron create --skill <name> ...`) can be exported back to a SKILL.md and published.

The catalog page documents the user-facing side: `/blueprint <name>` (CLI, TUI, messengers) asks for required values one question at a time and schedules after confirmation, `name=value` pairs can be passed inline, and the dashboard/desktop Cron page has a Blueprints tab. Plugins can add namespaced blueprints (`/blueprint teamtools:standup`) that cannot replace built-ins and are listed only in profiles where the plugin is enabled; plugin authors register them with `ctx.register_automation_blueprint()`. Separately, the **Automation Blueprints guide** is a collection of copy-paste recipes (cron-scheduled, GitHub webhook and API-triggered automations); it is documentation, not a different mechanism.

## 12.10 What this section does not cover

> Sources: [Bundled Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) · [Official Optional Skills Catalog](https://hermes-agent.nousresearch.com/docs/reference/optional-skills-catalog)

Per-skill documentation (each bundled and optional skill has its own generated page under the catalogs), the complete `hermes skills` flag reference (see the CLI reference), per-platform skill enablement (running bare `hermes skills` opens an interactive TUI for it, per the Working with Skills guide), and the Curator usage-telemetry and report format (see the Curator page).

---

# 13. Memory System — MEMORY.md, USER.md, Providers

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) · [Honcho Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho) · [Memory Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [tools/memory_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/memory_tool.py) · [tools/memory_tool_store.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/memory_tool_store.py)

Hermes has two memory layers: **built-in curated memory** (two small bounded files, always injected into the system prompt) and an optional **external memory provider** (one at a time, additive). A third recall mechanism, `session_search` over the SQLite session store, is separate from both. This section covers the first two; building a provider plugin is covered in section 21 and the developer guide linked above.

## 13.1 Built-in memory files

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [tools/memory_tool_store.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/memory_tool_store.py)

| File | Purpose | Default char limit |
|---|---|---|
| `MEMORY.md` | The agent's personal notes: environment facts, conventions, things learned | 2,200 (docs: about 800 tokens) |
| `USER.md` | User profile: preferences, communication style, expectations | 1,375 (docs: about 500 tokens) |

Both live in the **profile's `memories/` directory**, i.e. `~/.hermes/memories/` for the default profile and `~/.hermes/profiles/<name>/memories/` for a named profile (source: `get_memory_dir()` returns `get_hermes_home() / "memories"`). Note: one developer-guide page (Prompt Assembly) abbreviates these as `~/.hermes/MEMORY.md`; the user guide and the source use the `memories/` subdirectory. Entries are separated by a `§` delimiter line (source: `ENTRY_DELIMITER = "\n§\n"`) and may be multiline. Memory is scoped per profile; the docs warn against pointing two agent processes at one Hermes home, since both would write into the same files. If agents must share memory, use an external provider.

Memory does not auto-compact. When a write would exceed a limit the tool returns an error with the current entries and the overflow size; the agent then reissues a single `operations` batch that removes or shortens (via `patch`) entries and adds the new one; the limit is checked only on the batch's final result. Exact duplicates are rejected as "no duplicate added". Entries are scanned for prompt-injection, credential-exfiltration and invisible-Unicode patterns before being accepted, since they enter the system prompt.

## 13.2 Memory snapshot in the system prompt

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly)

At session start the entries are rendered into the system prompt as a **frozen snapshot**. Per the Prompt Assembly page this lands in the volatile tier, alongside the skills index, user profile snapshot and external-provider block:

```text
══════════════════════════════════════════════
MEMORY (your personal notes) [67% — 1,474/2,200 chars]
══════════════════════════════════════════════
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
```

The header shows the store, usage percentage and character counts. The snapshot is captured once and not changed mid-session, which preserves the provider prefix cache. Writes are persisted to disk immediately and tool responses show live state, but the system prompt only reflects them at the next session start or at a rebuild path (the Prompt Assembly page names compression-triggered rebuild; the compression docs say in-place compaction rebuilds the system prompt). Practical consequence stated by the docs: on messaging gateways a chat is one long-lived session, so run `/new` at natural boundaries so fresh memory is re-read.

## 13.3 The `memory` tool

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [tools/memory_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/memory_tool.py)

| Action | Behaviour |
|---|---|
| `add` | Add an entry |
| `patch` | Replace one span (`old_text`) inside an existing entry, keeping the rest (`content` is the replacement). `old_text` must match exactly one entry and occur once within it |
| `remove` | Remove the entry matched by a unique substring `old_text` |

There is no `read` action (content arrives in the system prompt) and no `replace` action (a `replace` call returns an error pointing to `patch`). Every call needs a top-level `target`: `"memory"` or `"user"`; a batch uses `operations` under that single target:

```json
{"target": "user", "operations": [{"action": "add", "content": "User prefers metric units"}]}
```

A target inside an operation that differs from the top-level one is an error returned before anything is written or staged.

### 13.3.1 Configuration and write approval

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  write_approval: false          # true = require approval before any memory write
  prefetch_spill_enabled: false  # external recall returned in full unless enabled
  provider: ""                   # empty = built-in only (see 13.5)
```

Source defaults also include `memory.nudge_interval: 10` (comment: periodic built-in memory review; `0` when an external provider auto-extracts); the user-guide page only references this key in passing. Disabling both `memory_enabled` and `user_profile_enabled` removes the `memory` tool and its prompt guidance but leaves any external provider's tools active; putting `memory` under `agent.disabled_toolsets` is the heavier switch that hides external provider tools too. With only one store disabled, the tool's schema advertises only the enabled target and writes to the other are rejected.

With `memory.write_approval: true`, interactive CLI foreground writes prompt inline; everything else (messaging, scripts, the background self-improvement review) is **staged**: review with `/memory pending`, `/memory approve <id|all>`, `/memory reject <id|all>`, `/memory approval on|off`. A staged `patch`/`remove` records the exact entry it targets and is refused if that entry changed in the meantime.

### 13.3.2 Background review

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)

After a turn, a background self-improvement review fork may save memory or update skills. Controls documented on the Memory page:

| Key | Default | Meaning |
|---|---|---|
| `display.memory_notifications` | `on` | `off` / `on` / `verbose` chat notice (e.g. a "Memory updated" line); per-platform override under `display.platforms.<platform>.memory_notifications` |
| `auxiliary.background_review.provider` / `model` | auto (main model) | Route the review to a cheaper model; a different model replays a compact digest instead of the full transcript |
| `auxiliary.background_review.enabled` | `true` | `false` skips automatic post-turn forks (manual `/refine` still works) |
| `auxiliary.background_review.max_input_tokens` | derived (75% of window, capped at 600,000; 120,000 fallback) | Cap on summed replayed input tokens per review; `<= 0` unlimited |
| `auxiliary.background_review.extra_tools` | `[]` | Names of extra safe tools allowed in the review fork |
| `auxiliary.background_review.defer` | `auto` | `auto` queues reviews on the managed local llama-server until idle; `never` spawns immediately |
| `auxiliary.background_review.defer_max_age_s` | `1800` | Run a queued review anyway after this long |

A same-model review always inherits the parent's reasoning effort so its prompt-cache prefix stays byte-identical.

### 13.3.3 Related: session search and the learning journey

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)

`session_search` queries the SQLite store (`~/.hermes/state.db`, FTS5) and returns actual messages with no LLM summarisation; the docs contrast it with memory (memory is a fixed-cost always-in-context fact sheet, session search is unlimited and on-demand). `hermes journey` (aliases `hermes learning`, `hermes memory-graph`; `/journey` in TUI/Desktop) draws a timeline of saved skills and memory entries and offers `list`, `delete <node>` and `edit <node>`.

## 13.4 Troubleshooting "it forgot"

> Sources: [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory)

The docs' checklist, in order: (1) confirm the model actually called the `memory` tool and the entry is in the file (weak tool-calling models sometimes claim a save they never made); (2) check the write was not staged by `write_approval`; (3) check you are reading the same profile's `memories/` directory; (4) check memory is enabled; (5) remember the snapshot is frozen, so start a new session after the write. Environment variables in `.env` are not memory; for a location needed on every run of a recurring task, a skill is often the better home.

## 13.5 External memory providers

> Sources: [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) · [Honcho Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho) · [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [agent/memory_manager.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_manager.py)

Only **one** external provider can be active at a time, and it runs **alongside** (never instead of) built-in memory. The docs state that Hermes ships two in-tree provider plugins (RetainDB and ByteRover) and that others (Honcho, Hindsight, Supermemory, Mem0, OpenViking, Holographic) are installed from the plugin catalog; the same page notes RetainDB and ByteRover are scheduled to leave core on 15 October 2026 (provider name, settings, data directory and tools unchanged). Memori is described as an externally supplied integration. Providers that stop shipping with Hermes are auto-installed from the catalog by `hermes update` for profiles that name them.

```bash
hermes plugins install honcho   # catalog providers: install first
hermes memory setup             # interactive picker + configuration
hermes memory status            # what is active
hermes memory off               # disable the external provider
```

```yaml
memory:
  provider: honcho   # or retaindb | byterover | hindsight | supermemory | mem0 | openviking | holographic
```

When a provider is active Hermes (per docs): injects provider context into the system prompt; prefetches relevant memories before each turn in the background; syncs each completed turn to the provider; extracts on session end where supported; mirrors built-in memory writes; and adds provider-specific tools. All data handed to a provider passes through the same secret scrub as chat/cron delivery (even with `security.redact_secrets: false`); shapeless credentials are not detected.

| Provider | Storage | Tools (per docs) | Distinguishing feature (per docs) |
|---|---|---|---|
| Honcho (catalog) | Cloud or self-hosted | `honcho_profile`, `honcho_search`, `honcho_context`, `honcho_reasoning`, `honcho_conclude` | Dialectic user modelling, session-scoped context, peer cards |
| OpenViking (catalog) | Self-hosted | six `viking_*` tools | Filesystem-style hierarchy, tiered retrieval |
| Mem0 (catalog) | Platform, self-hosted dashboard or OSS in-process | four tools | Server-side LLM fact extraction |
| Hindsight (catalog) | Cloud or local | `hindsight_retain`, `hindsight_recall`, `hindsight_reflect` | Knowledge graph and cross-memory synthesis |
| Holographic (catalog) | Local SQLite | `fact_store`, `fact_feedback` | FTS5, trust scoring, HRR algebra |
| RetainDB (bundled) | Cloud | ten `retaindb_*` tools | Hybrid search, delta compression |
| ByteRover (bundled) | Local, optional cloud sync | `brv_query`, `brv_curate`, `brv_status` | Pre-compression extraction; tree at `$HERMES_HOME/byterover/` |
| Supermemory (catalog) | Cloud or self-hosted | four tools | Context fencing, session graph ingest |
| Memori | Cloud | five `memori_*` tools | Tool-aware memory, structured recall |

Isolation is per profile: local-storage providers use `$HERMES_HOME/` paths; config-file providers keep config under `$HERMES_HOME/`; cloud providers derive profile-scoped project names; env-var providers read each profile's `.env`. Provider pricing and licensing change; consult the Memory Providers page rather than this table.

### 13.5.1 Honcho specifics

> Sources: [Honcho Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho) · [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)

Honcho is maintained by its vendor and installed from the catalog; config lives in `$HERMES_HOME/honcho.json` (profile-local) with fallback resolution `$HERMES_HOME/honcho.json` then `~/.hermes/honcho.json` then `~/.honcho/config.json`. Architecture per the docs: a base layer (session summary plus representation plus peer card) refreshed on `contextCadence`, plus a dialectic LLM supplement refreshed on `dialecticCadence` that picks cold-start or warm prompts by whether base context exists. Three independent cost/depth knobs: `contextCadence` (default 1), `dialecticCadence` (default 2), `dialecticDepth` (default 1, clamped 1 to 3). Other documented keys include `recallMode` (`hybrid` / `context` / `tools`), `writeFrequency`, `observationMode`, `sessionStrategy`, and gateway identity mapping (`pinUserPeer`, `userPeerAliases`, `runtimePeerPrefix`). See the Honcho page for the full key table.

### 13.5.2 Provider lifecycle and compression interplay

> Sources: [Memory Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py)

A provider implements the `MemoryProvider` ABC (`agent/memory_provider.py`). Required: `name`, `is_available()` (no network calls), `initialize(session_id, **kwargs)`, `get_tool_schemas()`, `handle_tool_call()`, `get_config_schema()`, `save_config()`. Optional hooks per the docs: `system_prompt_block()`, `prefetch(query)`, `queue_prefetch(query)`, `sync_turn(user, assistant)`, `on_session_end(messages)`, `on_pre_compress(messages)`, `on_memory_write(action, target, content, metadata)`, `shutdown()`, plus profile-created/renamed hooks. Two details relevant here:

- **Pre-compression:** `on_pre_compress()` is best-effort by default. A provider can opt into the fail-closed checkpoint contract (`pre_compress_checkpoint_api_version = 2`) and operators then set `compression.checkpoint_required: true`; compression is blocked (`BLOCKED_MISSING_PREREQUISITE`) unless the checkpoint succeeded. The docs state no bundled provider advertises v2, and enabling the gate without one blocks every compression attempt.
- **Oversized recall:** prefetch results are returned in full up to a safety ceiling of 10 times `hooks.output_spill.max_chars` (at least 100,000 characters); `memory.prefetch_spill_enabled: true` opts into head/tail spill previews.

Full plugin-development detail (directory vs packaged layout, threading contract, config schema, CLI commands) belongs to section 21 and the [Memory Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) page.

## 13.6 Memory around compression and session end

> Sources: [Agent Loop Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/agent-loop) · [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

The Agent Loop page states that before compression, memory is flushed to disk first; the `/compress` slash command description likewise reads "flush memories + summarize". This page also still describes compression as creating a child session lineage id; the compression page documents that the default (`compression.in_place: true`) keeps one session id (see section 14), so treat the lineage statement as describing the legacy `in_place: false` path. External providers see `on_pre_compress` before the lossy rewrite and `on_session_end` at session boundaries.

---

# 14. Context Compression & Prompt Caching

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [Micro-compaction](https://hermes-agent.nousresearch.com/docs/developer-guide/micro-compaction) · [Context Engine Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/context-engine-plugin) · [Context References](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-references) · [agent/context_compressor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/context_compressor.py) · [agent/prompt_caching.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_caching.py)

Hermes manages long conversations with layered mechanisms: a gateway pre-agent hygiene pass, the in-loop compressor (default engine), optional opt-in variants (prepared compaction, micro-compaction, proactive tool-result prune, provider-native compaction) and an optional replacement engine via plugin. Separately, prompt caching reduces input cost on Claude-style providers. This section documents what the docs state; no behaviour here was executed.

## 14.1 Architecture: two layers plus a pluggable engine

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [gateway/run_turn_hygiene.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_turn_hygiene.py) · [agent/context_engine.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/context_engine.py)

```text
                   +---------------------------+
 Incoming message  |  Gateway session hygiene  |  fixed 85% of context window
 ----------------> |  (pre-agent; real tokens  |  safety net for sessions that
                   |   if known, else rough)   |  grew between turns
                   +-------------+-------------+
                                 |
                                 v
                   +---------------------------+
                   |  Agent context engine     |  default 50% of context
                   |  (ContextCompressor, in   |  (configurable; <512K windows
                   |   the tool loop, real     |   floored at 75%)
                   |   usage-anchored counts)  |
                   +---------------------------+
```

- **Gateway session hygiene** (`gateway/run_turn_hygiene.py`): threshold fixed at 85% of the model context length; runs only when `len(history) >= 4` and compression is enabled. Token source order: last API-reported tokens, then the usage anchor persisted on the session row, then a rough character estimate. It exists because a gateway session can grow overnight; setting it equal to the agent's threshold caused premature compression every turn.
- **Agent ContextCompressor** (`agent/context_compressor.py`): the primary system, run inside the tool loop with provider-reported counts.
- **Pluggable engine:** both sit behind the `ContextEngine` ABC (`agent/context_engine.py`); the built-in compressor is the default.

The docs also state that intermediate "context pressure" warnings were removed: compression fires at the configured threshold with no prior warning step.

## 14.2 Configuration reference

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

All settings are read from `config.yaml` under `compression:`. The summary model is configured under `auxiliary.compression`.

```yaml
compression:
  enabled: true
  threshold: 0.50             # fraction of the MAIN model's context window
  # threshold_tokens: 200000  # optional absolute cap: fires at the lower of ratio and this
  # model_thresholds:         # per-model overrides; substring match, longest key wins
  #   "glm-5.2": 0.40
  #   "openai-codex:astra": 0.85   # "<provider>:<substring>" scopes to one route
  target_ratio: 0.20          # legacy tail budget = threshold_tokens x target_ratio
  tail_mode: lean             # lean | legacy
  protect_last_n: 20
  min_tail_user_messages: 1
  protect_first_n: 3
  max_attempts: 3
  in_place: true              # compact on the same session id (default true)
  prepare_ahead: false        # opt-in background summary preparation
  idle_compact_after_seconds: 0   # 0 = off
  checkpoint_required: false  # fail-closed with a checkpoint-capable memory provider
  progress_notices: false     # routine compression notices on chat platforms
  codex_gpt55_autoraise: true
  codex_app_server_auto: native   # native | hermes | off
  codex_responses_native: false   # opt-in server-side compaction (narrow model/route gate)
  codex_responses_compact_threshold: null
  proactive_prune_tokens: 0   # 0 = off; e.g. 48000 enables the no-LLM tool-result prune
  proactive_prune_min_result_chars: 8000
  proactive_prune_min_reclaim_tokens: 4096
  micro_compact: false        # opt-in (see 14.6)

auxiliary:
  compression:
    model: null               # summary model (null = auto)
    provider: auto
    base_url: null
```

| Key | Default | Notes (per docs) |
|---|---|---|
| `threshold` | `0.50` | Fires at prompt tokens >= `threshold x context_length`; models with windows below 512K are floored at `0.75` (raise-only) |
| `threshold_tokens` | `null` | Optional absolute cap, clamped to the window |
| `model_thresholds` | `{}` | Re-resolved on every `/model` switch; the small-window floor still applies on top |
| `target_ratio` | `0.20` (0.10 to 0.80) | Tail budget in `legacy` mode only |
| `tail_mode` | `lean` | `lean` keeps a clamped tail of 2.5% of the window (10K floor, 25K cap, never above 20% of the window) and carries continuity in the summary; `legacy` keeps a `target_ratio`-sized verbatim tail |
| `protect_last_n` | `20` | Minimum recent messages, honoured only up to a small count floor; tail is otherwise token-bounded and capped at 20% of the window |
| `min_tail_user_messages` | `1` | Real user messages guaranteed to survive in the tail (wins over the token budget) |
| `protect_first_n` | `3` | Non-system head messages kept verbatim (`0` pins only the system prompt, summary and tail). The docs table labels this "hardcoded", while the config example and source defaults expose it as a key; trust the key |
| `in_place` | `true` | One durable session id; pre-compaction rows soft-archived (`active=0, compacted=1`), still searchable. `false` restores the legacy rotating path with `parent_session_id` links |
| `prepare_ahead` | `false` | See 14.5 |
| `idle_compact_after_seconds` | `0` | Compact up front when a session resumes after this many idle seconds; skipped when context is already at or below `threshold x target_ratio` |
| `max_attempts` | `3` | Retry rounds before "max compression attempts reached" (validated >= 1, hard cap 10 per the example file) |
| `checkpoint_required` | `false` | See 13.5.2 |
| `progress_notices` | `false` | Routine automatic compression is silent on chat platforms by design; failures and manual `/compress` feedback always show |

Keys visible in source defaults (`hermes_cli/config_defaults.py`) but not described in the developer-guide compression page (the page only cites some indirectly): `hygiene_hard_message_limit` (5000, gateway force-compress by message count), `hygiene_timeout_seconds` (30), `hygiene_total_ceiling_seconds` (600), `hygiene_failure_cooldown_seconds` (300), `hygiene_max_turn_hold_seconds` (10, also in the example config), `context_timeout_seconds` (120), `context_total_ceiling_seconds` (600) and `abort_on_summary_failure` (false: when true an auto-compression whose summary fails aborts instead of dropping the middle). Treat these as source-level facts at this SHA.

## 14.3 Triggering, token accounting and failure handling

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/usage_anchor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/usage_anchor.py)

`threshold_tokens` derives from the **main** model's context window, never the auxiliary summary model's. For a 200K-window model at defaults the docs give: `threshold_tokens = 100,000`, legacy `tail_token_budget = 20,000`, `max_summary_tokens = min(200,000 x 0.05, 12,000) = 10,000`.

Every compaction gate (turn-start preflight, idle, pre-API pressure, post-tool) consults the **usage anchor** (`agent/usage_anchor.py`): the provider's last prompt and completion counts plus a rough estimate of only the messages appended since. It is keyed by a content fingerprint, persisted on the session row (so `--resume` and gateway restarts restore it), and cleared by compaction or reset. With no anchor (first request, rewind), a whole-context rough estimate over threshold waits one request for provider evidence; the docs state this is not an exact-count-only policy and several estimate fallbacks remain (post-tool usage-less fallback, micro-compaction, tail sizing). Images are priced at a per-image cost learned from provider usage (default 1,500 before the first vision turn); opaque provider blobs count as zero.

**Failure cooldown:** a failed or stalled summary arms a per-session cooldown (escalating 60s, 300s, 900s, never shorter than `compression.context_timeout_seconds`, persisted in `state.db`) that defers threshold-triggered compaction. Exceptions that run a real attempt anyway: manual `/compress`, the same-turn fallback-chain retry, repeated stall (ends in a deterministic fallback summary), and provider-proven context-length overflow (bounded by `max_compression_attempts`). If the summary provider is overloaded and the one-shot main-model retry also fails, compress aborts and keeps the transcript; after three consecutive such aborts it commits a deterministic fallback summary instead. `abort_on_summary_failure: true` hard-aborts every failed summary.

**Telemetry:** every `compress_context` call logs one content-free INFO line (`context compression attempt telemetry: {...}`) with `trigger_source` (`manual`, `idle`, `turn_start_threshold`, `engine_preflight`, `pre_api`, `post_tool`, `gateway_hygiene`, `overflow`), `route`, `commit_status`, `method` (`llm_summary`, `aux_fallback_main`, `deterministic_fallback`, `provider`, `none`) and before/after message and token counts.

## 14.4 The compaction algorithm (`ContextCompressor.compress()`)

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/context_compressor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/context_compressor.py)

The docs describe four phases:

```text
Phase 1  prune        old tool results (>200 chars) outside the tail ->
                      "[Old tool output cleared to save context space]"  (no LLM call)
Phase 2  boundaries   head = protect_first_n; tail = walk back by token budget;
                      align so tool_call/tool_result groups are never split
Phase 3  summarize    middle turns -> ONE auxiliary call (task="compression")
                      with a structured template
Phase 4  assemble     head + summary message + tail; _sanitize_tool_pairs()
                      removes orphan results / stubs missing results
```

```text
[ system + first exchange ] [ ---- middle: SUMMARIZED ---- ] [ tail: recent, verbatim ]
        protect_first_n                                       token budget (<=20% of window)
```

- **Phase 1 details:** a tool round the model has not yet answered keeps its results verbatim in the tail. A round exceeding 20% of the input budget can be summarised and its older images retired.
- **Loaded skills across a compaction:** skill content a pass prunes or summarises leaves `[SKILL_PRUNED: ... reload with skill_view(name='...')]`; the summary lists them under `## Pruned Skills` and ends with a `## Skill Safety Rule` instructing a reload before reliance. Skills preloaded with `-s` or `skills.auto_load` live in the system prompt and are never compacted.
- **Phase 3 template** (sections): Goal; Constraints & Preferences; Progress (Done / In Progress / Blocked); Key Decisions; Relevant Files; Next Steps; Critical Context. The v0.23.0 release notes add that summaries list uncertain claims on `UNVERIFIED:` lines. Summary budget: `content_tokens x 0.20`, minimum 2,000, maximum `min(context_length x 0.05, 12,000)`. In `lean` mode there is exactly one auxiliary LLM call per attempt; continuity comes from an identifier-preserving session log, a regex-extracted anchor index (PR numbers, SHAs, paths, error strings), every real user message quoted verbatim, and a `session_search` recovery pointer.
- **Summary model warning:** the summary model's context window must be at least as large as the main model's; if the call fails with a context-length error the middle may be dropped without a summary (the most common cause of degraded compaction per the docs). The docs also describe auxiliary feasibility clamps: a smaller auxiliary model can lower the live trigger.
- **Phase 4:** head messages (a compaction note is appended to the system prompt on the first compression), the summary message (role chosen to avoid consecutive same-role messages), and the unmodified tail.
- **Iterative re-compression:** later compactions pass the previous summary (`_previous_summary`) to the model with instructions to update it; "In Progress" items move to "Done" and obsolete information is removed.

**In-place compaction (default):** with `compression.in_place: true` a compaction rewrites the live message list on the same session id, rebuilds the system prompt, and soft-archives the pre-compaction rows. The `session:compress` event carries `in_place` and `old_session_id` (empty string in in-place mode). Manual `/compress` honours the same setting; `/compress here [N]` keeps the last N exchanges verbatim and `/compress <focus>` narrows what the summary preserves.

## 14.5 Prepared compaction (opt-in)

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/prepared_compaction.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prepared_compaction.py)

Off by default (`compression.prepare_ahead: false`, read at agent construction). When the last provider-reported prompt count is within 12% of the trigger, the turn loop starts one background pass that plans the same window as `compress()` and runs the same `_generate_summary` on copies, leaving the live transcript, system prompt and prompt cache untouched. At the next automatic threshold compaction, `compress()` splices the finished summary if it still covers the same messages (same session, unchanged head boundary, same message structure vouched by each message's durable `message_uid`); later messages stay as tail. If a pass is still running and would fit, compaction waits for it instead of re-summarising. Manual `/compress`, focus compaction, overflow recovery and compactions where a memory provider contributes summary context run inline. Plugin context engines, Codex app-server sessions, native Responses compaction and micro-compaction are not prepared. State is in memory per compressor, so it only helps agents that live across turns. The v0.23.0 release notes describe the same feature.

## 14.6 Micro-compaction (opt-in)

> Sources: [Micro-compaction](https://hermes-agent.nousresearch.com/docs/developer-guide/micro-compaction) · [agent/micro_compaction.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/micro_compaction.py)

An alternative that amortises the cost: after each completed turn, `finalize_turn` asks the compressor to absorb **one** exchange (an assistant message plus its tool results, up to the next user message) into a single rolling summary. It is **off by default**.

```yaml
compression:
  micro_compact: true                       # default false
  micro_compact_every_n_turns: 1            # cadence; values < 1 clamp to 1
  micro_compact_defrag_threshold_tokens: 2000
```

Documented properties: user messages are never absorbed (they stay verbatim); the head and the token-budgeted tail are never touched; only the newest summary marker is kept (earlier ones are redundant and dropped, with adjacent user turns joined into a model-only message); when the rolling summary exceeds the defrag threshold the next pass re-summarises the summary itself (no messages absorbed, cursor unmoved); a cursor recovers from the last marker after a restart; each pass also calls `archive_and_compact` to soft-archive the replaced rows in the session DB; if the same exchange fails to summarise three times in a row the cursor skips past it. Failures are logged and swallowed (the turn completes unchanged). Batch compaction still fires if the window fills. Each pass uses the `auxiliary.compression` model and runs at the end of the turn, so a small fast non-reasoning model is advised. Each pass rewrites already-sent history and therefore **breaks the prompt-cache prefix every turn** (tune with `micro_compact_every_n_turns`); the docs frame this as trading one cost for another and recommend measuring with the per-pass telemetry line and `scripts/micro_compaction_report.py`. When `compression.checkpoint_required` is armed, micro-compaction is forced off.

## 14.7 Other compaction paths

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

- **Proactive tool-result prune** (`proactive_prune_tokens`, default `0` = off): a deterministic no-LLM prune of old tool results (dedupes identical results, summarises oversized ones, truncates large tool-call arguments) that runs independently of `threshold`, useful on 512K/1M windows where the ratio rarely fires. A prune only commits if it reclaims at least `proactive_prune_min_reclaim_tokens` (default 4096) so cache breaks stay episodic. Built-in compressor only.
- **Codex app-server threads:** the codex agent owns the thread context, so Hermes does not rewrite its transcript; `/compress` calls the app-server's own compaction and `compression.codex_app_server_auto` (`native` default, `hermes`, `off`) controls automatic behaviour.
- **Native Responses compaction** (`codex_responses_native`, default `false`): opt-in server-side compaction for a narrow gate (gpt-5.6-family on the direct OpenAI API or Codex subscription, plus `gpt-6-astra` on official Codex OAuth). The local compressor remains armed as the fallback; the native threshold is clamped about 8K tokens below the local trigger, and a structured provider rejection disables native compaction for the session.
- **Codex OAuth threshold autoraise** (`codex_gpt55_autoraise`, default `true`): on the ChatGPT Codex OAuth route the trigger is raised to 85% for the affected model families because the backend caps their window; only that exact route is affected. The Codex `-900k` picker variants are opt-in aliases that keep the global threshold.
- **Bedrock context cache:** `context_length_cache.yaml` under the Hermes home persists provider-confirmed limits with a documented precedence (explicit override, confirmed limit, revalidated legacy entry, static table).

## 14.8 Pluggable context engine

> Sources: [Context Engine Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/context-engine-plugin) · [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/context_engine.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/context_engine.py)

```yaml
context:
  engine: "compressor"    # default built-in; a plugin engine is selected by its name, e.g. "lcm"
```

Resolution order: `plugins/context_engine/<name>/`, then the general plugin system (`register_context_engine()`), then the built-in compressor. Plugin engines are never auto-activated, and only one engine is active. Required methods: `name`, `update_from_response(usage)`, `should_compress(prompt_tokens)`, `compress(messages, current_tokens, focus_topic)`; the engine must maintain `last_prompt_tokens`, `last_completion_tokens`, `last_total_tokens`, `threshold_tokens`, `context_length`, `compression_count`. Optional: `on_session_start/end/reset`, `update_model`, `get_tool_schemas`/`handle_tool_call` (engine-provided tools such as an `lcm_grep` example), `should_compress_preflight`, `get_status`, `clone_for_agent`, and two orthogonal per-turn hooks, `select_context()` (request-only, before cache control; fail-open) and `on_turn_complete()` (best-effort observation). Every persisted message carries a stable `message_uid` for engines that keep their own per-message state. The `compression.*` block belongs to the built-in compressor, with the explicit exception of `compression.model_thresholds`, which is part of the engine contract. When `checkpoint_required` is armed, the host calls `on_pre_compress()` before a plugin engine's `compress()`.

Lifecycle (docs): init, `on_session_start()`, `update_from_response()` after each API call, `should_compress()` each turn, `compress()` when true, `on_session_end()` at session boundaries; `/new` calls `on_session_reset()`.

## 14.9 Prompt caching

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [agent/prompt_caching.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_caching.py)

Prompt caching is automatically enabled when the model is a Claude model (detected by name) and the provider supports `cache_control` (the compression page names the native Anthropic API and OpenRouter; the Configuration page also lists Nous Portal). Hermes uses the `system_and_3` strategy within Anthropic's limit of four breakpoints:

```text
Breakpoint 1: system prompt                      (stable across all turns)
Breakpoint 2: 3rd-to-last non-system message  -+
Breakpoint 3: 2nd-to-last non-system message   +- rolling window
Breakpoint 4: last non-system message         -+
```

`apply_anthropic_cache_control()` deep-copies messages and injects markers `{"type": "ephemeral"}` or, for the one-hour tier, `{"type": "ephemeral", "ttl": "1h"}`. Placement by content type: string content becomes a one-element text list carrying the marker; list content gets it on the last element; empty content gets `msg["cache_control"]`; tool messages get `msg["cache_control"]` on native Anthropic only.

```yaml
prompt_caching:
  cache_ttl: "5m"    # "5m" | "1h" | "auto"
```

Per the Configuration page, `5m` and `1h` are sent as-is and other values are ignored; source defaults add that `auto` picks `1h` for human-paced sources (cli, tui, desktop, messaging) and `5m` for machine-paced sources (subagent, cron, oneshot, webhook, kanban, api, tool, batch), and that falsy values (false, null, "off", "disabled", "no", "none") disable caching. The example config file lists only the two explicit tiers. Providers with their own caps (for example a 5-minute maximum) still clamp to what the upstream allows. The CLI prints caching status at startup.

Cache-aware design rules from the docs: keep the system prompt stable mid-conversation (compression appends its note only on the first compaction); cache hits require an exact prefix, so inserting or removing middle messages invalidates everything after; after a compaction the compressed region's cache is invalidated while the system-prompt breakpoint survives and the rolling window re-establishes within one or two turns; and any mid-conversation model or credential change (a `/model` switch, fallback activation, credential-pool rotation) means the next request gets no cache hits, since provider caches are scoped to model and account. The v0.23.0 release notes record further cache work (shared cache key for repeated cron fires; post-compaction, goal and heartbeat turns no longer rewriting the cached prefix).

## 14.10 Context references (`@` syntax) and compression

> Sources: [Context References](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-references)

Context references inject content into a message and append it under an `--- Attached Context ---` section: `@file:path`, `@file:path:10-25` (1-indexed inclusive range), `@folder:path`, `@diff`, `@staged`, `@git:N` (1 to 10 commits) and `@url:...`. Size limits: a soft limit of 25% of the context length adds a warning, a hard limit of 50% refuses expansion and returns the original message; folder listings are capped at 200 entries. Sensitive paths (SSH keys and config, shell profiles, credential files, the Hermes `.env`, and the `.ssh`, `.aws`, `.gnupg`, `.kube` and `skills/.hub` directories) are blocked, paths outside the workspace are rejected, and binary files are refused. The feature is primarily a CLI feature: the gateway does not expand `@` syntax on messaging platforms. Interaction with compression: injected content counts toward context, and after compaction it is summarised rather than preserved verbatim, so prefer line ranges for large files.

## 14.11 Not covered here

> Sources: [Context Compression and Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [Context Engine Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/context-engine-plugin)

Context-file loading (`AGENTS.md`, `SOUL.md`; see sections 6 and 7), the session database layout that backs soft-archived rows (section 15), the shared-metrics counters (`hermes.compression.count`, `hermes.prepared_compaction.count`, `hermes.compaction` marks) documented in the NeMo Relay shared-metrics page, and the full context-engine contract tests. Consult those pages directly.

---

# 15. Session Storage — SQLite Schema

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [State DB Recovery](https://hermes-agent.nousresearch.com/docs/developer-guide/state-db-recovery) · [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py) · [hermes_state_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_schema.py)

Hermes keeps conversations in one SQLite database per profile, `state.db`. This section describes what the docs and source say about the file, its tables, search indexes, migration policy, contention handling and the operator commands around it. Everything here is derived from the docs and source at the pinned commit; no database was opened or migrated while writing it.

Version scoping note: the developer guide states "Current schema version: **31**", but `SCHEMA_VERSION` in `hermes_state_common.py` at this commit is **32**. The source comment says v32 is a version bump only (no data migration) recording that new tables and three columns were declared under the v31 shape. Treat the source value as authoritative and the docs table as lagging by one.

## Where the database lives and who writes it

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions) · [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py)

- Default path: `get_hermes_home() / "state.db"`. That is `~/.hermes/state.db` for the default profile and `<default root>/profiles/<name>/state.db` for a named profile; wherever `HERMES_HOME` points wins. Callers should not hard-code the path. SQLite also creates `state.db-wal` and `state.db-shm` beside it.
- Each named profile has its own database. A process that serves several profiles (multiplex gateway, `hermes serve`) must bind the right profile home when it opens a handle; a subprocess that drops `HERMES_HOME` can fall back to the default profile.
- Writer model: the docs state that the profile's gateway is the only writer of canonical sessions. `hermes chat`, the TUI, local Desktop, the API server, ACP, cron, Bot Chat and Kanban workers are clients that admit input to the gateway's session authority over `/api/ws`; out-of-process managed workers write through the owner (`RuntimeSessionStore` RPC) rather than opening `state.db` themselves for transcript writes. Read-only consumers (dashboard session browsing, `hermes sessions list`) open the file read-only.
- Legacy: per-session JSONL transcripts are no longer written or read; the docs call leftover `*.jsonl` files safe to delete once the session exists in `state.db`. `~/.hermes/sessions/sessions.json` is a legacy mirror of the `gateway_routing` table, written while `gateway.write_sessions_json` is true (the default), and is not the session list.
- Not stored here: batch-runner and RL trajectories are separate systems.

## Source layout

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py) · [hermes_state_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_schema.py) · [hermes_state_fts.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_fts.py) · [hermes_state_search.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_search.py) · [hermes_state_wal.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_wal.py) · [hermes_state_errors.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_errors.py) · [hermes_state_repair.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_repair.py) · [hermes_state_portability.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_portability.py) · [hermes_state_maintenance.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_maintenance.py)

The old single-file `hermes_state.py` is now a facade (it holds `SessionDB`, the write-contention constants and the connection setup) plus a large family of `hermes_state_*.py` siblings at the repository root. Examples verified in the tree:

| Module | Role (from module names/docstrings/docs) |
|---|---|
| `hermes_state_common.py` | `SCHEMA_SQL`, `SCHEMA_VERSION`, FTS trigger names, maintenance thresholds |
| `hermes_state_schema.py` | Applies the schema and the version-gated migration chain |
| `hermes_state_fts.py`, `hermes_state_search.py` | FTS5 index DDL (including the CJK index) and `search_messages` |
| `hermes_state_wal.py` | Journal-mode and PRAGMA policy (WAL with fallback to DELETE, `database:` config pragmas) |
| `hermes_state_errors.py`, `hermes_state_repair.py` | Lock/corruption error classification, repair paths |
| `hermes_state_sessions.py`, `hermes_state_messages.py`, `hermes_state_titles.py` | Session lifecycle, message append/read, titles |
| `hermes_state_portability.py`, `hermes_state_maintenance.py` | Export/import, prune and maintenance |

This is not the full list; run `fd 'hermes_state' <checkout>` to enumerate the siblings (gateway, lock owners, usage, rewind, compression and many more).

## Tables

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py) · [hermes_state_fts.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_fts.py)

The developer guide's architecture diagram lists the tables below. `SCHEMA_SQL` in `hermes_state_common.py` additionally declares `system_prompts`, `gateway_hygiene_state`, `conversation_generations`, `gateway_heartbeats` and `session_turn_leases`, which I verified in source.

```
state.db (SQLite; WAL by default)
├── sessions               session metadata, token counts, billing, gateway routing, workspace
├── messages               full message history per session
├── session_model_usage    per model/provider/task usage attribution rows
├── system_prompts         prompt text deduplicated by hash (sessions.system_prompt_hash)
├── messages_fts           FTS5 index (content, tool_name, tool_calls)
├── messages_fts_trigram   FTS5 trigram index (substring / CJK)
├── messages_fts_cjk       FTS5 index using the cjk_unicode61 tokenizer
├── state_meta             key/value markers (FTS stale/rebuild markers, backfill cursors, ...)
├── gateway_routing        session-key -> active session routing index
├── gateway_hygiene_state, conversation_generations, gateway_heartbeats
├── compression_locks, session_turn_leases      cross-process leases
├── async_delegations      async delegation bookkeeping
├── delivery_obligations   gateway outbox (owed replies), created lazily by gateway/delivery_ledger.py
├── runtime_epoch, session_admissions, worker_executions, worker_receipts
│                          session-authority ledger (owner epoch, durable input FIFO, managed workers)
└── schema_version         single integer
```

### sessions

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py)

One row per session. Verified columns in `SCHEMA_SQL` include identity and lineage (`id`, `source`, `user_id`, `parent_session_id`), gateway routing metadata (`session_key`, `chat_id`, `chat_type`, `thread_id`, `display_name`, `origin_json`, `expiry_finalized`), model and prompt (`model`, `model_config`, `system_prompt`, `system_prompt_hash`), timing (`started_at`, `ended_at`, `end_reason`, `last_activity_at`), counters (`message_count`, `tool_call_count`, `api_call_count`), token and billing columns (`input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`, `billing_provider`, `billing_base_url`, `billing_mode`, `estimated_cost_usd`, `actual_cost_usd`, `cost_status`, `cost_source`, `pricing_version`), workspace fields (`cwd`, `git_branch`, `git_repo_root`), `title` and `title_source`, handoff and compression-failure bookkeeping, and the flags `profile_name`, `transport_profile`, `rewind_count`, `archived`, `auto_archived`, `pinned`, `hidden`. Timestamps are Unix-epoch REAL values.

Constraints stated in the docs: `parent_session_id` self-references `sessions(id)`; `title` has a unique index that allows many NULLs (only non-NULL titles must be unique). Indexes on `source`, `parent_session_id` and `started_at DESC` are documented.

`user_id` semantics per the docs: messaging adapters store the platform sender id; Desktop/dashboard sessions opened through an authenticated `hermes serve` store the login as `<provider>:<user id>`; sessions with nobody behind them (anonymous loopback, `subagent`, `cron`, `kanban`) keep it empty. It is set at row creation and never inferred later.

### messages

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py)

One row per message: `id` (autoincrement), `session_id`, `role`, `content`, `tool_call_id`, `tool_calls` (JSON string), `tool_name`, `timestamp`, `token_count`, `finish_reason`, plus reasoning carriers (`reasoning`, `reasoning_content`, `reasoning_details`, `codex_reasoning_items`, `codex_message_items`). Newer columns named by the docs: `effect_disposition`, `platform_message_id`, `observed`, `active`, `compacted`, `api_content`, `display_kind`, `display_metadata`, `message_uid`, `absorbed_message_uids`, `tool_call_uids`, `tool_call_uid`.

Behaviors the docs call out:

- `active` / `compacted`: in-place compaction archives old rows with `active=0` and inserts the retained context as `active=1` rows. A protected message can legitimately exist in both generations with identical content; do not delete archive rows as duplicates, and use `active=1` when diagnosing duplicate live writes.
- `api_content` is a byte-fidelity sidecar holding the exact string sent to the API when it differs from `content` (ephemeral injections, persist overrides); NULL means `content` was sent verbatim.
- `message_uid` is a durable 32-hex per-message id minted at first insert and copied by every clone (compaction generations, rotation children, export/import); context engines key per-message state on it. `absorbed_message_uids` is the merge witness (uids folded into a surviving row). `tool_call_uids` / `tool_call_uid` give each tool call a per-occurrence id because provider tool-call ids can repeat.
- Reasoning is always kept in history; on the chat-completions wire it is replayed on the carriers the route reads, and a route that rejects a field by name gets it stripped and retried once for that session without touching history.
- `display_metadata` is a sidecar that never reaches provider messages; the gateway uses it for input-ownership markers on its exception path (arbitration of failure writers, explicitly not universal exactly-once delivery).

### session_model_usage

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py)

Per-model usage attribution. Verified in source: primary key `(session_id, model, billing_provider, billing_base_url, billing_mode, task)`, with api-call, token, estimated/actual cost and first/last-seen columns; rows cascade on session delete. Added by migrations v20 (seeded from historical aggregates) and v22 (`task` joins the primary key).

### Session-authority and delivery ledger

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Session Storage Recovery](https://hermes-agent.nousresearch.com/docs/user-guide/session-storage-recovery) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py) · [gateway/delivery_ledger.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/delivery_ledger.py)

`runtime_epoch` (one row: owner epoch plus instance id), `session_admissions` (durable per-session input FIFO: queued, started, terminal, or `unknown` on owner loss), `worker_executions` and `worker_receipts` (managed worker, child, cron and kanban executions bound to a session and epoch, with ordered digest-checked results). The docs state these reference `sessions` with `ON DELETE RESTRICT`: deleting or pruning a session retires its terminal ledger rows in the same transaction and refuses while work is live or `unknown`. An input that was `started` when its owner died becomes `unknown` at the next owner start and is never replayed; later inputs wait until it is resolved (`prompt.resolve_unknown`). The user guide documents `hermes sessions` handling for a turn lost in a gateway crash (see the Sessions page, "Discard a Turn Lost in a Gateway Crash"). `delivery_obligations` (owed replies) is created lazily by `gateway/delivery_ledger.py`.

## Full-text search indexes

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [State DB Recovery](https://hermes-agent.nousresearch.com/docs/developer-guide/state-db-recovery) · [hermes_state_fts.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_fts.py) · [hermes_state_search.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_search.py) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py)

- `messages_fts` is an FTS5 table over `content`, `tool_name`, `tool_calls`, kept in sync by INSERT/UPDATE/DELETE triggers. The triggers are gated on `fts_rebuild_high_water` / `fts_rebuild_progress` markers in `state_meta` so a background rebuild does not double-index.
- `messages_fts_trigram` (trigram tokenizer, substring and CJK) and `messages_fts_cjk` (`cjk_unicode61` tokenizer) are additional indexes. Per the migration table, cron sessions (v29) and delegate-child/subagent sessions (v30) are excluded from the trigram index only; their rows remain in `messages` and in the standard word index, so `session_search` still finds them.
- FTS storage layout is tracked independently of `SCHEMA_VERSION` in the `state_meta` key `fts_storage_version`; v23 is the "external-content" redesign, and `hermes sessions optimize-storage` migrates an existing database to that compact layout.
- Query syntax supported by the search tool (the user guide): implicit-AND keywords, `"exact phrase"`, `OR`, `NOT`, and prefix `deploy*`. `_sanitize_fts5_query()` strips unmatched quotes, wraps hyphenated terms in quotes and drops dangling boolean operators.
- The agent-facing `session_search` tool uses FTS5 with no LLM call. The docs describe four shapes inferred from arguments (no `mode` parameter): discovery (`query`), scroll (`session_id` + `around_message_id`, with `window`), read (`session_id` alone) and browse (no arguments); optional `sort`, `detail` and `role_filter`.
- Derived-index failure policy: `sessions` and `messages` are canonical and the FTS tables are derived. On an FTS corruption error `SessionDB` records a durable `fts_stale` marker, removes the FTS triggers in the same transaction, retries canonical writes without them and serves searches from a `LIKE` fallback; the failing live operation never runs `FTS5('rebuild')`.

## Schema version and migrations

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_common.py) · [hermes_state_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_schema.py)

- `schema_version` holds one integer; source value at this commit is 32.
- Policy (docs and the source comment): every writable open runs `executescript(SCHEMA_SQL)` plus `_reconcile_columns()`, which diffs live columns against the declared schema and adds missing ones (idempotent, additive). The version-gated chain is reserved for data migrations and index/FTS changes that cannot be declared. Schema drift is detected by a read probe, not by the number; a new table or column needs no bump.
- Documented data-migration milestones (versions not listed were declarative column additions):

| Version | Change (per docs) |
|---|---|
| 1-9 | Initial sessions/messages/FTS5, then `finish_reason`, `title` (+ unique index), billing columns, reasoning columns, `api_call_count`, `codex_message_items` |
| 10-11 | Trigram index; FTS widened to `tool_name` + `tool_calls` |
| 16 | Tag delegate subagent rows in `model_config` (`$._delegate_from`) |
| 18 | Gateway metadata consolidation (`display_name`, `origin_json`, `expiry_finalized` backfilled from `sessions.json`) |
| 20, 22 | `session_model_usage` seeded, then `task` added to its primary key |
| 23 | External-content FTS redesign (opt-in transition for existing databases) |
| 29, 30 | Cron, then subagent sessions leave the trigram index |
| 31 | `message_uid` family of columns, chunked resumable backfill, insert trigger minting uids |
| 32 | Version bump only (source comment); no data migration |

## Connection, journal mode and PRAGMAs

> Sources: [Hermes Agent Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) · [hermes_state_wal.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_wal.py) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

- WAL is the default so that many readers and one writer can coexist. The `database:` section of `config.yaml` selects `journal_mode` (`wal` default, or `delete` for filesystems where WAL is unsafe) and optional PRAGMAs. The config template shows:

```yaml
database:
  journal_mode: "wal"          # "wal" or "delete"
  # wal_autocheckpoint: 1000     # pages between automatic checkpoints
  # journal_size_limit: 67108864 # cap on WAL/journal size in bytes
  # synchronous: FULL            # OFF, NORMAL, FULL, EXTRA (or 0-3); unset = SQLite's compile-time default
```

- `hermes_state_wal.apply_database_pragmas` also reads `cache_size`, `mmap_size` and `temp_store` from this section (source docstring); non-integer values are ignored with a warning. On macOS the docs say `synchronous` is a floor, not a pin: values below FULL are refused (EXTRA honored).
- Journal mode is applied by `apply_wal_with_fallback`: if WAL cannot be enabled the connection falls back to DELETE. The docs state fresh databases on virtiofs/9p mounts are created in delete mode automatically, and that an existing WAL database is never live-downgraded when you set `journal_mode: delete` (Hermes logs that the setting did not apply; `hermes doctor` warns). Conversion is explicit: stop every process, then `hermes sessions set-journal-mode delete` (or `wal`); it refuses while any process holds the file and verifies the header.
- Foreign keys are enabled per connection (`PRAGMA foreign_keys=ON` in `hermes_state.py`).

## Write contention handling

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py) · [hermes_state_errors.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_errors.py)

Several processes open one `state.db` (the gateway as canonical writer, read-only browsers, maintenance commands, writers of non-session tables). `SessionDB` therefore uses:

- a short SQLite busy timeout (1 s) instead of the default 30 s, with time-budgeted application-level retry and random jitter;
- `BEGIN IMMEDIATE` transactions so contention surfaces at transaction start;
- a passive WAL checkpoint every 50 successful writes.

Constants verified in `hermes_state.py` (they replace the older attempt-counted constants such as `_WRITE_MAX_RETRIES`, which are no longer the model):

| Constant | Value | Meaning |
|---|---|---|
| `_WRITE_PATIENCE_S` | 20.0 | routine write budget (seconds) |
| `_TRANSCRIPT_WRITE_PATIENCE_S` | 60.0 | transcript writes (failure aborts the turn) |
| `_ACTIVITY_WRITE_PATIENCE_S` | 0.5 | observation-only activity writes |
| `_WRITE_RETRY_MIN_S` / `_MAX_S` | 0.020 / 0.150 | fast jitter for the first 2 s |
| `_WRITE_RETRY_SLOW_MIN_S` / `_MAX_S` | 0.250 / 1.000 | slow jitter afterwards |
| `_CHECKPOINT_EVERY_N_WRITES` | 50 | passive checkpoint cadence |
| `_READ_BUSY_TIMEOUT_S` | 5.0 | read-only open busy budget |

When a writer exhausts its budget the turn ends with `session_persistence_failed:locked`; the docs say that on Linux `hermes_state_lockowners` logs a warning naming the process that held the lock (read from `/proc/locks`). Lock contention is recognised by SQLite result code (`SQLITE_BUSY`/`SQLITE_LOCKED`), not message text; the dashboard answers 503 rather than 500 when the lock outlasts its read budget.

## Python API (SessionDB)

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py) · [hermes_state_sessions.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_sessions.py) · [hermes_state_messages.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_messages.py) · [hermes_state_search.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_search.py) · [hermes_state_titles.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_titles.py) · [hermes_state_portability.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_portability.py) · [hermes_state_maintenance.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_maintenance.py)

The documented public surface (names confirmed in the split modules; signatures as the docs show):

```python
from hermes_state import SessionDB
db = SessionDB()                                   # default: get_hermes_home()/state.db
db.create_session(session_id=..., source="cli", model=..., user_id=..., parent_session_id=None)
db.append_message(session_id=..., role="assistant", content=..., tool_calls=[...], token_count=..., finish_reason="stop", reasoning=...)
db.get_messages(session_id)                        # raw rows
db.get_messages_as_conversation(session_id)        # OpenAI-format replay
db.search_messages("docker deployment", source_filter=["cli"], exclude_sources=[...], role_filter=["user"])
db.set_session_title(...); db.resolve_session_by_title(...); db.get_next_title_in_lineage("Fix Docker Build")  # "... #2"
db.end_session(session_id, end_reason="user_exit"); db.reopen_session(session_id)
db.export_session(session_id); db.export_all(source="cli")
db.prune_sessions(older_than_days=90); db.clear_messages(session_id); db.delete_session(session_id)
```

Because the gateway is the canonical writer, application code outside the gateway generally should not call the write methods directly against a live profile; see "Where the database lives and who writes it". Search results carry `snippet` with `>>>match<<<` markers, 1-message context, and parent-session `source`, `model`, `session_started`.

## Session lineage

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions)

Compression-triggered splits create chains through `parent_session_id`; titles auto-number in a lineage (`my project`, `my project #2`). The docs give recursive-CTE queries for ancestors and descendants:

```sql
WITH RECURSIVE lineage AS (
    SELECT * FROM sessions WHERE id = ?
    UNION ALL
    SELECT s.* FROM sessions s JOIN lineage l ON s.id = l.parent_session_id
)
SELECT id, title, started_at, parent_session_id FROM lineage;
```

Pruning and archiving treat a compressed conversation as a unit through its live tip; older segments stay while a later segment is kept.

## Retention, pruning and maintenance

> Sources: [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Session Storage Recovery](https://hermes-agent.nousresearch.com/docs/user-guide/session-storage-recovery) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

Config (`sessions:` block, shown commented in the template):

```yaml
sessions:
  auto_prune: false            # opt-in; true deletes ended sessions past retention_days
  retention_days: 90
  vacuum_after_prune: true
  min_vacuum_interval_days: 30
  min_interval_hours: 24
  orphan_end_grace_s: 300
  # max_resume_messages: 20000, max_export_messages: 20000 (documented on the Sessions page; 0 disables)
```

Per the Sessions page: auto-prune is off by default; only ended sessions are pruned (aged from last activity); after a prune that removed rows, `VACUUM` runs only if `min_vacuum_interval_days` has elapsed and more than 25% of pages are reclaimable (source constant `AUTO_VACUUM_MIN_FREELIST_RATIO = 0.25`); a low-disk prompt (`/autoprune on|off|later`) can ask once when the config has no `auto_prune` key; open sessions from state-owned automation sources are closed (not deleted) after the retention window; `sessions.orphan_end_grace_s` ends local sessions whose last viewer vanished.

`hermes sessions` subcommands listed in the CLI reference: `list`, `browse`, `export`, `delete`, `prune` (many filters, `--dry-run`, `--yes`), `archive`, `stats`, `rename`, `optimize` (FTS merge + VACUUM), `optimize-storage` (FTS v23 layout), `repair`, `repair-routing`, `repair-prompts`, `refresh-prompts`, `repair-profiles`, `recover`, `retitle-skills`. The Sessions page additionally documents `pin`, `set-journal-mode` and the crashed-turn discard command. Rewriting maintenance commands (`optimize`, `optimize-storage`, `prune`) refuse while another process holds the database unless `--force` is passed; `--dry-run` previews are never blocked. Under a running gateway, `prune` shows its preview from a read-only view and the gateway performs the deletion.

## Recovery and corruption handling

> Sources: [State DB Recovery](https://hermes-agent.nousresearch.com/docs/developer-guide/state-db-recovery) · [Session Storage Recovery](https://hermes-agent.nousresearch.com/docs/user-guide/session-storage-recovery) · [hermes_state_repair.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_repair.py) · [hermes_state_errors.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_errors.py)

- Retired-WAL refusal: if one process rewrites the store while another still holds an old copy of the write-ahead log, the stale holders stop writing on purpose. The documented fix: stop every Hermes process on the profile (`hermes gateway stop`, quit Desktop, stop dashboard), run `hermes doctor` to list holders as `PID N (command)`, then start one process. Do not run `hermes doctor --fix` while processes run, do not delete `-wal`/`-shm`, do not copy `state.db` alone.
- Bare `SQLITE_CORRUPT`/`SQLITE_NOTADB` on a live write quarantines the handle (`StateDbCorruptError`): the write propagates the typed error, later writes fail without touching the file, the handle never reopens, and `close()` skips its explicit checkpoint. Pending transcripts spool to `sessions/<id>.jsonl` and `pending_messages/`. Next steps in the docs: `hermes gateway stop`, then `hermes sessions recover --source <state.db> --inspect-only`, then `--output <new file>`, or restore the newest snapshot from `state-snapshots/`.
- `hermes sessions repair --check-only` / `repair` for schema/FTS damage (makes a backup by default; stop all processes first).
- `hermes sessions recover` copies row-bearing tables into a recovered database (FTS indexes and `schema_version` regenerated; `delivery_obligations` included when present). The four runtime-ledger tables are inventoried and reported as excluded, not copied; a full `hermes backup` restore keeps the complete ledger.
- Side files you may see beside `state.db`: `state.db.retired-wal-*`, `state.db.pre-update-emergency-*.bak`, `state.db.corrupt.*.bak`, `state-snapshots/` (the Session Storage Recovery page tabulates what to do with each).

## Browsing and test isolation

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state_guard.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state_guard.py)

- Dashboard session browsing opens `state.db` read-only; GET listing, search, details and export never create an absent database, quarantine a zero-byte file, reconcile a stale schema or trigger auto-archive. Uninitialized or incompatible stores return HTTP 503. Read-only mode may still create empty WAL/SHM sidecars (SQLite coordination files).
- Tests must use a temporary `HERMES_HOME` or explicit temp database. A live-system guard raises before a test-context process opens a production `state.db`; `HERMES_STATE_DB_GUARD_BYPASS=1` and `@pytest.mark.live_system_guard_bypass` are test-only escapes that should never be set in normal shells.

Not covered here: individual per-module internals of the `hermes_state_*` family, the gateway's input-ownership arbitration details beyond the summary above, and Codex app-server input ownership (see the Session Storage page).

---

# 16. MCP (Model Context Protocol) Integration

> Sources: [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [Use MCP with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [tools/mcp_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool.py) · [tools/mcp_tool_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_schema.py) · [tools/mcp_tool_registration.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_registration.py)

MCP lets Hermes use tools hosted outside the agent: local stdio subprocesses and remote HTTP (Streamable HTTP or SSE) servers, configured under `mcp_servers` in `config.yaml`. Hermes discovers tools at startup and registers them in the normal tool registry. This section is a reference for configuration and behavior as the docs and source describe it at the pinned commit; none of it was executed. For a task-oriented walkthrough see the Use MCP with Hermes guide.

## Implementation layout

> Sources: [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [tools/mcp_tool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool.py) · [tools/mcp_tool_config.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_config.py) · [tools/mcp_tool_discovery.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_discovery.py) · [tools/mcp_tool_registration.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_registration.py) · [tools/mcp_tool_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_schema.py) · [tools/mcp_tool_transport.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_transport.py) · [tools/mcp_oauth.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_oauth.py)

The MCP client is split across `tools/mcp_tool*.py` (config, discovery, registration, schema conversion, transport, handlers, sampling, costs, events, health, lifecycle) and `tools/mcp_oauth*.py` / `tools/mcp_schema_cache.py` (OAuth manager and provider, device flow, schema cache). Enumerate with `fd 'mcp' tools/`. The older single-file `tools/mcp_tool.py` is the facade.

## Configuration

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [tools/mcp_tool_config.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_config.py) · [tools/mcp_tool_common.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_common.py)

Root shape (from the config reference):

```yaml
mcp_servers:
  <server_name>:
    command: "..."          # stdio
    args: []
    env: {}
    # OR
    url: "..."              # HTTP
    headers: {}
    ssl_verify: true        # bool or CA bundle path
    client_cert: "/path/to/cert.pem"   # mTLS; string, [cert, key] or [cert, key, password]
    # client_key: "/path/to/key.pem"
    enabled: true
    timeout: 120            # tool-call timeout (reference default: 300)
    connect_timeout: 60
    supports_parallel_tool_calls: false
    tools:
      include: []
      exclude: []
      resources: true
      prompts: true
```

Server keys documented in the reference (see the page for full semantics):

| Key | Applies to | Notes from the docs |
|---|---|---|
| `command`, `args`, `env`, `cwd` | stdio | `cwd` defaults to the session working directory when pinned |
| `url`, `headers` | HTTP | `transport: sse` selects SSE instead of Streamable HTTP; `skip_preflight` bypasses the content-type probe |
| `ssl_verify`, `client_cert`, `client_key` | HTTP/SSE | mTLS and custom CA; `ssl_verify: false` disables verification (insecure) |
| `timeout` | both | tool-call timeout seconds; docs default `300` (source `_DEFAULT_TOOL_TIMEOUT = 300`) |
| `connect_timeout` | both | default `60` (source `_DEFAULT_CONNECT_TIMEOUT = 60`) |
| `enabled` | both | `false` = no connection, discovery or registration; config is kept |
| `protocol` | both | `auto` (default), `stateless`, `legacy` negotiation |
| `keepalive_interval` | both | liveness ping cadence (floor 5 s); HTTP defaults to 180; stdio off unless set |
| `lazy` | both | register tools from the on-disk schema cache and connect on first call (default false; needs one prior live connect) |
| `idle_timeout_seconds`, `max_lifetime_seconds` | stdio | recycle memory-heavy servers; `0` disables |
| `supports_parallel_tool_calls` | both | opt-in concurrent execution (default sequential) |
| `auth` | HTTP | `oauth` enables OAuth 2.1 with PKCE |
| `sampling`, `elicitation`, `events` | both | server-initiated requests (see below); `events` is experimental, default false |
| `trust`, `tool_costs`, `spend_limit`, `cost_unit` | both | trust tier and paid-tool budgets (see below) |
| `identity_header` | HTTP/SSE | per-user header `{name, value_from: static or profile, value}`; stdio ignores it |

Related config outside the per-server block: `mcp.discovery_concurrency` (default 4 simultaneous server connects, `0` = unlimited, per the MCP page and source) and `mcp_discovery_timeout` (default `1.5` in `hermes_cli/config_defaults.py`) which bounds the startup wait; late servers are picked up on a later refresh, so correctness does not depend on the timer. Each wave of connects keeps a 120 s budget with a 300 s cap for the whole pass, per the docs.

Interpolation: any string value may use `${VAR}` or `${env:VAR}`, resolved from the active profile's secret scope then the process environment; context variables `${userHome}`, `${workspaceFolder}`, `${workspaceFolderBasename}`, `${pathSeparator}` / `${/}` are also supported. An unset variable in a remote `url`/`headers` fails the connect closed rather than sending a literal placeholder.

## Transports and server kinds

> Sources: [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [tools/mcp_tool_transport.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_transport.py)

- stdio: Hermes spawns the process. The environment is not the full shell environment: only configured `env` plus a safe baseline (and, per source, `XDG_*` and variables injected by an external secret source) are passed (`_build_safe_env`).
- HTTP: Streamable HTTP by default; `transport: sse` for SSE. Standard proxy variables apply (`HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, `NO_PROXY` including CIDR and wildcard patterns).
- mTLS: set `client_cert` (and `client_key`); `~` paths expand and missing files fail fast with a server-scoped error.
- Presets: `hermes mcp add <name> --preset codex` fills in command/args for the Codex CLI's MCP server.

## Tool naming and registration

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [tools/mcp_tool_schema.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_schema.py) · [tools/mcp_tool_registration.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_registration.py)

Native MCP tools are registered as `mcp__<server>__<tool>` (double underscores), matching the convention shared with other coding agents. Utility tools use the same prefix (`mcp__<server>__list_resources`, `read_resource`, `list_prompts`, `get_prompt`).

```
server "my-api", tool "list-items.v2"  ->  mcp__my_api__list_items_v2
```

- Sanitization: every character outside `[A-Za-z0-9_]` in server and tool names becomes `_` (`sanitize_mcp_name_component`).
- Length clamp: names over 64 characters are shortened with a deterministic 8-character hash suffix and a one-time warning (`_MCP_TOOL_NAME_MAX_LENGTH = 64`); dispatch is unaffected because handlers keep the original tool name.
- Filters (`include`/`exclude`) use the original MCP tool name, not the sanitized one.
- Each server with at least one registered tool also forms a runtime toolset named `mcp-<server>`.
- Docs inconsistency: the prose on the MCP feature page still shows the older single-underscore form (`mcp_<server>_<tool>`), while the config reference and `tools/mcp_tool_schema.py` (`MCP_TOOL_NAME_PREFIX = "mcp__"`) use the double-underscore form. This section follows the source. The older architecture notes that said `mcp_github_create_issue` are obsolete.
- Result handling: invisible Unicode TAG characters are stripped from tool results, resource content and tool descriptions; vendor `_meta` is passed through while protocol-reserved key prefixes are dropped; `ImageContent` results are cached to disk and reported as `MEDIA:` paths (and attached for vision-capable models).

## Filtering and utility tools

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [Use MCP with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes) · [tools/mcp_tool_registration.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_registration.py)

- `tools.include` (string or list): whitelist of server-native tools. `tools.exclude`: blacklist. Entries may be exact names or fnmatch-style globs (`*_radar_*`). If both are set, `include` wins and `exclude` is ignored.
- `resources: false` / `prompts: false` disable the corresponding utility wrappers. An empty `include: []` with `resources: true` yields a resource-only server.
- Capability-aware: resource/prompt utilities register only if the MCP session actually advertises that capability. Event utilities (`list_event_types`, `subscribe_event`, `unsubscribe_event`) register whenever `events: true`, because the SDK drops the events capability while parsing.
- If filtering removes every native tool and no utilities register, no empty runtime toolset is created.
- `hermes mcp configure <name>` toggles tool selection interactively; `hermes mcp test <name>` connects and lists tools (the guide documents exit codes 0/1/3).

## Authentication: OAuth 2.1 and headers

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [OAuth over SSH / Remote Hosts](https://hermes-agent.nousresearch.com/docs/guides/oauth-over-ssh) · [tools/mcp_oauth.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_oauth.py) · [tools/mcp_oauth_device.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_oauth_device.py)

```yaml
mcp_servers:
  protected_api:
    url: "https://mcp.example.com/mcp"
    auth: oauth
```

Documented behavior:

- Uses the MCP SDK's OAuth 2.1 PKCE flow (metadata discovery, client identification, token exchange, refresh). Tokens persist under `~/.hermes/mcp-tokens/<server>.json` (named profiles use their own `mcp-tokens/` under the profile home) with owner-only permissions, and refresh automatically; re-authorization happens only when refresh fails. Applies to URL-based servers only.
- Refresh tokens are bound to the authorization server that issued them; a changed issuer drops the stored refresh token. RFC 9207 `iss` checking and RFC 8414 issuer matching are enforced, with documented narrow exceptions.
- Client identification: a Client ID Metadata Document (CIMD) when the server advertises support, otherwise Dynamic Client Registration (DCR). Pre-registered `oauth.client_id` / `client_secret` skip registration; `oauth.cimd: false` forces DCR; callback ports for CIMD flows are pinned to the 27890-27894 range.
- Headless and remote hosts: Desktop-hosted callback relay, paste-back of the redirect URL, `hermes mcp login <server> --flow device` (RFC 8628 device authorization; `oauth.flow: device` makes it the default for `login` and `reauth`), SSH port forward, or a proxied `oauth.redirect_uri`. `oauth.timeout` extends the approval wait (default 5 minutes).
- Background reconnects (gateway, `/reload-mcp`, self-probe) never open a browser; an expired grant parks the server until `hermes mcp login <server>` is run.
- Static headers (`headers:` with `${VAR}`) remain available for bearer-token servers. `hermes mcp add --auth oauth|header` sets the mode at add time.
- Under a multiplexed gateway an OAuth connection is never shared across profiles.

## Server-initiated requests: sampling, elicitation, events

> Sources: [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [tools/mcp_tool_sampling.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_sampling.py) · [tools/mcp_tool_events.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_events.py)

- Sampling (`sampling/createMessage`): the docs state it is enabled by default per server when the SDK supports it, with a per-server `sampling:` block: `enabled`, `model`, `max_tokens_cap` (4096), `timeout` (30), `max_rpm` (10), `max_tool_rounds` (5), `allowed_models`, `log_level`. A sliding-window rate limiter, per-request timeout and tool-loop depth cap are described. Set `sampling.enabled: false` for servers you do not trust.
- Elicitation (`elicitation/create`): enabled by default; form-mode requests route through the normal approval surface (Allow Once / Deny; YOLO and `approvals.mode: off` do not auto-answer them), URL-mode requests are declined; `elicitation.timeout` defaults to 300 s.
- Events (experimental, off by default): `events: true` plus `platforms.webhook.extra.public_url` (HTTPS) and a running gateway; events arrive as webhook deliveries that start agent runs. Poll and push delivery modes and secondary-profile subscriptions under multiplexing are listed as not supported.
- Dynamic discovery: `notifications/tools/list_changed` triggers a lock-protected refresh of that server's tools without `/reload-mcp`; prompt/resource list-changed notifications are received but not acted on.

## Trust tiers, tool costs and spend limits

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [tools/mcp_tool_costs.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_costs.py)

The v0.23.0 release notes advertise "price tags and spend limits for paid tools"; the docs describe the mechanism as follows.

- `trust: full` (default) or `untrusted`. On an `untrusted` server every write-capable call (any tool lacking `readOnlyHint: true`) needs approval through the standard surface (Allow Once, Session, or Always, which is written to `command_allowlist` as `mcp:<server>/<tool>`). YOLO and `approvals.mode: off` skip this prompt but not a declared `spend_limit`. The docs stress that `readOnlyHint` is a server-supplied hint, so mark servers you do not control as `untrusted`; unrecognized values are treated as `untrusted`.
- `tool_costs` maps MCP tool names to a non-negative number, a `[min, max]` range, or `{credits: ..., note: ...}`. `spend_limit` is a per-chat budget for the server's costed tools (`0`/absent = no limit); `cost_unit` is the label (default `credits`).

```yaml
mcp_servers:
  renders:
    command: "render-mcp"
    trust: untrusted
    tool_costs:
      create_image: 20
      animate: {credits: [5, 20], note: "per animation"}
    spend_limit: 100
    cost_unit: credits
```

- The approval prompt names the cost and the chat's running total. A call that would cross the limit asks first even on a `trust: full` server; where nobody can answer (cron, `hermes chat -q`) it is refused. Totals are in memory, per chat and per server; they follow compression/`/branch` continuations, reset on restart, and are a budget for one conversation, not the vendor's bill.
- A malformed cost/limit key skips the server with an error naming the key. A portable plugin package can declare the same table; a same-name `config.yaml` entry overrides only the keys it sets.

## Runtime behavior and operator commands

> Sources: [MCP (Model Context Protocol)](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [tools/mcp_tool_discovery.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_discovery.py) · [tools/mcp_tool_lifecycle.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/mcp_tool_lifecycle.py)

- Discovery: servers connect at most 4 at a time per pass (`mcp.discovery_concurrency`). Lazy servers show as `lazy` with a cached tool count until first use.
- Reload: `/reload-mcp` re-reads config and refreshes tools (it is also the explicit way to re-probe availability-gated tools). A running gateway also watches `config.yaml`: a removed or disabled entry is torn down and a new one connected within about a minute, and a server whose first connect failed is retried on a cooldown schedule (30 s doubling to 10 min).
- CLI (`hermes mcp ...`): picker and `catalog` for the Nous-approved catalog, `install <name>`, `add <name> [--url|--command|--auth|--args|--preset]`, `remove`, `list`, `test`, `configure`, `login`. The docs say catalog installs run the manifest's bootstrap commands and the server's own code, so read the manifest first.
- Running Hermes itself as an MCP server: this moved out of core. The CLI reference states it is now the official `mcp-serve` plugin: `hermes plugins install mcp-serve`, then `hermes mcp-serve`; `hermes mcp serve` only prints the new spelling. MCP clients configured with `"args": ["mcp", "serve"]` must change to `["mcp-serve"]`; tool names and parameters are unchanged. Tool surface and the `--tools` mode are documented in that plugin's own README (not in this repository), so they are not described here.

## Example configurations

> Sources: [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [Use MCP with Hermes](https://hermes-agent.nousresearch.com/docs/guides/use-mcp-with-hermes)

```yaml
mcp_servers:
  github:                       # allowlist + no utility wrappers
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "${GITHUB_TOKEN}"
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      resources: false
      prompts: false

  payments:                     # blacklist
    url: "https://mcp.example.com"
    headers:
      Authorization: "Bearer ${PAYMENTS_MCP_TOKEN}"
    tools:
      exclude: [delete_customer, refund_payment]

  partner_api:                  # separate client cert and key
    url: "https://mcp.partner.example.com/mcp"
    client_cert: "~/secrets/client.crt"
    client_key: "~/secrets/client.key"
```

Not covered here: the catalog manifest format (`optional-mcps/`), the Windows/WSL2 browser-bridge recipes in the guide, and the full OAuth CIMD edge cases; see the MCP Config Reference.

---

# 17. Provider Runtime Resolution

> Sources: [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [Model Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin) · [Adding Providers](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-providers) · [LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) · [hermes_cli/runtime_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider.py) · [hermes_cli/auth.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/auth.py) · [providers/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/__init__.py) · [providers/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/base.py)

One shared resolver turns "which provider/model did the user ask for" into a concrete runtime (provider, `api_mode`, `base_url`, credential, source). The docs state it is used by the CLI, gateway, cron jobs, ACP and auxiliary model calls. This section covers the resolver, the provider plugin registry, credential pools, fallback chains, auxiliary routing and the model catalog, as documented at the pinned commit.

## Components

> Sources: [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [hermes_cli/runtime_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider.py) · [hermes_cli/runtime_provider_custom.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider_custom.py) · [hermes_cli/runtime_provider_backends.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider_backends.py) · [hermes_cli/auth.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/auth.py) · [hermes_cli/providers.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/providers.py) · [agent/auxiliary_client.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/auxiliary_client.py) · [providers/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/__init__.py)

| Piece | Role |
|---|---|
| `hermes_cli/runtime_provider.py` | `resolve_requested_provider()`, `resolve_runtime_provider()`; extracted collaborators `runtime_provider_custom.py` (named custom providers), `runtime_provider_backends.py` (OpenRouter, Bedrock, Azure Foundry, external-process), `runtime_provider_missing_key.py` |
| `hermes_cli/auth.py` | `PROVIDER_REGISTRY`, `resolve_provider()`, auth stores |
| `hermes_cli/providers.py` | `determine_api_mode()` |
| `hermes_cli/model_switch.py` | shared `/model` switch pipeline (CLI and gateway), per the docs |
| `agent/auxiliary_client.py` | auxiliary model routing |
| `providers/` | `ProviderProfile` ABC and registry (`register_provider`, `get_provider_profile`, `list_providers`) |
| `plugins/model-providers/<name>/` | bundled provider plugins declaring `api_mode`, `base_url`, `env_vars`, `fallback_models` |

`runtime_provider.py` asks the profile registry for the canonical `base_url`, `env_vars` priority list, `api_mode` and `fallback_models`, so a new plugin that calls `register_provider()` needs no branch in the resolver.

## Resolution precedence

> Sources: [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [hermes_cli/runtime_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider.py)

High-level order per the docs: (1) explicit CLI/runtime request, (2) `config.yaml` model/provider, (3) environment variables, (4) provider-specific defaults or auto resolution. The saved model/provider is the source of truth so a stale shell export cannot silently override the endpoint last selected in `hermes model`. In source, `resolve_requested_provider` returns the explicit argument, then `model.provider` from config, then `HERMES_INFERENCE_PROVIDER`, else `"auto"`.

The `resolve_runtime_provider` docstring describes an ordered ladder where each rung returns or falls through:

```
1. disabled-provider guard (providers.<name>.enabled: false)
2. requested-name shortcuts (e.g. moa, azure-foundry, vertex)
3. named custom provider / llamacpp alias / bare-custom direct alias
4. local-endpoint bypass (config base_url at a non-cloud host, no explicit creds)
5. auth.resolve_provider -> explicit --api-key / --base-url path
6. credential pool
7. OAuth specs (nous, codex, xai, qwen) -> minimax-oauth -> external-process
   -> anthropic env -> bedrock -> registry API-key providers
8. OpenRouter / bare-custom fallback
9. model.openai_runtime overlay (may rewrite api_mode to codex_app_server)
```

Output (as the docs list it): `provider`, `api_mode`, `base_url`, `api_key`, `source`, plus provider metadata such as expiry/refresh info.

## api_mode

> Sources: [Model Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [hermes_cli/providers.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/providers.py) · [providers/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/base.py)

Recognized built-in modes: `chat_completions`, `codex_responses`, `anthropic_messages`, `bedrock_converse`, plus any mode a plugin registers with a transport. Selection order per the plugin guide: user `model.api_mode`; OpenCode per-model dispatch; URL auto-detection (for example an `/anthropic` suffix selects `anthropic_messages`); the profile's `api_mode` as fallback; default `chat_completions`. A profile naming an unregistered mode degrades to `chat_completions`. Examples from the runtime guide: native Anthropic uses `anthropic_messages` through `agent/anthropic_adapter.py`; OpenAI Codex uses `codex_responses`.

## Provider plugin registry

> Sources: [Model Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin) · [Adding Providers](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-providers) · [providers/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/__init__.py) · [providers/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/providers/base.py) · [plugins/model-providers/README.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/model-providers/README.md)

- Discovery is lazy (first `get_provider_profile()` / `list_providers()` call) in this order: bundled `plugins/model-providers/<name>/`; user plugins in `$HERMES_HOME/plugins/model-providers/<name>/` (picked up on the next provider lookup, no restart); installed plugins whose `plugin.yaml` declares `kind: model-provider`; legacy single files in `providers/`. The source docstring also lists pip entry-point distributions.
- User plugins override bundled ones of the same name (`register_provider()` is last-writer-wins) and are per profile home. The override also rewrites the matching `PROVIDER_REGISTRY` row's endpoint/env-var fields so runtime resolution follows it.
- Key `ProviderProfile` fields: `name`, `aliases`, `api_mode`, `env_vars`, `base_url`, `models_url`, `auth_type` (`api_key`, `oauth_device_code`, `oauth_external`, `aws_sdk`, `external_process`, or plugin-defined), `auth_handler`, `refresh_credential`, `classify_api_error`, `fallback_models`, `model_capabilities`, `default_headers`, `default_aux_model`, and others in the plugin guide.
- Inspect what loaded with `hermes doctor` (Provider Connectivity section) or `providers.list_providers()`.
- Adding a provider: simple API-key providers are a plugin directory; OAuth/native providers additionally touch `hermes_cli/auth.py`, `models.py`, `runtime_provider.py`, the CLI picker, auxiliary routing, and (for native wire formats) an adapter. Read Adding Providers for the checklist and pitfalls.

## Providers

> Sources: [LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [plugins/model-providers/README.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/model-providers/README.md)

Do not rely on a fixed count. Bundled provider plugin directories verified under `plugins/model-providers/` include (not a complete statement of everything selectable): `ai-gateway`, `alibaba`, `alibaba-coding-plan`, `anthropic`, `arcee`, `azure-foundry`, `bedrock`, `commandcode`, `custom`, `deepinfra`, `deepseek`, `fireworks`, `gemini`, `gmi`, `huggingface`, `kilocode`, `kimi-coding`, `lmstudio`, `minimax`, `nebius-token-factory`, `nous`, `novita`, `nvidia`, `ollama-cloud`, `openai-codex`, `opencode-zen`, `openrouter`, `qwen-oauth`, `router`, `stepfun`, `tencent-tokenhub`, `upstage`, `vertex`, `xai`, `xiaomi`, `zai`, among others. The LLM and Model Providers page is the maintained list of setup commands, env vars and aliases.

Corrections relative to older notes: canonical ids are the short ones used in config (`bedrock`, `nvidia`, `kilocode`, `zai`, `lmstudio`; the provider page lists aliases). Copilot (`copilot`, `copilot-acp`) is stated in the v0.23.0 release notes to have moved to the `github-copilot` labs plugin, and the provider guide has a "GitHub Copilot (labs plugin)" section; it is not in the bundled directory list above.

Custom endpoints: `provider: custom` with `model.base_url`, or named entries under the top-level `providers:` dict (the docs say the legacy `custom_providers` list is still read for back-compat and migrated).

```yaml
providers:
  local:
    api: http://localhost:8080/v1
  work:
    api: https://gpu-server.internal.example.com/v1
    key_env: CORP_API_KEY
    transport: chat_completions      # or anthropic_messages / codex_responses
  my-gateway:
    base_url: "https://gateway.internal.example.com/v1"
    api_mode: chat_completions
    key_cmd: "my-auth-cli print-token"   # command prints a short-lived token; preferred over api_key/key_env
```

The docs state `key_cmd` beats a static `api_key`/`key_env` on the same entry (an explicit `--api-key` still wins), and the minted credential is used for auxiliary tasks too.

## Credential scoping

> Sources: [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [hermes_cli/runtime_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/runtime_provider.py)

The docs state each provider's key is only sent to its own base URL: `OPENROUTER_API_KEY` to `openrouter.ai`, `AI_GATEWAY_API_KEY` to `ai-gateway.vercel.sh`, `OPENAI_API_KEY` for custom endpoints and as fallback. A real saved custom endpoint is distinguished from the OpenRouter fallback path. The release notes also state that a built-in provider's key no longer leaks onto a custom entry with the same name. A bare `custom` request that resolves with no credential raises a typed auth error rather than silently using another endpoint (`_raise_for_credentialless_bare_custom` in source).

## Native Anthropic and Codex paths

> Sources: [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) · [agent/anthropic_adapter.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/anthropic_adapter.py)

- Anthropic: `api_mode = anthropic_messages`, native Messages API via `agent/anthropic_adapter.py`. Credential resolution prefers refreshable Claude Code credentials over copied env tokens; manual `ANTHROPIC_TOKEN` / `CLAUDE_CODE_OAUTH_TOKEN` remain explicit overrides; refresh is preflighted before calls, and one retry follows a 401 after rebuilding the client.
- OpenAI Codex: `api_mode = codex_responses` with its own auth store. A resumed session whose lingering reasoning items are rejected (400 `invalid_encrypted_content` or 401 `token_expired`) strips them and replays once before any credential refresh or pool rotation.
- Reasoning shapes: OpenAI-compatible relays may return `reasoning` / `reasoning_content` as strings, text-part dicts or lists; Hermes flattens them, keeps reasoning separate from the visible answer, and appends `delta.reasoning_details` records in arrival order without rewriting them.

## Credential pools

> Sources: [Credential Pools](https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools) · [agent/credential_pool.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/credential_pool.py) · [agent/credential_pool_admin.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/credential_pool_admin.py) · [agent/credential_pool_model_cooldowns.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/credential_pool_model_cooldowns.py) · [hermes_cli/auth_commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/auth_commands.py)

Pools hold several API keys or OAuth credentials per provider and rotate within the same provider; they are tried before cross-provider fallback.

```
request -> select credential (fill_first | round_robin | least_used | random)
   429 plan/usage limit      -> rotate immediately
   429 generic               -> retry same key once, then rotate (1 h cooldown)
   402 billing               -> rotate immediately (1 h cooldown)
   401 expired OAuth         -> try refresh, rotate if it fails (5 min cooldown)
   all exhausted             -> fallback_providers (different provider)
```

- Storage: `~/.hermes/auth.json` under `credential_pool` (custom endpoints use a `custom:` prefix); strategies live in `config.yaml` under `credential_pool_strategies` (`fill_first` is the default). Borrowed secrets (env vars, secret-manager references) are reference-only at the `auth.json` boundary: only metadata and a non-reversible fingerprint persist.
- Commands: `hermes auth` (wizard), `auth list`, `auth add <provider> [--type api-key|oauth] [--priority N]`, `auth priority`, `auth remove`, `auth reset`, `auth refresh`.
- Auto-discovery seeds pools from env vars (including numbered siblings `NAME_2`, `NAME_3`, ...), OAuth stores, Claude Code credentials and custom endpoint config; auto-seeded entries are refreshed on every load and pruned when their source disappears, while manual entries are never auto-pruned. This replaces an older note that described `load_pool()` as "non-destructive" for env-seeded credentials; the current docs describe the reconciliation behavior above instead.
- Concurrency: source has `DEFAULT_MAX_CONCURRENT_PER_CREDENTIAL = 1` in `agent/credential_pool.py`. Cross-process OAuth refreshes are serialized through a file lock on `auth.json` per the docs.
- Anthropic 429s cool down per model; a permanently rejected OAuth refresh marks the entry `dead` instead of benching it on a timer. Delegated subagents share the parent's pool for the same provider.
- Cache note from the docs: rotation to a different key means the new key has no cached prompt prefix.

## Fallback providers

> Sources: [Fallback Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [hermes_cli/fallback_config.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/fallback_config.py) · [agent/chat_completion_helpers.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/chat_completion_helpers.py)

Config is a top-level list; manage it with `hermes fallback` (`add`, `list`, `remove`, `clear`):

```yaml
fallback_providers:
  - provider: openrouter
    model: anthropic/claude-sonnet-4
  - provider: custom
    model: my-local-model
    base_url: http://localhost:8000/v1
    key_env: MY_LOCAL_KEY
fallback:
  min_switch_reset_seconds: 0   # skip the swap when a rate-limited primary reopens sooner than this
```

- `fallback_providers` (list, tried in order) supersedes the legacy singular `fallback_model`, which is still honored; both `provider` and `model` are required per entry. The docs say there are no environment variables for the primary chain.
- Triggers: 429 and 5xx after retries, 401/403/404 immediately, repeated invalid/empty responses. A router's "Connect timeout" 200-body is treated as invalid.
- Scope change from older notes: the old text said fallback is single-shot for the whole session. The user guide states fallback is turn-scoped: each new user message starts with the primary restored, and within one turn fallback activates at most once. Rate-limit reset times are honored (a primary whose reset has not elapsed stays benched). The internal `_fallback_activated` flag and `_try_activate_fallback()` (in `agent/chat_completion_helpers.py`) are described in the developer guide.
- Switch effects: credentials re-resolved, client rebuilt, model/provider swapped in place, reasoning effort re-resolved for the fallback model, retry counter reset. The prompt cache is invalidated.
- Where it applies: CLI, gateway, Desktop/TUI, subagents (`delegation.fallback_providers`, else only unpinned children inherit), unpinned cron jobs (a job with its own provider/model/base_url never falls back), and auxiliary tasks on `provider: auto`.

## Auxiliary model routing

> Sources: [Fallback Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers) · [Configuring Models](https://hermes-agent.nousresearch.com/docs/user-guide/configuring-models) · [Provider Runtime Resolution](https://hermes-agent.nousresearch.com/docs/developer-guide/provider-runtime) · [agent/auxiliary_client.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/auxiliary_client.py)

Side tasks (vision, compression, MCP helper operations, approval classification, title generation, review, triage specifier, and others) each have an `auxiliary.<task>` block with `provider`, `model`, `base_url`, `api_key`, and an optional task-specific `fallback_chain`.

```yaml
auxiliary:
  vision:
    provider: "auto"      # auto | openrouter | nous | codex | main | anthropic | ...
    model: ""
    base_url: ""          # if set, bypasses provider resolution
  compression:
    provider: auto
    model: ""
    fallback_chain:
      - provider: openrouter
        model: <fallback-model-id>
```

- `provider: auto` first tries the main provider and model for the task, then `auxiliary.<task>.fallback_chain`, then the top-level `fallback_providers`, then skips the task with a warning.
- The built-in discovery chains (OpenRouter, Nous Portal, custom endpoint, Codex, API-key providers; vision has its own order) run only when no main provider is selected (`model.provider: auto` or unset). With an explicit main provider and no declared fallback, an unavailable route skips the task rather than billing another logged-in account.
- `provider: main` resolves through the same shared runtime path as normal chat, so saved custom endpoints work. `main` is valid for auxiliary tasks only, not for `model.provider`.

## OpenRouter provider routing

> Sources: [Provider Routing](https://hermes-agent.nousresearch.com/docs/user-guide/features/provider-routing) · [LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)

Applies only to OpenRouter (Nous Portal ignores it; direct provider connections are unaffected). Keys under `provider_routing`: `sort` (`price`, `throughput`, `latency`), `only`, `ignore`, `order`, `require_parameters`, `data_collection` (`allow`/`deny`), and per-model overrides under `models`. They are sent as the top-level `provider` object via `extra_body` on chat requests and iteration-limit summaries; auxiliary tasks use their own `auxiliary.<task>.extra_body`. This is a different layer than fallback: it chooses sub-providers behind OpenRouter, not a different Hermes provider.

## Default model and the model catalog

> Sources: [Model Catalog](https://hermes-agent.nousresearch.com/docs/reference/model-catalog) · [Configuring Models](https://hermes-agent.nousresearch.com/docs/user-guide/configuring-models) · [hermes_cli/models_catalog_static.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/models_catalog_static.py) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example)

- "Silent default": the model used when a user never selected one. The catalog docs say it is the manifest entry marked `"default": true`, read cache-only at runtime, falling back to the in-repo constant `PREFERRED_SILENT_DEFAULT_MODEL`. In `hermes_cli/models_catalog_static.py` at this commit that constant is `"z-ai/glm-5.3"`, and the v0.23.0 release notes state the silent default moved from GLM-5.2 to GLM-5.3. The example model in `cli-config.yaml.example` is an illustrative value, not the silent default.
- `model.default` and `model.model` are interchangeable keys. A fresh install's `model:` is an empty string until `hermes setup` / `hermes model` upgrades it to a mapping (`provider`, `default`, `base_url`, `api_mode`).
- The curated pickers for OpenRouter and Nous Portal come from a remote JSON manifest with an in-repo fallback: config keys `model_catalog.enabled`, `url`, `ttl_minutes` (default 20), per-provider override URLs and `excluded_providers`. Pricing and context length are not in the manifest. Cache: `~/.hermes/cache/model_catalog.json`.
- Mid-session `/model` switches reset the prompt cache and prompt for confirmation above `model.switch_context_confirm_tokens` (default 100000 per the docs); `/model reset` returns to the configured default (release notes).

## Related: the passwords vault is not provider auth

> Sources: [Passwords & Logins](https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-vault)

`credential-vault` documents website logins, cards and addresses the agent can fill into pages without seeing the secret (encrypted under `~/.hermes/vault/`, per profile; `hermes vault ...`; optional keychain storage via `vault.key_storage`). It is separate from LLM provider credentials, which live in `.env`, `auth.json` pools and `config.yaml` as described above.

Not covered here: per-provider setup recipes (the LLM and Model Providers page), local-model servers, per-provider request options, and the full `ProviderProfile` hook list; follow the links above.

---

# 18. Gateway Internals (Messaging Platforms)

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [gateway/run.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run.py) · [gateway/session.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/session.py)

This section is a reference for how the messaging gateway is put together. It is written from the upstream docs and source at commit `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (the v0.23.0 line). Where it says "the docs state", that is what the page says; nothing here was executed. Several developer-guide pages disagree with each other or with source on details; those spots are called out inline so a reader can tell which source a statement comes from.

## Scope and how to read this

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/AGENTS.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/AGENTS.md)

Covered here: the process model, message flow, session keys, authorization, platform adapters, delivery, hooks, process and service management, multiplexed profiles, the shared session authority that local clients attach to, Group Chats, the Webapp, and monitoring. Not covered in depth (see the linked reference pages): per-platform setup, the full slash-command catalogue (section 24), the relay wire protocol, and the complete `config.yaml` schema (section 3).

Platform, tool and command counts are deliberately not stated. To enumerate platforms, use the comparison table on the Messaging Gateway page, `hermes gateway setup` (the interactive menu), or `/platform list` and `/platforms` on a running gateway.

## What the gateway is

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions)

The docs describe the gateway as a single long-running background process that connects to all configured platforms, handles sessions, runs cron jobs and delivers voice messages. Two changes matter for anyone working from older notes:

- **One gateway owns local sessions.** Per the v0.23.0 release notes and the developer guide, the classic CLI, TUI, Desktop, API server, ACP editors, bots and cron attach to each profile's single gateway instead of running their own agents against the same database. A chat is one conversation with one queue. See "Shared session authority" below.
- **Multiplexing is on by default.** One gateway process can serve the default profile plus every named profile (`gateway.multiplex_profiles`, described under "Multiplexed profiles").

`hermes gateway run` works with no OS service installed and even with no messaging platform configured; a service is a separate, opt-in choice (see "Optional service installation").

## Key files

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/platform_registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platform_registry.py) · [gateway/run_inbound.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_inbound.py)

| File | Purpose (per the docs) |
|------|---------|
| `gateway/run.py` | `GatewayRunner` facade. It composes sibling mixins: `gateway/run_*.py` (startup, adapters, inbound, turn, busy, goals, notifications, shutdown, ...) and `gateway/slash_commands_*.py` handlers |
| `gateway/run_inbound.py` | Inbound handling, including `_handle_message()` and the running-agent guard |
| `gateway/run_busy.py` | Busy-policy routing of slash commands while an agent is running |
| `gateway/session.py` | `SessionStore`, session-key construction (`build_session_key`) |
| `gateway/delivery.py`, `gateway/delivery_ledger.py` | Outbound delivery and the durable delivery ledger |
| `gateway/pairing.py` | DM pairing store |
| `gateway/authz_mixin.py` | `_is_user_authorized` and related checks |
| `gateway/channel_directory.py` | Maps chat IDs to human-readable names (used for cron delivery by name) |
| `gateway/hooks.py` | Hook discovery, loading and lifecycle dispatch |
| `gateway/mirror.py` | Cross-session message mirroring for `send_message` |
| `gateway/status.py` | Scoped token locks and profile-scoped process tracking |
| `gateway/platform_registry.py` | Adapter registry, factories, deferred (lazy) loaders for bundled platform plugins |
| `plugins/platforms/<name>/` | Bundled messaging adapters (most platforms: `adapter.py` + `plugin.yaml`) |
| `gateway/platforms/` | Shared `base.py` plus HTTP-ingress adapters (`api_server`, `webhook`, `msgraph_webhook`) |
| `gateway/relay/` | Generic relay adapter for connector-backed platforms (experimental) |
| `gateway/profile_routing.py`, `gateway/session_identity.py` | `profile_routes` matching and the frozen per-event routing identity |

The previous version of this guide listed adapters such as `telegram.py`, `discord.py` and `slack.py` directly under `gateway/platforms/`. Per the current developer guide, those now live as plugin-packaged adapters under `plugins/platforms/<name>/`; `gateway/platforms/` keeps the base class and the HTTP-ingress adapters.

## Architecture and message flow

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/platforms/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/base.py) · [gateway/run_inbound.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_inbound.py)

```text
 platform ──► adapter ──► MessageEvent ──► GatewayRunner._handle_message()
                                              │
              ┌───────────────┬───────────────┼───────────────────┐
              ▼               ▼               ▼                   ▼
       resolve session   authorization   slash command?     agent already
       key               check           (registry lookup)   running? (guard)
                                              │                   │
                                              └──────┬────────────┘
                                                     ▼
                                          AIAgent turn on the session
                                                     │
                                                     ▼
                                    reply via the delivering platform adapter
```

Message flow as the developer guide states it:

1. The platform adapter receives a raw event and normalizes it into a `MessageEvent`.
2. The base adapter checks its active-session guard. If an agent is already running for the session, the message is queued and an interrupt event is set. `/approve`, `/deny` and `/stop` bypass the guard and are dispatched inline.
3. `GatewayRunner._handle_message()` resolves the session key, checks authorization, checks whether the text is a slash command, intercepts commands such as `/stop` and `/status` when an agent is running, and otherwise creates an `AIAgent` and runs the conversation.
4. The response is sent back through the platform adapter.

Internal push wakes (for example background-process completions) enter through `gateway.wake.admit_internal_event`. The docs state that a handler returning `None` is not acceptance: an internal event counts as admitted only once its process-local `_gateway_accepted` receipt is set after scheduling or queue insertion. Custom adapters that override ingress should delegate internal events to `BasePlatformAdapter.handle_message()`.

### Session key format

> Sources: [Gateway Session Lifecycle](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-session-lifecycle) · [gateway/session.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/session.py)

Source (`build_session_key`) documents the layout as:

```text
<ns>:<platform>:<chat_type>[:<slack scope_id>][:<chat_id>][:<thread_id>][:<user>]
```

The namespace `<ns>` is `agent:main` for the default profile; a named profile gets `agent:<name>` (a profile literally named `main` is marked `agent:main~` so it can never collide with the default). The session-lifecycle page lists these `chat_type` values: `dm`, `group`, `channel`, `thread`. Examples from that page (default profile):

| Scenario | Key |
|---|---|
| DM | `agent:main:telegram:dm:<chat_id>` |
| DM with thread | `agent:main:telegram:dm:<chat_id>:<thread_id>` |
| Group chat, per-user isolation | `agent:main:telegram:group:<chat_id>:<user>` |
| Thread in a group, shared | `agent:main:discord:group:<chat_id>:<thread_id>` |
| Channel | `agent:main:slack:channel:<chat_id>` |

Rules the docs and `build_session_key` agree on:

- DMs are isolated per `chat_id`; without a `chat_id` the sender id is used; with neither, all DMs on that platform collapse to one session. In DM keys the participant precedes the thread; in group keys the thread precedes the participant.
- `group_sessions_per_user` (default `true`) appends the participant id to group/channel keys. `thread_sessions_per_user` (default `false`) means threads are shared by default.
- WhatsApp identifiers are canonicalized so JID/LID alias flips do not fragment a session.
- Slack's `scope_id` is inserted before the chat id.
- Never construct keys by hand; call `build_session_key()` from `gateway/session.py`.

Documentation inconsistency: `gateway-internals.md` shows an example `agent:main:telegram:private:<id>`, while the session-lifecycle page and the `chat_type` enumeration use `dm`. This guide follows the session-lifecycle page and source.

### Two-level message guard

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/run_busy.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_busy.py)

While an agent is running, an incoming message passes two guards in sequence:

1. **Base adapter** (`gateway/platforms/base.py`): checks `_active_sessions`; if active, queues the message in `_pending_messages` and sets an interrupt event.
2. **Gateway runner** (`gateway/run_inbound.py`): checks `_running_agents`; intercepts `/stop`, `/new`, `/queue`, `/status`, `/approve` and `/deny`; anything else triggers `running_agent.interrupt()` under the default busy mode.

Commands that must reach the runner while the agent is blocked (such as `/approve`) are dispatched inline with `await self._message_handler(event)`, bypassing the background-task system to avoid races.

User-facing busy behavior is configurable (`display.busy_input_mode`: `interrupt` default, `queue`, or `steer`; per-profile, no environment override):

```yaml
display:
  busy_input_mode: steer   # or queue, or interrupt (default)
  busy_ack_enabled: true   # false suppresses the chat acknowledgement
  busy_text_debounce_seconds: 0.35
  busy_text_hard_cap_seconds: 1.0
```

### Slash command dispatch

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/run_busy.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_busy.py) · [Gateway command parity](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-command-parity)

All gateway slash commands go through one pipeline:

1. `resolve_command()` (`hermes_cli/commands.py`) maps input to a canonical name (aliases, prefix matching).
2. The name is checked against `GATEWAY_KNOWN_COMMANDS`.
3. `_handle_message()` looks the handler up by name (`_handle_<name>_command` on the `gateway/slash_commands_*.py` mixins) through `_command_handler_table` over `_IDLE_COMMANDS` / `_PLAIN_COMMANDS`; the docs state there is no `if canonical == ...` chain.
4. Some commands are config-gated (`gateway_config_gate` on `CommandDef`).

While `_quick_key in self._running_agents`, `_dispatch_busy_slash_command()` routes each command by its `CommandDef.busy_policy` / `busy_handler`: a mid-run variant if one exists, else the normal handler if the policy allows, else a rejection message. `/stop`, `/new`, `/approve`, `/deny`, `/queue` and `/status` have mid-run handlers.

The generated "Gateway command parity" page tabulates, for every registry command, what the shared gateway (`slash.exec`), the classic CLI, the Ink TUI, Desktop and ACP each do with it. The page states 105 registry commands at generation time (a generated, SHA-dated figure; regenerate rather than rely on it).

## Authorization

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [gateway/authz_mixin.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/authz_mixin.py)

The default is deny: the docs state that users who are not on an allowlist or paired via DM are denied. Three sources describe the check order, and they differ:

| Source | Order |
|---|---|
| `gateway-internals.md` | per-platform allow-all, platform allowlist, DM pairing, global allow-all, deny |
| `security.md` | per-platform allow-all, DM pairing approved list, platform allowlists, `GATEWAY_ALLOWED_USERS`, global allow-all, deny |
| `_is_user_authorized` docstring in source | trusted-upstream delegation, chat-scoped group allowlists, `{PLATFORM}_ALLOW_BOTS`, per-platform allow-all, adapter role auth, pairing store, env/config allowlists, `GATEWAY_ALLOW_ALL_USERS`, deny |

The source order is the authoritative one (the others are simplifications). Practical points the docs agree on:

- Allowlists are comma-separated IDs in `~/.hermes/.env`, per platform (for example `TELEGRAM_ALLOWED_USERS`, `DISCORD_ALLOWED_USERS`) plus a cross-platform `GATEWAY_ALLOWED_USERS`.
- `DISCORD_ALLOW_ALL_USERS`-style flags are per-platform allow-all; `GATEWAY_ALLOW_ALL_USERS=true` is global. The global flag can also be set as `gateway.allow_all_users: true` in `config.yaml`; it is bridged to the environment variable at startup (an explicit env var wins), and in a multiplexed gateway a secondary profile sets it in its own `.env`.
- Webhook events are HMAC-verified, and adapters registered with `trusted_inbound=True` (for example a service event bus with no human sender) skip user allowlists and pairing. The adding-adapters guide warns never to set it for a chat platform.

### DM pairing

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [CLI commands reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [gateway/pairing.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/pairing.py)

The old guide showed an admin `/pair` command. The current docs do not document one. As documented, an unknown user who DMs the bot receives a one-time pairing code; the operator approves it from the CLI:

```bash
hermes pairing approve telegram <code>   # approve a pending code
hermes pairing list                      # pending + approved users
hermes pairing revoke telegram <user-id> # remove access
hermes pairing clear-pending
```

The docs state codes expire after 1 hour, are rate-limited and use cryptographic randomness; unknown email senders are ignored unless email pairing is explicitly enabled. Pairing state persists in `gateway/pairing.py`. Source also shows an `unauthorized_dm_behavior` setting (`pair`, `ignore`, `decline`) in `gateway/config.py`, and the adding-adapters guide documents an `allowlist_suppresses_pairing` flag under which a platform with an allowlist configured ignores unknown DM senders instead of sending a code.

### Admins and regular users

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [gateway/slash_access.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/slash_access.py)

Beyond "can this person reach the bot", an optional admin/user split gates slash commands per platform and per scope (DM versus group/channel). If `allow_admin_from` is not set for a scope the split is disabled and every allowed user has full access (backward compatible).

```yaml
gateway:
  platforms:
    discord:
      extra:
        allow_from: ["<user-id-1>", "<user-id-2>"]
        allow_admin_from: ["<user-id-1>"]          # admins: all slash commands
        user_allowed_commands: [status, model]      # what non-admins may run
        group_allow_admin_from: ["<user-id-1>"]     # optional separate group/channel scope
        group_user_allowed_commands: [status]
```

`/whoami` reports the active scope and tier. The always-allowed floor for regular users is `/help` and `/whoami`. The docs warn that `/login` binds the whole install and that operators of shared gateways must set `allow_admin_from`.

### Other bots in a chat

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [gateway/bot_loop_guard.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/bot_loop_guard.py)

A sender is treated as a bot when the platform says so; on platforms without a bot flag, list peers under `bot_users`. A bot sender is admitted only through that platform's `*_ALLOW_BOTS` setting, never receives runtime notices, and `gateway.bot_loop_guard` ends bot-to-bot exchanges (documented defaults: `max_hops` 4; `enabled: false` turns the rules off).

## Platform adapters

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/platforms/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/base.py)

Every adapter extends `BasePlatformAdapter` and implements `connect()`, `disconnect()` and `send()` (with optional `send_typing()` and `get_chat_info()`); inbound events are normalized to a `MessageEvent` and forwarded with `handle_message()`.

The Messaging Gateway page's comparison table lists these platforms at this SHA (capabilities per platform are in that table; no total is asserted here): Telegram, Discord, Slack, Google Chat, WhatsApp, WhatsApp Cloud API, Signal, SMS, Email, Home Assistant (plugin), Mattermost, Matrix, DingTalk, Feishu/Lark, WeCom, WeCom Callback, Weixin, BlueBubbles, Photon (iMessage), QQ, Yuanbao, Microsoft Teams, LINE, ntfy, Raft, IRC, Buzz and SimpleX. The same section of the docs also documents the API server (OpenAI-compatible; see section 23), Webhooks, A2A, Open WebUI via the API server, a Microsoft Graph webhook listener and a Teams Meetings pipeline. Hermes Relay is described separately as an experimental connector system rather than a chat platform.

Layout per the developer guide: most platforms are plugin-packaged adapters under `plugins/platforms/<name>/` (for example `telegram`, `discord`, `slack`, `whatsapp`, `whatsapp_cloud`, `matrix`, `mattermost`, `email`, `sms`, `signal`, `irc`, `bluebubbles`, `weixin`, `qqbot`, `yuanbao`); `gateway/platforms/` holds `base.py`, `webhook.py`, `api_server.py` and `msgraph_webhook.py`.

### Deferred loading and the platform registry

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/platform_registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platform_registry.py)

Bundled `kind: platform` plugins register cheap deferred loaders so platform SDKs import only when the gateway starts, delivers, or runs setup/status, not on plain `hermes chat`. Resolution loads one adapter on lookup; full enumeration runs pending loaders only where every platform is needed.

### Token locks

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/status.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/status.py) · [gateway/restart.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/restart.py)

Adapters with unique credentials call `acquire_scoped_lock()` in `connect()` and `release_scoped_lock()` in `disconnect()`, so two profiles cannot use the same bot token at once. A lock conflict is emitted as `{scope}_lock` with `retryable=True`, so a mid-run reconnect can recover. At startup a live foreign holder is treated as a configuration conflict: the platform is parked fatal, and with nothing else connected the gateway exits with code 78 (`EX_CONFIG`, `gateway_state=startup_failed`) so supervisors stop restarting it (the docs describe the systemd, s6 and launchd mechanisms).

### Relay (experimental)

> Sources: [Hermes Relay](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/relay) · [Relay and Connector contract](https://hermes-agent.nousresearch.com/docs/developer-guide/relay-connector-contract) · [gateway/relay/descriptor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/relay/descriptor.py)

Both pages are marked experimental and subject to change without a deprecation cycle. When `GATEWAY_RELAY_URL` or `gateway.relay_url` is configured, the gateway registers a `relay` platform and dials out to an external connector over one WebSocket. The connector holds the platform credentials and sends a `CapabilityDescriptor` at handshake (`contract_version`, `platform`, `max_message_length`, streaming/edit/thread capabilities, markdown dialect, and optional fields); inbound events, outbound actions and interrupt frames ride that single socket, so the gateway opens no inbound port. `hermes gateway enroll` stores relay credentials. Shared-ingress platforms (the WhatsApp bridge and Relay) run on the default profile only under multiplexing.

### Writing a new adapter

> Sources: [Adding a Platform Adapter](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-platform-adapters) · [gateway/platform_registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platform_registry.py)

The guide recommends the plugin path: a directory with `plugin.yaml` (`kind: platform`, `requires_env` / `optional_env`) and `adapter.py` with a `register()` entry point calling `ctx.register_platform(...)`. The registration automatically wires adapter creation, config parsing, authorization env vars (`allowed_users_env` / `allow_all_env`), env-only auto-enable, the YAML-to-env bridge, cron delivery (`cron_deliver_env_var`), `hermes config` entries, the send engine, the channel directory, system-prompt hints, chunking and setup menus. Optional `PlatformEntry` fields include `trusted_inbound`, `display_tier`, `shared_env_prefixes`, `group_allowed_chats_env` and `allowlist_suppresses_pairing`; the guide recommends feature-detecting them with `dataclasses.fields(PlatformEntry)` for compatibility with older cores. The built-in path (editing the platform enum, config, runner, toolsets and docs) is documented as a multi-file checklist. The plugin system itself is section 20.

## Sessions and restart recovery

> Sources: [Gateway Session Lifecycle](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-session-lifecycle) · [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [gateway/session.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/session.py)

Data model per the lifecycle page: `SessionSource` (where a message came from: platform, chat id and type, user id, thread id, optional alternate ids, `is_bot`, guild/parent ids, message id, `role_authorized`), `SessionEntry` (session key to current `session_id`, token and cost counters, and state flags), and `SessionStore` (an in-memory dict persisted to `sessions.json`, with SQLite `SessionDB` as the canonical transcript and metadata store; see section 15).

State flags and what they do:

| Flag | Meaning |
|---|---|
| `suspended` | Hard wipe: the next access forces a new `session_id` (set by `/stop` or stuck-loop escalation after 3+ consecutive restart failures) |
| `resume_pending` | Soft recovery: the next access keeps the same `session_id`; cleared after the next successful turn. Reasons: `restart_timeout`, `shutdown_timeout`, `restart_interrupted` |
| `is_fresh_reset` | Set by explicit `/new` or `/reset` |

Session continuity changed relative to older notes. The Messaging Gateway page and lifecycle section 6 state that gateway conversations do not reset after inactivity or at a daily boundary: use `/new` or `/reset` for an explicit new conversation; context compression stays automatic; core ignores legacy `session_reset` settings (startup and `hermes doctor` warn if present), and time-based resets are available only by installing the catalog plugin `hermes-session-reset-policy`. Note that the state-transition diagram on the lifecycle page still draws a "policy says reset (idle/daily)" branch; that diagram is stale relative to the same page's section 6, and this guide treats it as historical.

Restart recovery (documented flow): on a start without the `.clean_shutdown` marker the gateway runs crash recovery, which acts only on durable active-turn markers (not on recency). A marked turn with a persisted reply goes to the delivery ledger (delivered once with a "Recovered reply" notice); a marked turn without one becomes `resume_pending` and auto-resumes once. Stuck-loop detection counts consecutive restarts in `restart_counts.json` and suspends a session active across 3+ restarts. Inbound messages queue while the startup restore runs.

Other documented session behaviors: `/model` overrides persist across restarts (credentials are re-resolved, never stored); `/sessions` lists chat-scoped sessions (`/sessions all` is admin-only); per-channel `channel_overrides` (model, provider, system_prompt, optional `allow_silence`) resolve by exact chat/thread id then parent id; and the silence tokens (`[SILENT]`, `SILENT`, `NO_REPLY`, `NO REPLY` and Chinese renderings) suppress delivery only when they are the entire final response.

### Memory flush lifecycle

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)

When a memory provider plugin is enabled the gateway builds an `AIAgent` per message with the session ID and routes provider tool calls through the memory manager. Explicit boundaries (`/new`, `/reset`, `/resume`) flush and finalize the outgoing session; idle time and daily boundaries never finalize it. Resource-only TTL/LRU/memory-pressure eviction commits the cached transcript to memory providers before releasing the agent's clients but does not close the conversation; the next turn reloads the same transcript. See section 13 for providers.

## Delivery

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/delivery_ledger.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/delivery_ledger.py) · [gateway/delivery.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/delivery.py)

Delivery modes per the developer guide: direct reply to the originating chat, home-channel delivery (cron outputs and background results), explicit-target delivery (`platform:<target>`, also exposed through the `hermes send` CLI), and cross-platform delivery. A bare platform name delivers to that platform's configured home channel; the original guide's target table (`origin`, `local`, `telegram:<chat_id>:<thread_id>` and so on) is now maintained in the cron documentation (section 19).

Delivery ledger (Messaging Gateway page): final responses are recorded in a durable ledger in `state.db` around each platform send, with honest at-least-once semantics. A send that never started is redelivered as-is; one that was mid-send is redelivered with a visible "Recovered reply ... may be a duplicate" prefix; flood-control rejections are retried after the recorded penalty; other rejections retry with backoff (30 s, then 2 min). Redelivery is bounded (3 attempts, 24-hour freshness, delivered rows pruned after 7 days). `gateway.delivery_ledger: false` disables it.

Cron delivery and session history: the developer guide states cron job deliveries are not mirrored into gateway session history by default and live in the cron job's own session (avoiding message-alternation violations). An opt-in "continuable" mode (`cron.mirror_delivery`, or per-job `attach_to_session`) can seed or mirror a labelled user turn; see section 19.

## Hooks

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Event Hooks](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks) · [gateway/hooks.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/hooks.py)

Gateway hooks are discovered from `<profile home>/hooks/` (so `~/.hermes/hooks/` for the default profile and one directory per served profile under multiplexing). Each hook is a directory with `HOOK.yaml` and `handler.py`; the handler must define `handle(event_type, context)` (sync or async), and errors are caught and logged rather than crashing the agent.

| Event | When it fires |
|---|---|
| `gateway:startup` | Gateway process starts (context: active platforms) |
| `session:start` / `session:end` / `session:reset` | Session created / ended / user ran `/new` or `/reset` |
| `session:compress` | Context compression completed for a session |
| `agent:start` / `agent:step` / `agent:end` | Message processing begins / each tool-calling iteration / finished |
| `reaction:added` / `reaction:removed` | Emoji reaction events (documented as Slack-only currently; needs the relevant scopes and event subscriptions) |
| `command:*` | Any slash command executed (wildcard matching) |

## Process and service management

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [CLI commands reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals)

```bash
hermes gateway                  # run in foreground (also: hermes gateway run)
hermes gateway setup            # interactive platform setup
hermes gateway install          # user service (Linux systemd) / launchd (macOS)
sudo hermes gateway install --system   # Linux boot-time system service
hermes gateway start | stop | restart | status
hermes gateway list             # all profiles and whether each gateway runs
```

The CLI reference also lists `ensure`, `attach`, `migrate`, `migrate-legacy`, `uninstall` and `enroll`. Windows uses a Scheduled Task per the Messaging Gateway page, and the docker image uses s6 supervision.

Profile scoping and the `--all` flag are described differently by two pages, so both are reported: the CLI reference says `--all` on `start`/`restart`/`stop` acts on every profile's gateway; the multi-profile guide says that under multiplexing `start --all` and `restart --all` mean the one host multiplexer and never sweep every gateway process. `stop --all` is described in the developer guide as using global process scanning to stop all gateways (used during updates). The PID file is `~/.hermes/gateway.pid` (profile-scoped); liveness uses a PID plus start-time fingerprint, never bare PID existence. Under multiplexing there is one gateway process per host, owned by whichever profile launched it; `hermes -p x gateway stop` for a served secondary parks only that profile, and a second `gateway run` for a served profile attaches and exits 0 (exit 75 under a service supervisor, so the unit retries).

### Optional service installation

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals)

`hermes setup` and `hermes gateway setup` ask interactive users once whether to install the service. The answer is stored as `gateway.service_install_choice` (`null`, `install`, `decline`); an unset value is not permission to install, noninteractive setup and imports do not install a missing service, and `hermes gateway install` is the explicit opt-in. Without a service, messaging and scheduled work stop at logout or reboot.

### `hermes gateway ensure` and `attach`

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [hermes_cli/gateway_runtime_attach.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/gateway_runtime_attach.py)

Clients (Desktop, `hermes chat`, the TUI) discover or start the profile's runtime with `hermes gateway ensure --json --timeout 60`. It never installs or rewrites services; an existing service takes precedence, and it fails closed on ambiguous ownership (reason codes such as `profile_mismatch`, `service_account_mismatch`, `service_identity_unverified`). Exit codes: `0` ready, `2` invalid invocation, `3` incompatible protocol, `4` authorization or profile mismatch, `5` deadline reached, `6` draining or update in progress, `7` inaccessible or startup failure. `hermes gateway attach --json` is the public attach step for a local client with its own UI: it prints one JSON line with a WebSocket `url`, `protocols` and a single-use `ticket` (30-second expiry, bound to the profile and the owner's instance id); flags are `--start`, `--recover` and `--timeout S`. Dial from loopback with no `Origin` header and speak `hermes-gateway-v1`.

### Liveness, watchdogs and signals

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [Gateway Monitoring](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-monitoring)

Documented behaviors: `SIGUSR2` appends every thread's stack to a faulthandler log on Linux and macOS; an out-of-loop watchdog (`gateway.loop_watchdog_probe_interval_s` default 30 s, `gateway.loop_watchdog_max_strikes` default 3, disable with `gateway.loop_watchdog: false`) dumps stacks, stamps `gateway_state.json` as degraded and exits with code 75; housekeeping refreshes `gateway_state.json` every 60 s as a heartbeat (stale after 120 s); and an optional Linux systemd watchdog is enabled with `gateway.systemd_watchdog_seconds` (default 0) followed by `hermes gateway install --force`. Circuit breaker: repeated retryable adapter failures auto-pause the adapter; it does not auto-resume, and `/platform resume <name>` re-arms it (`/platform list`, `pause`, `resume`).

## Multiplexed profiles

> Sources: [Multiplexing Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/multiplexing-gateway) · [Running Many Gateways at Once](https://hermes-agent.nousresearch.com/docs/user-guide/multi-profile-gateways) · [hermes_cli/profiles.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/profiles.py)

Per the docs, `gateway.multiplex_profiles` defaults to `true` with a safety rule: an unset flag is settled at each boot by a preflight (the same one as `hermes gateway migrate --multiplex`); if nothing blocks, the gateway multiplexes and writes `true` into the default profile's `config.yaml`. An explicit `true` bypasses the preflight; `false` is documented as retired and is rewritten to `true`. A named profile can set `gateway.standalone: true` as a temporary compatibility shim to keep its own gateway. `GATEWAY_MULTIPLEX_PROFILES` overrides the unset decision.

What it changes: the default profile plus every served profile get their own adapters, secrets, sessions and cron ticks, sharing one event loop, one HTTP listener, one process lock and one status surface. The invariant the design states: profile A's turns must never observe profile B's state. Mechanisms the docs name:

- **Scopes.** Every inbound event composes two context-local scopes (a HERMES_HOME override and a secret scope) before profile-owned code runs; nothing is written to `os.environ`. `get_secret()` fails closed once multiplexing is active.
- **Routing.** One bot per profile (separate credentials), `/p/<profile>/` URL prefixes for HTTP platforms, and `gateway.profile_routes` for several communities sharing one bot token.
- **Session lanes.** Session keys are namespaced by profile; every event carries a frozen routing identity separating the bot that received it from the profile that executes it, and delivery goes through the receiving bot.
- **Persistence.** Session DB handles resolve per profile at call time; pairing stores are per served profile.
- **Known limitations** (as listed at this SHA): MCP discovery and tool registration are process-global; terminal env, the built-in tool registry, provider registries and the HTTP listener are process-wide. Multiplexing isolates profiles, not end users.

```yaml
gateway:
  multiplex_profiles: true
  profile_routes:
    - name: acme-server
      platform: discord
      guild_id: "<guild-id>"
      profile: acme
    - name: acme-support
      platform: discord
      guild_id: "<guild-id>"
      chat_id: "<channel-id>"
      profile: acme-support
```

`profile_routes` match conjunctively with additive specificity (`user_id` 16, `thread_id` 8, `chat_id` 4, `guild_id` 2), require multiplexing on, and drop (rather than misdeliver) events whose target profile is not served. Authorization is decided by the receiving bot's profile. Sender routing selects a profile and is not deny-by-default authorization.

## Shared session authority (local clients)

> Sources: [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions) · [Classic gateway client](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-classic-cli) · [Gateway command parity](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-command-parity)

The composed gateway listener can create fresh local CLI, TUI or GUI sessions as canonical admissions; the existing `GatewayRunner` and `TurnRunner` own execution, approvals, tools, storage and event delivery. Documented properties:

- `session.create` returns a stable logical session id, reserves the session identity before returning, and supports lost-ACK retry with a client `request_id`. `prompt.submit` returns a durable admission receipt, not inference completion. Closing every viewer leaves the execution and its pending approval alive; a new viewer resumes with `session.resume`.
- Verified operators of the same gateway profile can attach to canonical local sessions created through another authentication path (shared access, not shared identity).
- Normal `hermes --cli`, `hermes chat` and `hermes -z` use the gateway authority. The classic client prints the persisted `Session:` id, detaches on `/quit`, `/exit`, `/detach` or EOF, and `/stop` sends a generation-fenced interrupt. The classic CLI is documented as not having full parity; see the parity table.
- A turn that was `started` when its owning gateway died is recovered as `unknown` and pauses that session's queue; the docs describe resolving it with `prompt.resolve_unknown` (Desktop "Discard", CLI `/discard <admission_id>`, `hermes sessions discard <id> --yes`). The input is never replayed automatically. Messaging users on a paused session are told once that `/reset` starts a fresh conversation.
- Native admission startup recovery re-runs only never-started queued work after adapters connect, and rechecks authorization.
- Revision-checked metadata: `session.mutate` (rename, archive; canonical reset, delete, rewind, model, branch and compression use the same receipt boundary) with `expected_revision`, idempotent receipts and `revision_conflict` / `admission_conflict` errors.
- Private native HTTP authentication uses a fresh one-use grant from the same-user control socket (30-second expiry) in the `X-Hermes-Gateway-Ticket` header, requiring a loopback peer and no `Origin` header. The docs call this a same-OS-user administrative boundary, not an OAuth bypass.

Canonical group metadata (see Group Chats below) and ancillary reads (`session.control.read`, `process.list`, `subagent.list`, `subagent.tail`) are also exposed on the authenticated WebSocket; the docs state ancillary polls never restore a cold session.

## Group Chats (Bot Mode)

> Sources: [Bot Mode](https://hermes-agent.nousresearch.com/docs/user-guide/bot-mode) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals)

Bot Mode turns profiles into a roster of named Bots; it ships built into the desktop app and is documented as on by default. A Bot is a profile; each has a canonical persistent Bot Chat. Bots can also share Group Chats ("rooms"). The docs describe two engines: a gateway group chat (the gateway owns the log and schedules turns through a durable driver, so a discussion keeps running with Desktop closed) and a classic, Desktop-driven group chat (rounds run in Desktop and pause while it is closed). A gateway-owned room can also seat a text-only Bot from another gateway over JSON-RPC.

On the gateway side, the authenticated canonical WebSocket exposes `groups.capabilities`, `groups.list`, `groups.state` and `groups.log` (with `session:read`) and `groups.create`, `groups.rename`, `groups.disband` (with `session:control`). The developer guide stresses this is metadata management, not hosted-room inference: the surface reports `driver: false` and does not advertise send, retry, approval or peer-grant operations. `profiles.list` returns the authenticated profile first, then served siblings, as discovery metadata. Group-chat turn handling (stop directives, member holds, lead/auto routing) is user-guide behavior; read the Bot Mode page rather than relying on this summary.

## Webapp (off by default)

> Sources: [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [CLI commands reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions)

The gateway can serve the Desktop workspace in a browser on its own listener. The configuration page states it is off until turned on, and the v0.23.0 release notes repeat that nothing is served until `webapp.enabled` or Web Access is enabled.

```yaml
webapp:
  enabled: false     # true: the gateway serves the Webapp
  host: 127.0.0.1    # 127.0.0.1, ::1 or localhost; or 0.0.0.0 / :: (dashboard sign-in required)
  port: 9120
  public_url: ""     # https address a reverse proxy or tunnel serves it at (sign-in required)
```

`hermes webapp` builds the bundle if needed, starts the gateway if not running and opens a one-use sign-in link (valid 60 seconds); options include `--no-open`, `--skip-build`, `--force-build`, `--build-only`, `--status` and `--sign-out`. Sign-in turns on for a wildcard `host` or when `public_url` is set. The gateway reads these settings at start, so run `hermes gateway restart` after changing them.

## Background process notifications

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Background completion backlogs](https://hermes-agent.nousresearch.com/docs/developer-guide/completion-backlog-delivery) · [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/)

`terminal(background=true, notify_on_complete=true)` arms a gateway watcher that detects completion and triggers a new agent turn. Verbosity is `display.background_process_notifications` (or `HERMES_BACKGROUND_NOTIFICATIONS`): `concise` (default), `all`, `result`, `error`, `off`. Adapter acceptance is at-least-once admission, not proof that a model turn or reply completed; completions that arrive while a session is busy fold into one queued turn. The separate completion-backlog page covers interactive surfaces (classic CLI, TUI, Desktop) and states it does not replace the gateway's own route-keyed batching.

## Monitoring

> Sources: [Gateway Monitoring](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-monitoring)

The previous guide's "Gateway Runtime Status" notes (`busy`, `drainable`, `active_agents`) are superseded by the monitoring page. It documents an opt-in, content-free OTLP/HTTP export of gateway health and cron scheduler state (gauges such as `hermes.gateway.up`, `hermes.gateway.active_agents`, `hermes.gateway.background_work`, `hermes.platform.up`, plus cron ticker heartbeat, catch-up and overdue counts, and lifecycle traces). It never exports prompts, messages, tool arguments or job names. Enable it under `monitoring.gateway_health_export` and `monitoring.export.otlp` in `config.yaml`; the page also covers collector examples, alert queries and how to extend the plane.

## Kanban with several gateways

> Sources: [Kanban multi-gateway deployment](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-multi-gateway)

Briefly: when several gateway processes run, only one owns the kanban dispatcher (`kanban.dispatch_in_gateway: true` by default); others set it to `false` (or `HERMES_KANBAN_DISPATCH_IN_GATEWAY=false`). Notification delivery is profile-owned: each gateway polls only subscriptions for profiles whose adapters it hosts. See the kanban documentation for the rest.

## Not covered here

> Sources: [Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals)

Per-platform setup pages (each linked from the Messaging Gateway "Next Steps" list), the `/login` flow details, mid-run plugin loading (`gateway/run_plugin_rewire.py`; handlers only, tools and prompt sections wait for the next session), the relay wire contract beyond the handshake, frozen local launch credentials, tool progress and typing-indicator options, the `channel_overrides` schema for every platform, and the full `hermes gateway` flag set. The relevant pages are linked in the Sources lines above.

---

# 19. Cron Internals

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [cron/scheduler.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/scheduler.py) · [cron/jobs.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/jobs.py)

The cron subsystem provides scheduled task execution: one-shot delays, recurring intervals and cron-expression jobs, with skill injection, optional pre-run scripts and cross-platform delivery. This section is a reference written from the docs and source at commit `9f23c4311a4c4837078081db4f3cd8deab23c0fe`. It describes documented behavior; nothing was executed. The earlier version of this section included a dated list of "cron improvements" with PR numbers; those items are folded into the current behavior below or dropped where the docs no longer support them.

## Key files

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [cron/jobs_time.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/jobs_time.py) · [tools/cronjob_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/cronjob_tools.py)

| File | Purpose |
|------|---------|
| `cron/jobs.py` | Job model, storage, atomic read/write of `jobs.json`, due-scan, `mark_job_run` |
| `cron/jobs_time.py` | Per-job `timezone` / `grace_minutes` validation and due-scan zone |
| `cron/scheduler.py`, `cron/scheduler_tick.py` | Scheduler loop: due-job detection, execution, repeat tracking |
| `cron/scheduler_provider.py` | `CronScheduler` provider resolution and the built-in in-process provider |
| `cron/executions.py` | Execution ledger (`executions.db`) |
| `cron/delivery_queue.py`, `cron/delivery_outcome.py` | Durable delivery queue and write-back of delivery outcomes |
| `cron/missed_runs.py`, `cron/occurrences.py` | Missed-occurrence accounting |
| `cron/cancellation.py` | Cancel requests for in-flight runs |
| `tools/cronjob_tools.py` | Model-facing `cronjob_manage` tool (toolset `cronjob`) |
| `hermes_cli/cron.py` | `hermes cron` subcommands |
| `plugins/cron_providers/chronos/` | Chronos managed-cron provider (see below) |

## Schedule formats

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [cron/jobs.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/jobs.py)

| Format | Example | Behavior |
|--------|---------|----------|
| One-shot delay | `in 30m`, `in 2h`, `in 1d` | Runs once after the duration |
| Interval | `every 2h`, `every 30m`, `30m` | Recurring. A bare duration such as `30m` is recurring |
| Natural day/time | `every monday 9am`, `weekdays at 9am`, `daily at 7am` | Recurring; compiled to cron expressions (needs `croniter`, installed by default) |
| Cron expression | `0 9 * * *`, `0 9 * * MON-FRI` | 5-field syntax; named weekdays/months accepted |
| ISO timestamp | `2026-03-15T09:00:00` | One-shot; an offset-less timestamp is read in the configured Hermes timezone |

Documentation conflict: the developer guide's schedule table lists `30m` under "Relative delay ... one-shot", while the user guide and the `parse_schedule` docstring say `30m` and `every 30m` are recurring and `in 30m` is the one-shot form. This guide follows the user guide and source. Default repeat: one-shot 1, intervals and cron expressions forever (override with `repeat`). A job can pin its own timezone (`--timezone`, the expression is read as that zone's wall clock; ignored for interval jobs) and its own catch-up window (`--grace-minutes`, 0 to 1440).

## Job record and storage

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)

Jobs live in `~/.hermes/cron/jobs.json` (per profile) with atomic write semantics (temp file, then rename). Run output goes to `~/.hermes/cron/output/<job_id>/<timestamp>.md`. The docs advise managing jobs through `cronjob_manage`, `hermes cron edit` or `/cron`, not by editing the file. Documented record shape (trimmed):

```json
{
  "id": "a1b2c3d4e5f6",
  "name": "Daily briefing",
  "prompt": "Summarize today's AI news and funding rounds",
  "schedule": {"kind": "cron", "expr": "0 9 * * *", "display": "0 9 * * *"},
  "skills": ["ai-funding-daily-report"],
  "deliver": "telegram:<chat_id>",
  "repeat": {"times": null, "completed": 42},
  "state": "scheduled",
  "enabled": true,
  "next_run_at": "2025-01-16T09:00:00Z",
  "last_run_at": "2025-01-15T09:00:00Z",
  "last_status": "ok",
  "model": null,
  "provider": null,
  "script": null
}
```

`timezone` and `grace_minutes` are top-level fields (not inside `schedule`); older records with a single `skill` field are normalized to `skills: [skill]` at load. Other per-job fields named in the user guide include `workdir`, `no_agent`, `context_from`, `continuity`, `catch_up`, `attach_to_session` and a model/provider/reasoning pin.

Related per-profile stores: `~/.hermes/cron/executions.db` (execution ledger, also holds failure incidents), `cron/deliveries.db` (durable delivery queue) and a file lock `~/.hermes/cron/.tick.lock`.

### Job states and `last_status`

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals)

Lifecycle states: `scheduled`, `paused`, `completed` (one-shot fired or repeat exhausted) and `running` (transient). Documentation conflict at the repeat limit: the developer guide says the job becomes `completed`, the Chronos contract says `mark_job_run` deletes the job at the limit, and the `mark_job_run` docstring in source says the record is retired "as a terminal completion". This guide follows the developer guide and source.

`last_status` literals and their detail fields: `ok`; `delivery_queued` (run succeeded, send handed to the durable queue and not yet settled; `last_delivery_queued`); `error` (`last_error`); `delivery_failed` (run succeeded but output never reached the target; `last_delivery_error`); `blocked_config` (pre-dispatch validation refused to burn a run). The docs warn that a consumer must never test `== "ok"` for "the user got their result". The queue row states `pending`, `delivering`, `delivered | failed | suppressed`; a row whose claiming gateway died mid-send is fenced `unknown` and never retried; `settle_queued_delivery` writes the terminal outcome back onto the originating run, fenced on `execution_id`.

## The scheduler tick

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [cron/scheduler_tick.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/scheduler_tick.py)

The docs describe a periodic tick, default every 60 seconds, run by the gateway. The developer guide's simplified view:

```text
tick()
  1. Acquire scheduler lock (prevents overlapping ticks)
  2. Load all jobs from jobs.json
  3. Filter to due jobs (next_run <= now AND state == "scheduled")
  4. For each due job:
       a. mark running
       b. fresh AIAgent session (no conversation history)
       c. load attached skills in order (injected as user messages)
       d. run the job prompt
       e. deliver the response to the configured target
       f. update run metadata and compute next run
       g. repeat exhausted -> completed, otherwise -> scheduled
  5. Write jobs back to jobs.json
  6. Release scheduler lock
```

This is a simplification. The detailed behavior is in the sections below: `tick()` advances `next_run_at` before dispatch (provisional advance), runs are claimed in an execution ledger before provider dispatch, and delivery is queued rather than inline. The lock is cross-process (`fcntl.flock` on Unix, `msvcrt.locking` on Windows); if it cannot be taken, `tick()` returns 0 immediately. Scripts and jobs are dispatched to worker pools rather than held under the tick lock, so a long job does not block other due jobs.

Execution isolation under systemd: each due job is handed to an external worker (`python -m cron.scheduler`) in a transient user scope (`systemd-run --user --scope`) so a gateway restart does not kill it. Hosts without a user session degrade by default (the job runs without cgroup isolation); `cron.require_restart_safe_scope: true` fails closed instead.

### Missed-occurrence contract

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [cron/missed_runs.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/missed_runs.py) · [cron/occurrences.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/occurrences.py)

The developer guide states recurring jobs are at-most-once per occurrence and every occurrence is accounted for: it runs (one execution row with its `scheduled_instant`) or its skip is logged with a reason. In the order the due scan applies them (`_evaluate_due_job`):

1. The pre-dispatch advance is provisional; the due scan stamps a `pending_slot` on the record so an advanced-but-never-claimed slot can be recovered.
2. A later scan that finds a `pending_slot` whose owner is provably gone restores it once (with a WARNING), so there is no replay of N slots.
3. A slot with a `completed` ledger row for that exact `scheduled_instant` is never run twice.
4. Late within grace fires late (`last_dispatch.kind = late`). Grace is half the period clamped to 120 s to 2 h.
5. Past grace, the backlog collapses and fires once (`catch_up`), or is skipped with a logged reason when catch-up is off. One-shots past their 120 s grace are retired with a diagnostic.
6. Paused, disabled and terminal jobs never fire. A recurring slot that came due while paused stays due on resume (one late/catch-up run or a logged skip).

User-level controls documented in the cron guide:

```yaml
cron:
  catch_up_missed: false   # default true; false skips past-grace recurring misses after planned downtime
```

Per job: `--catch-up on|off` and `--grace-minutes N`. Every passed-over slot is booked as a `missed xN` row in the ledger (visible in `hermes cron runs <job>`).

A firing run holds a `fire_claim` lease (300 s) refreshed by a heartbeat every 60 s; a confirmed loss cancels the in-flight run, but the outcome is decided against the store at completion.

## Providers: built-in ticker and Chronos

> Sources: [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [Chronos managed-cron contract](https://hermes-agent.nousresearch.com/docs/developer-guide/chronos-managed-cron-contract) · [cron/scheduler_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/scheduler_provider.py) · [plugins/cron_providers/chronos/verify.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/cron_providers/chronos/verify.py)

In gateway mode the cron trigger (when a due job fires) is chosen by a pluggable `CronScheduler` provider; the gateway calls `resolve_cron_scheduler()` and runs the provider's `start()` in a background thread alongside a separate housekeeping thread. `cron.provider` empty (default) selects the built-in `InProcessCronScheduler` (the 60 s `tick()` loop). A named provider is discovered from a plugin directory; if it is missing, fails to load or reports unavailable, the resolver falls back to the built-in with a warning, so cron is never left without a trigger. A provider controls only the trigger; execution and delivery stay in `run_job()` / `_deliver_result()`.

Documentation inconsistency about the provider directory: the developer guide says `plugins/cron_providers/<name>/` or `$HERMES_HOME/plugins/<name>/`; the CLI reference says `plugins/cron/<name>/`; the Chronos contract names `plugins/cron/chronos/verify.py`. The tree at this SHA contains `plugins/cron_providers/chronos/` (with `verify.py`, `_nas_client.py`, `plugin.yaml`), which is the path cited here.

Under multiplexing the docs state an external `cron.provider` does not support multiplexing and falls back to the built-in ticker with a warning. One gateway process ticks every served profile's store in sequence under a per-profile cron scope; a profile running its own gateway is skipped per tick.

### Chronos (managed cron for scale-to-zero)

> Sources: [Chronos managed-cron contract](https://hermes-agent.nousresearch.com/docs/developer-guide/chronos-managed-cron-contract) · [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals)

Hosted gateways can set `cron.provider: chronos`. Instead of a 60 s loop that keeps the process awake, the agent asks the Nous account service to arm one managed one-shot per job at its real next-fire time. At fire time the service calls back over an authenticated webhook; the gateway verifies, claims the job with a store compare-and-set, runs it and re-arms the next one-shot. The agent holds no scheduler credentials.

```text
create/update a cron job
  -> Chronos asks the service to arm a one-shot at next_run_at   (agent's Nous token)
  -> at fire time: POST {callback_url}/api/cron/fire             (short-lived, purpose-scoped JWT)
  -> gateway verifies token, claims job (CAS), returns 202, runs it in background, re-arms
```

| Key | Meaning |
|---|---|
| `cron.provider` | `chronos` to activate (empty = built-in ticker) |
| `cron.chronos.portal_url` | Service base URL (arming and the token issuer) |
| `cron.chronos.callback_url` | The gateway's own public base URL for inbound fires |
| `cron.chronos.expected_audience` | This agent's fire-token audience |
| `cron.chronos.nas_jwks_url` | Key set for verifying the fire token |

The contract documents: outbound `provision` and `cancel` calls plus a scheduler-to-service `relay` hop; the inbound verifier checks signature, audience, issuer, expiry and `purpose == "cron_fire"` (401 on failure); a gateway still starting makes the fire wait up to 7 s then answer 503 with `Retry-After: 60`; recurring jobs re-arm after each fire; reconciliation runs at start, on job mutation and after each fire; there is no periodic wake. A misconfigured provider or a deterministic `403 invalid_client` degrades to the built-in ticker. While `hermes pause` is engaged the fire webhook answers 503 so the scheduler redelivers after `hermes resume`. For hosted deployments a `cron.misfire_grace_minutes` sweep (the docs' example value is 10; 0 disables) claims and runs a job locally if the external fire hand-off failed.

## Run execution

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [cron/scheduler.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/scheduler.py)

Each job runs in a completely fresh agent session: no conversation history from earlier runs, the prompt must be self-contained, and cron cannot ask clarifying questions. Persistent memory (MEMORY.md and USER.md) loads as in any agent run. Cron-run sessions have the `cronjob` toolset disabled as a recursion guard. Model resolution at fire time is: per-job pin, then `cron.model` / `cron.model_provider`, then the main agent model. Before building agent machinery the scheduler validates the job (provider key resolves, skills are ready, delivery targets known, named MCP servers resolved); a failure books `blocked_config`, sends one alert and makes no model call. Disable with `cron.preflight: false`.

Skill-backed jobs load attached skills in order and inject their SKILL.md content before the task prompt. Provider recovery: credential-pool rotation applies to every job; fallback providers apply to unpinned jobs only.

Script-backed jobs: a `script` (a file under `$HERMES_HOME/scripts/`, never a command line) runs before the agent turn and its stdout is injected as context; a final stdout line `{"wakeAgent": false}` skips the agent for that tick. Timeouts documented in the user guide and environment reference:

| Setting | Default | Notes |
|---|---|---|
| Pre-run script timeout | 3600 s | `HERMES_CRON_SCRIPT_TIMEOUT` env, then `cron.script_timeout_seconds`, then default; bounds the script only |
| Agent inactivity timeout | 600 s | `HERMES_CRON_TIMEOUT`; idle-time based, `0` = unlimited |
| Post-run cleanup | 10 s | `cron.cleanup_timeout_seconds` |
| Media attachment send | 300 s | `HERMES_CRON_MEDIA_SEND_TIMEOUT` / `cron.media_send_timeout_seconds` |
| Standalone send | 60 s | `cron.standalone_send_timeout_seconds` |

The original guide gave the script timeout default as 120 seconds; the current docs state 3600 seconds. No-agent mode (`no_agent`) runs a script on a schedule and delivers its stdout verbatim with zero model involvement; empty stdout is a silent tick and a non-zero exit delivers an error alert. Jobs can chain with `context_from` (another job's latest output is prepended) or carry their own previous output with `continuity`. Event-triggered firing is available through a webhook route with `cron_job` set (see the webhook docs).

## Delivery

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [cron/delivery_queue.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/delivery_queue.py)

The agent's final response is delivered automatically to the job's `deliver:` target; the agent does not send messages itself. A bare platform name targets that platform's home channel; `platform:<target>` names a destination resolved at fire time; many platforms accept `platform:<chat_id>:<thread_id>`; comma-separated lists fan out.

| Target | Syntax (trimmed from the docs) |
|---|---|
| Origin chat / local file | `origin`, `local` (saved under `~/.hermes/cron/output/`) |
| Telegram | `telegram`, `telegram:<chat_id>`, `telegram:<chat_id>:<thread_id>` |
| Discord | `discord`, `discord:#channel`, `discord:<channel_id>[:<thread_id>]` |
| Slack | `slack`, `slack:#channel`, `slack:<channel_id>[:<thread_ts>]` |
| Others with explicit syntax | Matrix, Feishu, WhatsApp, Signal, SMS, Email, Weixin; other platforms accept `platform:<chat_id>` |
| Bot Chat | `bot-chat`, `bot-chat:<profile>` (machine-local pseudo-platform) |
| Fan-out | `all`, `telegram,discord`, `origin,all` |

An agent run with a delivery target hands content to the durable delivery queue and books `last_status = delivery_queued`; a gateway housekeeping tick drains it through live adapters, each row claimed at most once, and the outcome is written back (see `last_status`). Delivered output is secret-redacted. `cron.wrap_response` (default true) wraps deliveries with a header naming the job and a footer noting the agent cannot see the message:

```yaml
cron:
  wrap_response: false   # deliver the raw agent output
  delivery:
    notify: false        # default true: cron output is sent with the notification flag set
```

A response containing `[SILENT]` suppresses delivery (the output is still saved; failed runs always deliver). An agent can declare a failed run by putting `[CRON_FAILURE]` alone on the first line. Continuable jobs (reply into a delivered brief) are opt-in and default off (`cron.mirror_delivery`, per-job `attach_to_session`): thread-capable platforms seed each delivery's own thread, DM-only platforms mirror a labelled `[Cron delivery: <task name>]` user turn into the DM session, and broadcast `all` expansions are never made continuable. The developer guide's blanket statement that deliveries are not mirrored holds for the default configuration. `bot-chat` delivery injects a real inbound turn into the target profile's gateway; with no running authority the run is recorded `delivery_failed` and there is no local fallback.

## Controlling runs

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [CLI commands reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [cron/cancellation.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/cancellation.py)

```bash
hermes cron list | status | tick | doctor
hermes cron create "every 2h" "Check server status" [--skill NAME ...] [--deliver TARGET]
hermes cron edit <job_id_or_name> [flags]
hermes cron pause|resume|run|cancel|remove <job_id_or_name>
hermes cron runs [job-id] --limit 20      # execution ledger (alias: history)
hermes cron incidents [ack <id>]
```

In chat the same verbs are available as `/cron add|list|pause|resume|run|remove`, and the model uses `cronjob_manage` with actions including `create`, `list`, `update`, `pause`, `resume`, `run`, `remove` and `cancel`. Mutating verbs accept a job name (case-insensitive); ambiguous names are refused with the candidate IDs.

- **Cancel (stop a run in flight).** `hermes cron cancel` (also the API, the agent's `cancel` action and a Desktop Stop button) stops the job's current run without touching its schedule: script jobs get their process tree terminated, agent jobs are interrupted at the next checkpoint. The run books a `cancelled` row with the reason and does not count toward the failure streak. Removing a job cancels a live run. The docs call cancel a request, not a kill, and it does not roll back side effects.
- **Missed-run policy.** Per-job `--catch-up on|off`, per-job `--grace-minutes` and the global `cron.catch_up_missed` (see the missed-occurrence contract above). Resume keeps a past-due slot due.
- **Manual runs.** `hermes cron run` triggers the job on the next scheduler tick; a manual run is an extra fire that does not spend the `repeat` count and does not consume the scheduled slot. The tool's `action="run"` runs in the background and re-enters the conversation with the outcome; a job already running is refused. The earlier note that `run` executes "immediately" is not how the CLI docs describe it.
- **Global stop.** `hermes pause [--reason ...]` stops all scheduled cron fires (including managed-cron webhooks and catch-up sweeps) without killing runs already in flight; `hermes resume` lifts it.
- **CLI mode.** The earlier claim that cron fires during active CLI sessions is not supported by the current docs: agent jobs execute inside the gateway, and `hermes cron tick` with no gateway skips agent jobs (script-only `no_agent` jobs still run).
- **Paused creation.** `hermes cron create ... --paused [--paused-reason ...]` creates a job without scheduling it.

## Failure handling and health

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [Cron Troubleshooting](https://hermes-agent.nousresearch.com/docs/guides/cron-troubleshooting)

```yaml
cron:
  failure_nudge_threshold: 3        # default; 0 disables the review nudge
  failure_repeat_alert_hours: 6     # reminder cooldown for a repeating identical failure; 0 = alert every run
  retry_unreachable: true           # default; re-runs after 5, 15, 30 min when the model was unreachable
```

- Failure incidents: an identical repeating failure alerts once, then a reminder after the cooldown; `hermes cron incidents ack <id>` silences that signature.
- Transient network errors before any model call trigger automatic re-runs; a provider-reported closed usage window parks the job (`quota_hold_until`) instead of re-firing into it.
- `hermes cron doctor` is a read-only fleet check (failed runs and deliveries, overdue `next_run_at`, missing scripts or workdirs) that exits 1 while findings stand.
- A non-writable cron store (full disk, read-only mount, permissions) makes the scheduler skip runs it cannot record; `hermes cron status`, `hermes doctor` and monitoring metrics surface it.
- An interrupted execution is marked `unknown` only when its owner is provably gone or silent past a stale bound; unknown attempts are audit records and never auto-rerun. The docs state this is not an exactly-once side-effect guarantee.
- A gateway running pre-update code yields its tick when another process holds the runtime lock; `hermes cron status` then reports the stale-code condition.

## Related features

> Sources: [Session Heartbeats](https://hermes-agent.nousresearch.com/docs/user-guide/features/heartbeat) · [Recurring Loops](https://hermes-agent.nousresearch.com/docs/user-guide/features/loops) · [Persistent Goals](https://hermes-agent.nousresearch.com/docs/user-guide/features/goals) · [Automate Anything with Cron](https://hermes-agent.nousresearch.com/docs/guides/automate-with-cron) · [Script-Only Cron Jobs](https://hermes-agent.nousresearch.com/docs/guides/cron-script-only)

Three in-session features look like cron but are separate. `/heartbeat every <interval> <prompt>` re-enters the current session whenever it is idle (one per session, minimum 60 s, state kept in the session store, coalescing missed ticks). `/loop` re-runs a prompt on a fixed or self-paced cadence inside the current session. `/goal` is judge-driven, not timer-driven. The heartbeat page's rule of thumb: if the recurring prompt needs this conversation's context, use `/heartbeat`; if it is a self-contained job, use cron. Practical patterns are in the automation guide (change monitor, weekly report, repository watcher, multi-skill workflows) and the script-only guide (watchdogs with `no_agent`). Bot Mode's Routines are ordinary cron jobs named `[bot:<name>] <routine>`, visible in `hermes cron list`.

## Not covered here

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [Gateway Monitoring](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-monitoring)

The `cron.delivery` push settings in detail, Telegram cron-topic routing, `workdir` project-context behavior, the `all` routing intent edge cases, Slack in-channel continuable surfaces, the full per-field `cronjob_manage` schema, the execution-ledger and incident database layouts, the stale-code yield mechanics, and the cron OTLP metrics (see section 18 monitoring and the Gateway Monitoring page). Consult the Scheduled Tasks page and `cron/AGENTS.md` for those.

---

# 20. Plugin System
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [Built-in Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/built-in-plugins) · [Plugin Catalog](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugin-catalog) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py) · [hermes_cli/plugins_context.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_context.py) · [hermes_cli/plugins_manifest.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_manifest.py)

Everything below is derived from the upstream docs and source at SHA `9f23c43` (docs-only review; nothing here was executed). The old description of "three discovery sources" and "four plugin types" no longer matches the documented system, which now covers several loaders, a reviewed catalog, update tooling, a consent layer and an optional out-of-process host.

## Overview: what a plugin can do
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugins_context.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_context.py)

A general plugin is a directory with a `plugin.yaml` manifest and an `__init__.py` exposing `register(ctx)`. The `ctx` object (`PluginContext`, `hermes_cli/plugins_context.py`) is the only supported way to extend Hermes. The docs' capability table, condensed:

| Capability | `ctx` API (as documented) |
|---|---|
| Tools | `ctx.register_tool(name, toolset, schema, handler, check_fn=None, requires_env=None, is_async=False, description="", emoji="", override=False)` |
| Lifecycle hooks | `ctx.register_hook(hook_name, callback, *, hold_stream=False)` |
| Middleware | `ctx.register_middleware(kind, callback)` |
| Slash commands (CLI and gateway sessions) | `ctx.register_command(name, handler, description)` |
| `hermes <plugin> ...` subcommands | `ctx.register_cli_command(name, help, setup_fn, handler_fn)` |
| Run another tool with the parent agent's context | `ctx.dispatch_tool(name, args)` |
| Inject / post messages | `ctx.inject_message(...)`, `ctx.post_message(text)` returning a handle with `.edit(text)` |
| Bundled skills (namespaced `plugin:skill`) | `ctx.register_skill(name, path, description)` |
| Host-owned LLM call | `ctx.llm.complete(...)` / `ctx.llm.complete_structured(...)` |
| Capability-gated MCP call | `ctx.call_mcp(server, tool, arguments, timeout=30)` |
| Gateway platform | `ctx.register_platform(...)` |
| Backends | `register_image_gen_provider`, `register_video_gen_provider`, `register_web_search_provider`, `register_browser_provider`, `register_terminal_environment_provider`, `register_secret_source`, `register_tts_provider`, `register_transcription_provider`, `register_delegation_provider`, `register_computer_use_provider` |
| Single-select providers | `register_memory_provider`, `register_context_engine`; model providers use `register_provider(ProviderProfile(...))` instead of `ctx` |
| Approval routing | `ctx.register_approval_transport(name, present_fn)` |
| Other | `register_dashboard_auth_provider`, `register_login_backend`, `register_session_env`, `register_system_prompt_section`, `register_locale`, `register_redaction_patterns`, `register_auxiliary_task`, `register_platform_handler` (all present in `PluginContext` at this SHA; see the developer guide for semantics) |

Notes the docs call out:

- `register()` runs once at startup; if it raises, that plugin is disabled and Hermes continues.
- Slash-command handlers run in the gateway process that owns the session, even for the classic CLI. They should return text rather than `print()`, and read the working directory with `agent.runtime_cwd.resolve_agent_cwd()` rather than `os.getcwd()`.
- Tool handlers have the signature `def handler(args: dict, **kwargs) -> str` and always return a JSON string (see section 22).

## Plugin kinds, locations and discovery
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Built-in Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/built-in-plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugins_manifest.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_manifest.py) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py) · [plugins/memory/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/memory/__init__.py)

`plugin.yaml` may carry a `kind`. The valid set in source is `standalone`, `backend`, `exclusive`, `platform`, `model-provider` (an unknown value is warned about and treated as `standalone`; memory and model providers without a declared kind are auto-detected from `__init__.py`).

Discovery sources documented for general plugins (later sources override earlier ones on a name collision):

| Source | Path | Notes |
|---|---|---|
| Bundled | `<repo>/plugins/` | Ships with Hermes; opt-in (see below) |
| User | `~/.hermes/plugins/` (profile-scoped via `HERMES_HOME`) | Personal or installed plugins |
| Project | `./.hermes/plugins/` | See the project-plugin note below |
| pip | `hermes_agent.plugins` entry-point group | Capabilities for pip plugins are declared in the `hermes_agent.plugin_capabilities` group |
| Nix | `services.hermes-agent.extraPlugins` / `extraPythonPackages` | Declarative installs, symlinked with a `nix-managed-` prefix |

Sub-category directories route to specialised loaders:

| Directory | Holds | Loader |
|---|---|---|
| `plugins/` root | tools, hooks, slash/CLI commands, skills | `PluginManager` (`standalone` or `backend`) |
| `plugins/platforms/<name>/` | gateway channel adapters | `PluginManager` (`platform`) |
| `plugins/image_gen/`, `plugins/delegation/` | image-generation backends, delegation strategies | `PluginManager` (`backend`) |
| `plugins/memory/<name>/` | memory providers (single-select) | own loader, `plugins/memory/__init__.py` |
| `plugins/context_engine/<name>/` | context engines (single-select) | own loader |
| `plugins/model-providers/<name>/` | inference backend profiles (many registered, one chosen per request) | own loader, `providers/__init__.py`, lazily scanned |

The repository tree at this SHA also contains `plugins/web`, `plugins/browser`, `plugins/video_gen`, `plugins/computer_use`, `plugins/cron_providers`, `plugins/config_backends` and `plugins/dashboard_auth`; the matching authoring guides are listed under "Specialised plugin types" below. Do not rely on a fixed count of bundled plugins; list `plugins/` or run `hermes plugins list`.

Two ordering rules differ by loader: general plugins and model providers use "later wins" (a user plugin can replace a bundled one), while memory providers resolve the opposite way (bundled, user, project, then entry points; first seen wins) so that a dropped-in directory cannot silently redirect memory.

**Project-local plugins (docs conflict, flagged).** The Built-in Plugins page still lists `./.hermes/plugins/` as requiring `HERMES_ENABLE_PROJECT_PLUGINS=1`. The newer Plugins page states that the gateway never loads project-local plugins (every session, including `hermes`, the TUI, Desktop, messaging and cron, runs in the one gateway), so the variable has no effect on sessions and Hermes only prints a warning; `gateway/session_local.py` contains that warning. The memory loader still reads the variable for its own project scan. Treat the Plugins page as authoritative: install the plugin to `~/.hermes/plugins/` and enable it.

## Opt-in model: enabling and disabling
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Built-in Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/built-in-plugins) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py)

General plugins and user-installed backends are disabled by default. Discovery lists them (`hermes plugins`, `/plugins`) but nothing loads until the name is in `plugins.enabled`. A name in `plugins.disabled` always wins.

```yaml
plugins:
  enabled:
    - my-tool-plugin
  disabled:
    - noisy-plugin
  clone_timeout_seconds: 300      # Git clone/fetch deadline during install (clamped at 3600)
  hook_callback_timeout: 30       # wall-clock cap for in-process hook callbacks; 0 disables (clamped at 600)
  load_timeout_seconds: 10        # import + register() deadline per plugin; 0 disables (clamped at 600)
```

Three states, as documented: `enabled` (in `plugins.enabled`, not disabled), `disabled` (explicitly off), and `not enabled` (discovered, never opted in; the default for a newly installed or bundled plugin). Bundled and platform/backend plugins are an exception in the picker: they show as on when active without a list entry. Gateway event hooks under `~/.hermes/hooks/` are not plugins and are not gated by these keys (placing `HOOK.yaml` + `handler.py` there is the opt-in).

Provider plugins are selected by their own keys rather than `plugins.enabled`:

```yaml
memory:
  provider: "honcho"      # empty string = built-in memory only
context:
  engine: "compressor"    # default built-in compressor
```

The docs also describe a one-time migration: user plugins already installed when opt-in plugins arrived (config schema v21) were grandfathered into `plugins.enabled`; bundled standalone plugins were not.

## Managing plugins from the CLI
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands) · [Plugin Catalog](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugin-catalog) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py)

Subcommands documented on the Plugins page and CLI reference:

| Command | Purpose |
|---|---|
| `hermes plugins` | Composite interactive UI (general toggles plus provider pickers) |
| `list`, `search [term]`, `browse`, `info <name>` | Inspect installed plugins and the catalog |
| `install <name\|owner/repo\|git-url> [--enable\|--no-enable] [--force] [--ref SHA] [--allow-removed]` | Install; catalog names resolve to the reviewed pinned commit; explicit repos are flagged as custom (unreviewed) |
| `enable`, `disable`, `remove` (`rm`, `uninstall`) | Toggle or uninstall |
| `update <name> [--force]` | Pull or re-pin one plugin (local edits are autostashed) |
| `update [--yes]` | With no name: list every plugin with an update, then ask `Update all N? [Y/n]`; without a terminal nothing updates unless `--yes` |
| `check-updates` | Read-only check for outdated installs |
| `auto-update [on\|off]` | Toggle unattended updates (see below) |
| `capabilities [name]` | Declared versus granted capabilities |
| `adopt <name>`, `trust-update-url <name>` | Track a self-cloned directory; approve a changed `update_url` |
| `doctor [path-or-id] [--ci]`, `validate <dir> [--install-deps]` | Author-side checks (see below) |
| `pack install\|export\|show` | Plugin packs (`hermes-pack.yaml`), a declarative set of plugins pinned to exact SHAs |

In a chat session the equivalents are `/plugins`, `/plugins update` and `/plugins auto-update [on|off]`; Desktop and the web dashboard expose an Update all action and an Auto-update switch.

### Updates and auto-update (opt-in)
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py)

- Install provenance (source, revision, pin status) is recorded in the profile-local `plugins/.install-metadata.json`; it holds no config values, secrets or capability grants.
- `--ref` accepts only a full 40-character commit SHA (tags, branches and abbreviated SHAs are not accepted). A pinned plugin is never moved by `hermes plugins update`; re-pin with `install <source> --force --ref <new-commit>`.
- A scheduled gateway check runs when `plugins.auto_update_check_hours` is due (default 24; `0` disables). Its receipt is shown by `hermes pm status` and the desktop sync-status view.
- Auto-update is off by default. Setting `plugins.auto_apply: true` (or `hermes plugins auto-update on`) lets the gateway's scheduled check apply catalog re-pins and tracked Git pulls itself. An unattended update never accepts a widened pin or Git update (new tools, hooks, capabilities), new capabilities, or a `caution` scan verdict on a Git update; those wait for a manual update. Pinned, manual, drifted and pip-distribution rows are not auto-managed. The setting is per profile.
- `hermes update` can move catalog pins; installed catalog plugins stay on their old pin until updated.
- A manifest that adds or changes `update_url` is refused until `hermes plugins trust-update-url NAME`.

### Dependencies, scanning and isolation
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py)

- **Python dependencies.** Declared via `python_dependencies` in `plugin.yaml` (or a `pyproject.toml`). Install requests consent; enabling prepares the union of core, extras and all enabled plugins through PM (the package manager). A failed resolution preserves the previous selection, and a declined or non-interactive install can leave the plugin installed but disabled.
- **Install-time security scan.** Every install and update scans the plugin tree. Verdicts: `safe` (silent), `caution` (confirm, or `--force`), `dangerous` (blocked; `--force` does not override).
- **Process isolation (`plugins.isolation`).** The default is `in_process`. `plugins.isolation: host` runs user-installed third-party plugins in one separate Python process per profile (optional `plugins.host.launcher` argv prefix). Bundled plugins stay in-process. Surfaces that need in-process code (platform adapters via `register_platform`, approval transports, model-provider profiles that build their own SDK client, terminal backends, streaming/websocket dashboard endpoints, plugins that monkeypatch Hermes) fail to load in host mode with a reason. The setting can be pinned in the managed scope so profiles cannot turn it off.
- **Not a sandbox.** Capabilities and scans are consent and audit layers; plugins otherwise run as regular Python with the user's permissions.

## Plugin catalog and trust tiers
> Sources: [Plugin Catalog](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugin-catalog) · [Submitting to the Plugin Catalog](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/catalog-submission) · [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [plugin-catalog/README.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugin-catalog/README.md)

The catalog is a curated directory of one YAML file per plugin under `plugin-catalog/` in the hermes-agent repository, browsable at `/docs/plugins` on the docs site (shelves such as Memory, Desktop, Platforms, Web & Browser, Tools, Voice, Automation, Models). Do not quote an entry count from here; the v0.23.0 release notes say it grew from "about 600 to over 1,050" at release time, and the way to enumerate it is `hermes plugins search` (omit the term to list everything) or `hermes plugins browse`, or the `plugin-catalog/` directory itself.

Entry fields documented include `name`, `description`, `repo`, `sha` (exact 40-hex commit), optional `subdir`, `tier`, `category`, `maintainer`, `capabilities` (tools, hooks, middleware, env vars), `requires_hermes` (enforced at install and update), `platforms`, `version`, `image`, `screenshots`, `readme`, `known_issues` and `onboarding`.

Tiers (the `tier` field; only Nous maintainers set the first two):

| Tier | Meaning per the docs |
|---|---|
| `official` | Maintained by NousResearch |
| `verified` | Verified by the Nous team to come from the reputable company whose product it integrates |
| `community` | Everything else |

Trust model in brief: human-merged admission PRs only; mandatory exact SHA pins (branches, tags and short SHAs rejected); no self-updating code in listed plugins; SHA bumps are new reviewed PRs; declared capabilities must match what the plugin registers; the install scanner runs at admission (a catalog install at exactly the pinned SHA does not re-prompt for `caution`, `dangerous` still blocks); desktop plugins must stay inside the SDK surface; a `removed.yaml` list blocks installs and updates of pulled plugins. The docs state plainly that "cataloged is not audited". `hermes plugins install <name>` installs disabled; enabling is a separate step.

## Manifest (`plugin.yaml`) and compatibility contract
> Sources: [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [Public plugin API](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/public-api) · [hermes_cli/plugins_manifest.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_manifest.py) · [hermes_cli/data/plugin_api.json](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/data/plugin_api.json) · [hermes_cli/plugin_capabilities.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugin_capabilities.py)

Minimal manifest:

```yaml
name: calculator
version: 1.0.0
description: Math calculator
provides_tools: [calculate, unit_convert]
provides_hooks: [post_tool_call]
requires_env:                # gate loading; prompted at install
  - SOME_API_KEY
capabilities:                # privileged surfaces; consent flow
  - tools.override
```

`provides_tools` does not decide whether tools load once the plugin is enabled; it drives `hermes plugins validate` (declared-vs-registered check), catalog chips, and dashboard auth hints. The additive manifest v2 fields (all optional; a manifest without `manifest_version` is v1 forever) include `manifest_version`, `api_version`, `requires_plugins` (advisory; load order honours it), `python_dependencies`, `python_runtime`, `config_schema` (settings under `plugins.entries.<id>.settings`, also drives the Desktop settings form), `config_keys`, `license`, `homepage`, `tags`, `provides_locales`, `requires_auth` and `oauth`. Unknown fields are ignored with a warning.

Compatibility contract (documented): behaviour-based rather than a single API version; `PluginContext` methods are not removed or renamed; hook payloads grow as additive keyword fields (callbacks should accept `**kwargs`); a deprecation requires at least two minor releases of support. Replacing or patching core functions, methods or tables at runtime is not a supported extension point and is refused by catalog validation (`no core override`). The docs add that the Sep 2026 module decomposition removed the old internal import paths, so a plugin importing them fails with `ImportError`. The list of Hermes modules plugins may import is the generated Public plugin API page, backed by `hermes_cli/data/plugin_api.json`; `hermes plugins validate` warns on other Hermes imports.

### Capabilities and consent
> Sources: [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugin_capabilities.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugin_capabilities.py)

Declared capabilities are shown at `install`/`enable` time and granted once; grants are stored under `plugins.entries.<id>.granted_capabilities`. Known ids: `tools.override`, `llm.provider_override`, `llm.model_override`, `llm.agent_id_override`, `llm.profile_override`, `llm.task_override`, `llm.side_question`, `gateway.platform_actions`. Non-interactive installs and updates fail closed (nothing is granted). A plugin should probe with `ctx.has_capability(...)` and degrade. Legacy per-feature keys (for example `allow_tool_override`) still work but are deprecated. `ctx.llm` calls run against the user's active provider and model by default; provider/model/agent/profile overrides raise a trust error until granted (see the Plugin LLM Access page).

## Hooks, middleware and observers
> Sources: [Event Hooks](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks) · [Observer Hooks](https://hermes-agent.nousresearch.com/docs/developer-guide/observer-hooks) · [Middleware](https://hermes-agent.nousresearch.com/docs/developer-guide/middleware) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py) · [hermes_cli/middleware.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/middleware.py)

Hermes documents four hook systems: gateway event hooks (`HOOK.yaml` + `handler.py` under `~/.hermes/hooks/`, gateway only), plugin hooks (`ctx.register_hook`), shell hooks (`hooks:` in `config.yaml`) and outbound webhooks (`hooks.outbound:`). The authoritative list of plugin hook names is the `VALID_HOOKS` set in `hermes_cli/plugins.py`.

Count caveat: the Plugins page says "27 lifecycle events", but the set in source at this SHA lists noticeably more names (it includes later additions such as `post_gateway_admission`, `on_room_member_activity`, `transform_api_error_classification`, kanban worker observers and `gateway_platform_event`). Enumerate from source or the Event Hooks catalog instead of trusting a number.

Descriptive categories used by the docs:

| Category | Examples |
|---|---|
| Directive/control | `pre_tool_call` (may return `{"action": "block"\|"approve"\|"modify", ...}`), `pre_llm_call` (context injection or `{"action": "block"}`), `pre_verify`, `pre_gateway_dispatch` |
| Transform | `transform_tool_result`, `transform_terminal_output`, `transform_llm_output`, `pre_transcription` |
| Observer (return ignored) | `post_tool_call`, `post_llm_call`, `pre_api_request`, `post_api_request`, `api_request_error`, `pre_auxiliary_call`, `post_auxiliary_call`, stream hooks, session lifecycle hooks (`on_session_start/end/finalize/reset/delete`), `post_compaction`, `subagent_start/stop`, approval and human-input observers, `pre_command`, kanban task hooks |

`pre_llm_call` context is injected into the current turn's user message (not the cached system prompt). Observer hooks are fail-open and carry `telemetry_schema_version = "hermes.observer.v1"` plus correlation ids (`turn_id`, `api_request_id`, `task_id`, `session_id`). Callbacks are bounded by `plugins.hook_callback_timeout`; a timed-out `pre_tool_call` fails closed.

Middleware changes behaviour rather than observing it. The four kinds (`VALID_MIDDLEWARE` in `hermes_cli/middleware.py`):

| Kind | Effect |
|---|---|
| `tool_request` | Return `{"args": {...}}` to rewrite tool arguments before hooks, guardrails and approvals see them |
| `llm_request` | Return `{"request": {...}}` to rewrite provider kwargs |
| `tool_execution` | Wrap the call; invoke `next_call(payload)` exactly once (or skip it) |
| `llm_execution` | Same, around the provider call |

A raising middleware is logged and skipped; payloads carry `middleware_schema_version = "hermes.middleware.v1"`. Order for an LLM call: build kwargs, `llm_request`, `pre_api_request`, `llm_execution`, `post_api_request`/`api_request_error`. For a tool call: coerce args, `tool_request`, availability/guardrails/approval, `tool_execution`, `post_tool_call`, `transform_tool_result`.

## Specialised plugin types and their guides
> Sources: [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [Model Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin) · [hermes_cli/plugins_context.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_context.py)

| Want to add | Registration | Guide |
|---|---|---|
| LLM / inference backend | `register_provider(ProviderProfile(...))` in `plugins/model-providers/<name>/` | Model Provider Plugins |
| Gateway channel | `ctx.register_platform()` | Adding Platform Adapters |
| Memory backend | subclass `MemoryProvider`, `ctx.register_memory_provider()` | Memory Provider Plugins (section 21) |
| Context compressor | `ctx.register_context_engine()`, selected by `context.engine` | Context Engine Plugins |
| Image / video generation | `ctx.register_image_gen_provider()` / `register_video_gen_provider()` | Image / Video Generation Provider Plugins |
| Web search / extract | `ctx.register_web_search_provider()` | Web Search Provider Plugins |
| Cloud browser session | `ctx.register_browser_provider()` | Browser Provider Plugins |
| Terminal backend | `ctx.register_terminal_environment_provider()`, chosen with `terminal.backend` | Terminal Environment Plugins |
| Secret manager | `ctx.register_secret_source()` | Secret Source Plugins |
| Delegation strategy | `ctx.register_delegation_provider()`, chosen with `delegation.backend` | Delegation Provider Plugins |
| Computer-use driver | `ctx.register_computer_use_provider()`, chosen with `computer_use.backend` | Build a Hermes Plugin |
| Password-manager login backend | `ctx.register_login_backend()` | Browser Vault Plugins |
| TTS / STT | config-driven `tts.providers.<name>` / `stt.providers.<name>` (`type: command`), or Python `register_tts_provider` / `register_transcription_provider` | TTS setup |
| Native desktop UI | `@hermes/plugin-sdk` | Desktop Plugin SDK |
| External tools | `mcp_servers.<name>` in `config.yaml` | MCP |

Guide pages: [Model Provider Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin), [Context Engine Plugins](https://hermes-agent.nousresearch.com/docs/developer-guide/context-engine-plugin), [Image Generation](https://hermes-agent.nousresearch.com/docs/developer-guide/image-gen-provider-plugin), [Video Generation](https://hermes-agent.nousresearch.com/docs/developer-guide/video-gen-provider-plugin), [Web Search](https://hermes-agent.nousresearch.com/docs/developer-guide/web-search-provider-plugin), [Browser Provider](https://hermes-agent.nousresearch.com/docs/developer-guide/browser-provider-plugin), [Terminal Environment](https://hermes-agent.nousresearch.com/docs/developer-guide/terminal-environment-plugin), [Secret Source](https://hermes-agent.nousresearch.com/docs/developer-guide/secret-source-plugin), [Delegation Provider](https://hermes-agent.nousresearch.com/docs/developer-guide/delegation-provider-plugin), [Browser Vault](https://hermes-agent.nousresearch.com/docs/developer-guide/browser-vault-plugin), [Plugin LLM Access](https://hermes-agent.nousresearch.com/docs/developer-guide/plugin-llm-access), [Desktop Plugin SDK](https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk). Plugins that front a desktop application can also declare an `app:` / `requires:` block (presence, version, GPU) described on the Application declarations page; per that page, the loader integration is separate work and core does not read plugin YAML for it automatically.

## Bundled plugins and policy
> Sources: [Built-in Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/built-in-plugins) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [hermes_cli/plugins.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins.py)

The Built-in Plugins page lists, as shipped and opt-in: `disk-cleanup` (hooks plus slash command), `security-guidance` (hooks on `write_file`/`patch`), `teams_pipeline`, `google_meet`, three `image_gen/*` backends, `hermes-achievements` and `kanban/dashboard` dashboard tabs. Memory providers and context engines are excluded from bundled scanning and listed on the Memory Providers page. The v0.23.0 release notes describe several features (Camofox, `mcp-serve`, agent importers, Singularity, Copilot) moving out of core into plugins that install themselves for existing users; so `hermes mcp serve` is now `hermes mcp-serve` (provided by the `mcp-serve` plugin). The developer guide also states the policy that plugins integrating someone else's product ship as standalone repos rather than into the core tree.

Not covered here: per-plugin authoring detail, Desktop SDK slots, plugin RPC and session events, dashboard plugin APIs, `ctx.side_question`, outbound webhooks and platform actions. See the pages cited above.

---

# 21. Memory Provider Plugin Development
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py) · [agent/memory_manager.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_manager.py) · [plugins/memory/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/memory/__init__.py)

A memory provider gives Hermes persistent, cross-session knowledge in addition to the built-in `MEMORY.md` / `USER.md`, which stay active alongside it. Providers are single-select (`memory.provider`), managed with `hermes memory setup|status|off` or `hermes plugins` > Provider Plugins > Memory Provider. Contract source: `agent/memory_provider.py`; orchestration: `agent/memory_manager.py`.

Which providers ship in-tree has changed: at this SHA the Memory Providers page says Hermes ships two external provider plugins (RetainDB and ByteRover; `plugins/memory/` contains `byterover` and `retaindb`), while Honcho, Hindsight, Supermemory, Mem0, OpenViking and Holographic are installed from the plugin catalog (`hermes plugins install <name>`). The page says RetainDB and ByteRover leave core on October 15, 2026. The bundled set is closed to new providers.

## Installation layouts and discovery
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [plugins/memory/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/memory/__init__.py)

Four sources, precedence order (earlier wins on a name collision; this is the reverse of general plugins):

| Source | Location |
|---|---|
| Bundled | `plugins/memory/<name>/` |
| User | `$HERMES_HOME/plugins/<name>/` (per profile) |
| Project | `./.hermes/plugins/<name>/` (opt-in via `HERMES_ENABLE_PROJECT_PLUGINS=1`) |
| Package | `hermes_agent.memory_providers` entry point (name = value users put in `memory.provider`; target is a `register(ctx)`) |

Discovery only enumerates; nothing is imported until `memory.provider` names it. Do not pip-install a provider into Hermes' selected environment; on PM-managed installs ship a directory provider with declared Python dependencies (`pyproject.toml`, or `python_dependencies` / legacy `pip_dependencies`). A directory provider looks like:

```text
plugins/memory/my-provider/
├── __init__.py      # MemoryProvider subclass + register(ctx)
├── plugin.yaml      # name, version, description, hooks
├── config_schema.py # optional: Desktop config panel
├── cli.py           # optional: register_cli(subparser)
└── README.md
```

Registration:

```python
def register(ctx) -> None:
    ctx.register_memory_provider(MyMemoryProvider())   # new instance per load
```

A provider may also call `ctx.register_skill(...)`; such skills are qualified by the entry-point name (`my-provider:maintenance`) and load only while that provider is active.

## MemoryProvider ABC
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py)

In source, exactly four members are `@abstractmethod`: the `name` property, `is_available()`, `initialize(session_id, **kwargs)` and `get_tool_schemas()`. Everything else has a default. (The developer guide's tables also mark `get_config_schema()` and `save_config()` as "yes" for setup-wizard use; in source they default to `[]` and a no-op, and `save_config` must be overridden unless every field is env-var only.)

| Method | When called | Notes |
|---|---|---|
| `name` (property) | always | the id used in `memory.provider` |
| `is_available()` | before activation | no network calls |
| `initialize(session_id, **kwargs)` | agent startup | accept `**kwargs`; `hermes_home` always present |
| `get_tool_schemas()` / `handle_tool_call(tool_name, args, **kwargs)` | tool injection / dispatch | return JSON-style strings |
| `system_prompt_block()` | prompt assembly | static provider info |
| `prefetch(query, *, session_id="")` / `queue_prefetch(...)` | before each API call / after each turn | recall; pre-warm |
| `sync_turn(user, assistant, *, session_id="", messages=None)` | after each completed turn | must be non-blocking |
| `on_turn_start(turn_number, message, **kwargs)` | start of turn | receives `model`, `author_id`, `author_name`, `author_is_bot` |
| `on_session_end(messages)` | real session boundary only | final extraction/flush |
| `on_session_switch(new_session_id, *, parent_session_id="", reset=False, rewound=False, **kwargs)` | session id reassigned mid-process (`/resume`, `/branch`, `/new`, compression) | rebind per-session state |
| `on_pre_compress(messages)` | before compression | return text is fed into the summary prompt |
| `on_delegation(task, result, *, child_session_id="", **kwargs)` | parent side of a finished delegation | |
| `on_memory_write(action, target, content, metadata=None)` | built-in memory add/replace/remove | `metadata["previous_content"]` carries the replaced/removed entry; absent on older Hermes |
| `get_config_schema()` / `save_config(values, hermes_home)` | `hermes memory setup` | schema fields: `key`, `description`, `secret`, `required`, `default`, `choices`, `type`, `env_var`, `url`, ... |
| `post_setup(hermes_home, config)` (optional) / `get_status_config()` (documented) | after setup / `hermes memory status` | not declared on the ABC at this SHA: `hermes_cli/memory_setup.py` calls `post_setup` only if the provider defines it |
| `read_config_fields` / `write_config_fields` | Desktop panel when `config_schema.py` declares `storage=STORAGE_PROVIDER` | |
| `backup_paths()` | `hermes backup`/`import` | paths outside `HERMES_HOME`; must work without `initialize()` |
| `on_profile_created(profile_name, hermes_home)` / `on_profile_renamed(old, new, hermes_home)` | profile lifecycle | run on an uninitialized instance with that profile's home and secrets bound; do network work on a background thread |
| `shutdown()` | process exit | |

Initialization kwargs documented: `hermes_home`, `platform` (`cli`, `gui`, `acp`, `telegram`, ...), `session_title`, `session_title_source`, `cwd`, `gateway_session_key`, `user_id` / `user_id_alt` / `user_name` / `chat_id`, `agent_identity`, `resumed`, `message_count`, `agent_workspace` and `agent_context` (`primary`, `cron` or `subagent`; skip automatic writes for non-primary values). Read optional keys with `kwargs.get(...)` because older Hermes omits newer ones.

Secrets inside providers should be read through `agent.secret_scope.get_secret`, not raw environment lookups. The Memory Providers page also states that everything handed to a provider (turns, tool output, recall queries, mirrored writes) passes through a secret scrub first.

## Setup UX, config schema and storage
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py)

A standalone (catalog) provider keeps every setup surface through files in its own directory: `get_config_schema()` / `save_config()` / `post_setup()` for `hermes memory setup`, `config_schema.py` for the Desktop panel, `cli.py` for subcommands, and `pyproject.toml` for dependencies. Fields with `secret: True` and an `env_var` are written to `.env`; non-secret values go to `save_config()`. Keep the schema minimal (only what the user must configure) and document optional settings in a provider config file under `$HERMES_HOME`.

`config_schema.py` defaults to flat JSON storage at `$HERMES_HOME/<name>/config.json`. A provider with its own layout declares `storage=STORAGE_PROVIDER` and implements `read_config_fields` / `write_config_fields` (a save returns 501 if the writer is missing, 503 if the provider fails to load; raising `ValueError` refuses the save as a 400).

A provider's name, `memory.<name>` config section, data directory and tool names are the contract with existing users. When a bundled provider moves to the catalog, `hermes update` installs the catalog plugin for every profile whose `memory.provider` names it (and once at agent start when `security.allow_lazy_installs` is on).

## Threading contract
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py)

`sync_turn()` must not block. Run backend work in a daemon thread spawned with `agent.memory_provider.spawn_context_thread(target, name=...)` (never a bare `threading.Thread`): profile isolation (active `HERMES_HOME` override and the per-turn secret scope) lives in `contextvars`, and a plain thread starts with an empty context, so under multiplexed profiles it would write into the default profile's store and `get_secret()` would fail closed. The source module also exposes `ctx_bound(fn)` for executors. The same rule applies to prefetch and writer threads. Small JSON sidecars use `utils.read_json_or_empty` / `utils.atomic_json_write`; edits to `config.yaml` go through `hermes_cli.config.save_config(..., merge_existing=True)`.

`messages` passed to `sync_turn` is optional OpenAI-style context (user/assistant text, assistant tool calls, tool results). Providers that do not declare the parameter keep working with the legacy signature. Cloud providers should document what parts of it leave the device.

```python
from agent.memory_provider import spawn_context_thread

def sync_turn(self, user_content, assistant_content, *, session_id="", messages=None):
    def _sync():
        try:
            self._api.ingest(user_content, assistant_content, session_id=session_id)
        except Exception as e:
            logger.warning("Sync failed: %s", e)
    if self._sync_thread and self._sync_thread.is_alive():
        self._sync_thread.join(timeout=5.0)
    self._sync_thread = spawn_context_thread(_sync, name="myprovider-sync")
    self._sync_thread.start()
```

## Profile isolation (critical)
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_manager.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_manager.py)

All storage paths must derive from the `hermes_home` kwarg of `initialize()`, never a hard-coded `~/.hermes`:

```python
# correct: profile-scoped, bound on the instance
def initialize(self, session_id, **kwargs):
    self._data_dir = Path(kwargs["hermes_home"]) / "my-provider"

# wrong: shared across every profile
data_dir = Path("~/.hermes/my-provider").expanduser()
```

On a multiplexed gateway several profiles run turns concurrently, each with its own `MemoryManager` and provider instance. The docs require: return a new instance per load (no module-level singleton; the manager refuses an instance another profile already bound and logs `memory_provider_instance_shared`); bind every path, client and tenant from `initialize()`; keep nothing profile-specific at module or class level. The manager runs tool calls and hooks only from the profile an instance was initialized for, and refuses callers that name no profile (`memory_caller_profile_unresolved`). `handle_tool_call` also receives `hermes_home`, `agent_identity` and `caller_gsk`; assert on them but do not rebind from them. Hermes cannot verify a provider's internals.

## Single provider rule
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers) · [agent/memory_manager.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_manager.py)

Only one external provider can be active. The `MemoryManager` rejects a second registration with a warning ("Only one external memory provider is allowed", in `agent/memory_manager.py`), to avoid tool-schema bloat and conflicting backends. Values `""`, `default`, `builtin`, `built-in`, `none` for `memory.provider` mean built-in only (`CORE_MEMORY_PROVIDER_SENTINELS` in `agent/memory_provider.py`). Forks that run without a provider (delegated subagents and similar) get no prefetch or turn sync; `initialize()`, `on_session_switch()` and `on_pre_compress()` still run when the checkpoint gate is armed.

## Adding CLI commands
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [plugins/memory/__init__.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/plugins/memory/__init__.py)

Add `cli.py` to the provider directory with `register_cli(subparser)` that builds an argparse tree; it is discovered at startup (`discover_plugin_cli_commands()`) and appears as `hermes <provider-name> <subcommand>`. Commands only appear when the provider is the active `memory.provider`. Package (entry-point) providers get this too, provided the entry point targets a package so `cli.py` and `config_schema.py` sit next to `__init__.py`.

```python
# plugins/memory/my-provider/cli.py
def my_command(args):
    sub = getattr(args, "my_command", None)
    print("Provider is active and connected." if sub == "status" else "Usage: hermes my-provider <status|config>")

def register_cli(subparser) -> None:
    subs = subparser.add_subparsers(dest="my_command")
    subs.add_parser("status", help="Show provider status")
    subs.add_parser("config", help="Show provider config")
    subparser.set_defaults(func=my_command)
```

## Pre-compression checkpoints and oversized recall
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_provider.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_provider.py)

- **Checkpoint API v2 (opt-in).** `on_pre_compress()` is best-effort by default. A provider that archives transcript evidence can set `pre_compress_checkpoint_api_version = 2` and raise on failure. With `compression.checkpoint_required: true` (default `false`) compression fails closed (`BLOCKED_MISSING_PREREQUISITE`) unless an active provider completed its checkpoint. The docs state that none of the bundled providers advertise v2, so enabling the gate without a third-party archiving provider blocks every compression attempt. Checkpoints must be idempotent (key writes by content). The gate also suppresses native/micro compaction paths; see the page for the full list.
- **Oversized prefetch.** External `prefetch()` results are returned in full by default; `memory.prefetch_spill_enabled: true` (default `false`) spills results above `hooks.output_spill.max_chars` to a private file with a head/tail preview.

## Packaging, survival and testing
> Sources: [Memory Provider Plugins (developer guide)](https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin) · [agent/memory_manager.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/memory_manager.py)

Wrapper-style providers that keep a sidecar venv should propagate `HERMES_HOME`; ordinary Hermes updates do not delete `$HERMES_HOME/plugins/**`, whereas `hermes plugins remove`, `hermes uninstall` and profile deletion do. For tests the docs point to `tests/agent/test_memory_provider.py` and adjacent files and show the manager flow:

```python
from agent.memory_manager import MemoryManager
mgr = MemoryManager()
mgr.add_provider(my_provider)
mgr.initialize_all(session_id="test-1", platform="cli")
mgr.sync_all("user msg", "assistant msg")
mgr.on_session_end([])
mgr.shutdown_all()
```

Not covered: per-provider behaviour (see the Memory Providers page), dashboard readiness checks, and the Honcho identity-mapping details.

---

# 22. Adding Custom Tools
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

Decision order (from the docs): first ask whether the capability should be a skill (instructions plus shell and existing tools). If it needs API keys, custom processing, binary data or streaming, it is a tool. Then choose the route:

| Goal | Route | Page |
|---|---|---|
| Personal, team or project tool; no core change | Plugin: `ctx.register_tool(...)` | Build a Hermes Plugin |
| Ship a new built-in tool in the repository | Core: `tools/your_tool.py` + `toolsets.py` | Adding Tools |

The Adding Tools page is explicit that it covers built-in core tools only and that plugins are the default for custom tools.

## Route A: a plugin tool
> Sources: [Build a Hermes Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/) · [Plugins](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins) · [hermes_cli/plugins_context.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/plugins_context.py) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

Inside a plugin's `register(ctx)`:

```python
def register(ctx):
    ctx.register_tool(name="calculate", toolset="calculator",
                      schema=schemas.CALCULATE, handler=tools.calculate)
```

Handler rules from the developer guide: signature `def handler(args: dict, **kwargs) -> str`; always return a JSON string for success and error alike; never raise; Hermes forwards only the context keywords the signature names (`task_id`, `session_id`, `conversation_id`, `user_task`, `parent_agent`, ...), so `**kwargs` opts into the full set. Key per-conversation state on `conversation_id` (stable for the conversation) rather than `session_id` (can change when compression rotates the transcript). A registration that would shadow a built-in tool from a different toolset is rejected unless `override=True`, which additionally requires the `tools.override` capability grant (or legacy `plugins.entries.<id>.allow_tool_override: true`). Declare every tool in `provides_tools` so `hermes plugins validate` can check it, and test with `hermes plugins doctor . --ci`. Enable with `hermes plugins enable <name>`; tools appear alongside built-ins (see section 20).

## Route B: a built-in core tool (two files)
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py)

1. `tools/your_tool.py`: handler, schema, check function and a top-level `registry.register()` call.
2. `toolsets.py`: add the tool name to `_HERMES_CORE_TOOLS` (all platforms) or to a specific toolset entry.

The old "add a discovery import" step is gone: `discover_builtin_tools()` in `tools/registry.py` AST-scans `tools/*.py` for top-level `registry.register()` calls and imports those modules, so helper modules are not imported. After core discovery, MCP tools (`tools.mcp_tool_discovery.discover_mcp_tools()`) and plugin tools are discovered.

```python
# tools/weather_tool.py
import json, os
from tools.registry import registry

def check_weather_requirements() -> bool:
    return bool(os.getenv("WEATHER_API_KEY"))

def weather_tool(location: str, units: str = "metric") -> str:
    if not os.getenv("WEATHER_API_KEY"):
        return json.dumps({"error": "WEATHER_API_KEY not configured"})
    try:
        return json.dumps({"location": location, "temp": 22, "units": units})
    except Exception as e:
        return json.dumps({"error": str(e)})

WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather for a location.",
    "parameters": {"type": "object",
        "properties": {"location": {"type": "string", "description": "City name or coordinates"}},
        "required": ["location"]},
}

registry.register(
    name="weather", toolset="weather", schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool(location=args.get("location", ""),
                                            units=args.get("units", "metric")),
    check_fn=check_weather_requirements, requires_env=["WEATHER_API_KEY"],
)
```

`registry.register()` parameters (Tools Runtime page): `name`, `toolset`, `schema`, `handler`, optional `check_fn`, `requires_env`, `is_async`, `description`, `emoji`. `schema["description"]` is the authoritative model-facing text; the separate `description=` only fills registry metadata.

## Critical rules
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

- Handlers must return a JSON string (`json.dumps`), never a raw dict.
- Errors are returned as `{"error": "message"}`, not raised. As a safety net `registry.dispatch()` and `handle_function_call()` wrap exceptions into error JSON.
- `check_fn` runs when tool definitions are built; `False` or an exception silently excludes the tool. Results are cached per call, and the same function shared by several tools runs once.
- The handler receives `(args: dict, **kwargs)`.
- Dynamic schema patching: after filtering, schemas such as `execute_code`, `browser_cdp` and `delegate_task` are adjusted to reference only tools that passed filtering.

## Async handlers, task_id and agent-loop tools
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py) · [agent/tool_executor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/tool_executor.py)

Mark coroutine handlers with `is_async=True`; the registry bridges through `_run_async()` (persistent loop on the CLI path, a disposable thread with `asyncio.run()` when a loop is already running, per-thread loops for parallel tool workers). Never call `asyncio.run()` yourself.

Per-session tools receive `task_id` through `**kwargs`:

```python
def _handle_weather(args, **kw):
    return weather_tool(args.get("location", ""), task_id=kw.get("task_id"))
```

Four tools need agent state and are intercepted by the agent loop before registry dispatch: `todo_list`, `memory`, `session_search`, `delegate_task` (`agent/tool_executor.py`, called from `agent/conversation_loop.py`). Their schemas remain in the registry; a direct `dispatch()` returns a stub error. Dispatch flow per the Tools Runtime page:

```
tool_call -> agent loop -> model_tools.handle_function_call
          -> [agent-loop tool?] handled directly
          -> pre_tool_call hook -> registry.dispatch(name, args, **kwargs)
          -> async bridge if needed -> result string -> post_tool_call hook
```

## Setup-wizard integration (OPTIONAL_ENV_VARS)
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [hermes_cli/config_defaults.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config_defaults.py)

If the tool needs an API key, register it in `OPTIONAL_ENV_VARS`. The Adding Tools page still shows `hermes_cli/config.py`, but at this SHA the dictionary is defined in `hermes_cli/config_defaults.py` (`hermes_cli/config.py` imports it from there), so edit the defining module:

```python
OPTIONAL_ENV_VARS = {
    "WEATHER_API_KEY": {
        "description": "Weather API key for weather lookup",
        "prompt": "Weather API key",
        "url": "https://weatherapi.com/",
        "tools": ["weather"],
        "password": True,
    },
}
```

## Toolsets and enumeration
> Sources: [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

Toolsets are named bundles resolved from explicit enabled/disabled lists, platform presets (`hermes-cli`, `hermes-telegram`, `hermes-acp`, ...), dynamic MCP toolsets and special sets; `model_tools.get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode)` resolves them and then applies registry `check_fn` filtering. Legacy `*_tools` toolset names map through `_LEGACY_TOOLSET_MAP`. Do not count tools from this page; use `hermes tools --summary` and the Built-in Tools / Toolsets reference pages.

## Checklist
> Sources: [Adding Tools](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-tools) · [toolsets.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)

- Decided plugin versus core (plugins by default).
- Handler returns JSON strings; errors as `{"error": ...}`.
- Schema description written for the model (it decides when the tool is called).
- Core route: registered in `tools/*.py`, added to `toolsets.py`; optional `OPTIONAL_ENV_VARS` entry; optional `toolset_distributions.py` entry for batch processing.
- Smoke test: `hermes chat -q "Use the weather tool for London"`.

---

# 23. Programmatic Integration (ACP / TUI / API Server)
> Sources: [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [ACP Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/acp-internals) · [ACP Host Integration](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp) · [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [TUI](https://hermes-agent.nousresearch.com/docs/user-guide/tui) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Using Hermes as a Python Library](https://hermes-agent.nousresearch.com/docs/guides/python-library) · [acp_adapter/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/acp_adapter/server.py) · [tui_gateway/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/server.py) · [gateway/platforms/api_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/api_server.py) · [hermes_cli/gateway_runtime_attach.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/gateway_runtime_attach.py)

Hermes documents three protocols for driving the agent from external programs, plus an in-process Python embed. Since v0.23.0 (per the release notes and the gateway docs) the first two of those, and the classic CLI, Desktop and messaging, all attach to a gateway that owns each conversation, rather than each surface running its own agent. The gateway is the session authority for each profile it serves; by default one host gateway process multiplexes the default profile and any live named profiles (Sections 5 and 18), and each profile keeps its own sessions and `state.db`. This section is a docs-level map; none of it was executed.

## Protocol comparison
> Sources: [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [ACP Host Integration](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp) · [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [TUI](https://hermes-agent.nousresearch.com/docs/user-guide/tui) · [acp_adapter/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/acp_adapter/server.py) · [tui_gateway/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/server.py) · [gateway/platforms/api_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/api_server.py)

| Protocol | Transport | Best for | Defined by |
|---|---|---|---|
| ACP | JSON-RPC over stdio (`hermes acp`) | IDE/editor hosts that speak the Agent Client Protocol (VS Code, Zed, JetBrains, other ACP hosts) | `acp_adapter/` |
| TUI gateway (`hermes-gateway-v1`) | JSON-RPC over WebSocket (`/api/ws`, one-use ticket) or stdio | Custom hosts needing sessions, slash commands, approvals, streaming events (used by the Ink TUI and Desktop) | `tui_gateway/server.py`, `entry.py` (stdio), `ws.py` (WebSocket) |
| API server | HTTP + Server-Sent Events | OpenAI-compatible frontends and language-agnostic clients | `gateway/platforms/api_server.py` |
| Python embed | in-process import of `AIAgent` | scripts and services that want no subprocess | `run_agent.py` |

The Programmatic Integration page states that all drive the same `AIAgent` core and differ in wire format and exposed features. There is no `--mode rpc` flag.

## How each surface attaches to the gateway (v0.23.0)
> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [Gateway local sessions](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-local-sessions) · [Classic gateway client](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-classic-cli) · [Gateway command parity](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-command-parity) · [TUI](https://hermes-agent.nousresearch.com/docs/user-guide/tui) · [Desktop](https://hermes-agent.nousresearch.com/docs/user-guide/desktop) · [hermes_cli/gateway_runtime_attach.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/gateway_runtime_attach.py) · [gateway/session_local.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/session_local.py)

```
  classic CLI   Ink TUI   Desktop/Webapp   ACP editor   API clients   bots/cron
        \          |           |               |             |           /
         +---------+-----------+---------------+-------------+----------+
                                   |
                     gateway (session authority per served profile)
                owns sessions, queue, approvals, storage
                                   |
                               AIAgent turns
```

What the docs state:

- **Release notes (v0.23.0).** "One gateway owns every conversation": the CLI, TUI, Desktop, editors (ACP), messaging bots and cron previously could each run their own agent against the same chat; now each machine runs one gateway that owns every session and each surface attaches as a client.
- **Classic CLI.** `hermes chat`, `hermes --cli`, `-q` and `-z` use the gateway authority; the client prints a persisted `Session:` ID and detaches on `/quit`, `/exit` or EOF. An explicit `HERMES_TUI_GATEWAY_URL` is remote-only. Per-command routing is in the generated Gateway command parity table (some commands run in the client, some go to the owner as `slash.exec`, some are refused on the shared gateway).
- **TUI.** By default it attaches to the profile's shared gateway over a local WebSocket `/api/ws` speaking `hermes-gateway-v1` with a single-use ticket; if none is running it starts one (a gateway started this way stops itself after idling). It reconnects with backoff and reattaches to the same session. `HERMES_TUI_GATEWAY_URL` is described as a dashboard-internal wiring detail, not a general remote-attach knob; the API server does not serve `/api/ws`.
- **Other local clients.** `hermes gateway attach --json [--start] [--recover] [--timeout S]` discovers (or ensures) the owner and prints one JSON line with `url`, `protocols`, `protocol`, `ticket` and `endpoint`. Dial the URL from loopback with those subprotocols and no `Origin` header, then call `runtime.describe` and `session.create`. `hermes gateway ensure` is the sibling "discover or start" verb.
- **Desktop.** The app runs `hermes gateway ensure` for a local profile and dials `/api/ws` with one-use tickets; the gateway outlives the app. One gateway per host serves every profile unless a profile sets `gateway.standalone: true`. Settings > Gateways registers many gateways (local, SSH, URL, cloud) and keeps each connected. With `webapp.enabled` (off by default) the gateway also serves the Desktop workspace to a browser tab; `hermes webapp` opens it.
- **ACP.** `hermes acp` runs as a viewer of the gateway daemon and never starts the `config.yaml` MCP servers itself (the old `HERMES_ACP_SKIP_CONFIGURED_MCP` variable has no effect). Editor `mcpServers` from `session/new` attach to the canonical session on gateways advertising `acp-session-mcp-v1`.
- **API server.** It is a gateway platform adapter; requests use the gateway's durable admission queue (the docs describe a "canonical" mode with a session authority and a "legacy" mode without).
- **Multiple viewers.** Within one authenticated gateway, resuming or activating a live session attaches another event subscriber; streaming and terminal events reach all attached clients, and disconnecting one does not end the session. Closing every viewer leaves an execution and its pending approval alive for a later viewer. The docs state this does not allow independent gateway processes to write the same session.

## ACP (Agent Client Protocol)
> Sources: [ACP Host Integration](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp) · [ACP Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/acp-internals) · [acp_adapter/entry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/acp_adapter/entry.py) · [acp_adapter/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/acp_adapter/server.py) · [acp_adapter/gateway_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/acp_adapter/gateway_server.py)

```bash
hermes acp                  # serve ACP on stdio (also: hermes-acp, python -m acp_adapter)
hermes acp --version
hermes acp --check          # verify ACP dependencies and adapter imports
hermes acp --setup          # interactive provider/model setup (terminal auth method)
hermes acp --setup-browser  # optional: install pinned Chromium for browser tools
```

Install the optional extra through the package manager (the docs show `pm.sync_venv(['acp'], explicit=True)` from the install checkout). The entry point's documented flags are `--version`, `--check`, `--setup` and `--setup-browser`; there is no `--bootstrap`.

Behaviour documented:

- Stdout is reserved for ACP JSON-RPC; logs go to stderr. Boot: `acp_adapter.entry.main()` parses `--version/--check/--setup` first, loads `~/.hermes/.env`, constructs `HermesACPAgent`, then `acp.run_agent(...)`.
- Components: `HermesACPAgent` (`server.py`: initialize/authenticate, new/load/resume/fork/list/cancel, prompt, per-session model switching), `SessionManager` (`session.py`), event bridge (`events.py`, AIAgent callbacks to `session_update` via `asyncio.run_coroutine_threadsafe`), permission bridge (`permissions.py`), tool rendering (`tools.py`), auth (`auth.py`). `gateway_server.py` projects terminal outcomes from the gateway's structured result: cancelled admission returns `stopReason: cancelled`, a failed one raises an ACP request error.
- Toolset: the curated `hermes-acp` set (file, terminal, web/browser, memory/todo/session search, skills, execute_code/delegate_task, vision; no messaging delivery or cron management), resolved like the gateway for platform `acp`. Override with `platform_toolsets.acp`; `agent.disabled_toolsets` removes toolsets from every ACP session; `hermes tools` has no ACP entry.
- Auth: no ACP-specific store; it reuses Hermes' runtime resolver and always advertises a terminal setup method (`hermes-setup`, args `--setup`).
- Sessions persist to the shared session database and can be listed, loaded, resumed or forked after a restart; a new session without a prompt stays in memory. The editor cwd is bound to the session so file and terminal tools run in the workspace.
- Approvals map to ACP permission options `allow_once`, `allow_session`, `allow_always` and reject; timeouts and bridge failures deny. The wait is `approvals.timeout` (default 300 s). A host may answer programmatically, which is unattended execution.
- Limitations noted: non-text prompt blocks are ignored for request text extraction; editor UX varies by client.

## TUI gateway JSON-RPC
> Sources: [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [TUI](https://hermes-agent.nousresearch.com/docs/user-guide/tui) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [tui_gateway/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/server.py) · [tui_gateway/entry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/entry.py) · [tui_gateway/ws.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/ws.py)

`tui_gateway/server.py` is the method and event catalog behind the Ink TUI (`hermes --tui`), Desktop and the dashboard PTY bridge. External hosts can speak the same frames over stdio (`tui_gateway/entry.py`) or WebSocket (`tui_gateway/ws.py`). A Node client, `@hermes/shared/gateway-client`, lives in `apps/shared`; it takes a `connect(start, recover)` function (for example one that shells out to `hermes gateway attach --json`), reads `runtime.describe` before `session.create`, reconnects with backoff and emits typed events.

Selected methods named on the Programmatic Integration page (treat as a sample, not a closed list; enumerate with `commands.catalog` / the contracts under `tui_gateway/contracts`):

```
prompt.submit  prompt.background  session.steer   session.create  session.list
session.active_list  session.activate  session.close  session.interrupt
session.history  session.compress  session.branch  session.title  session.usage
session.status  clarify.lock  config.set / config.get  commands.catalog
client.capabilities  gateway.capabilities  ping  command.resolve  command.dispatch
cli.exec  reload.mcp  reload.env  process.stop  delegation.status
subagent.interrupt  subagent.steer  spawn_tree.save / list / load
terminal.resize  clipboard.paste  image.attach
```

Plugins add `plugin.<plugin_id>.<name>` methods via `ctx.register_rpc` and session events `plugin.<plugin_id>.<event>` via `ctx.emit_to_session`.

Notable protocol details:

- **Approvals and questions are server-to-client requests, not events.** The gateway sends JSON-RPC requests (`approval`, `clarify`, `sudo`, `secret`, `vault.code`, `vault.unlock_prompt`, `connection`, `terminal.read`, `window.read`, `preview.act`, `tour`) and the client answers with a normal JSON-RPC response carrying the same id. The Programmatic Integration page's method catalog does not list separate `*.respond` methods for these. Respond with `-32601` for methods you do not implement.
- **Breaking for WebSocket integrations:** after `gateway.ready`, a client must call `client.capabilities` with `{"server_requests": true}` once per connection, otherwise the gateway fails every server-to-client request for that client immediately.
- **Events streamed back (documented):** `message.delta`, `message.complete`, `tool.start`, `tool.generating`, `tool.complete`, `gateway.ready`, `request.cancel`, plus session lifecycle and error events. `request.cancel {id, method, reason}` withdraws a question; `session.resume`/`session.activate` results carry `open_requests` and `inflight` so a reconnecting client can re-render and answer.
- **`session.create` model overrides.** A `model`/`provider` pair the provider cannot serve is refused with JSON-RPC `-32602`, with `error.data.suggestions`.
- **Rewind on `prompt.submit`.** Truncation parameters (`truncate_before_user_ordinal`, `truncate_before_row_id`) require `confirm_truncate` (plus `confirm_empty_truncate` / `confirm_deep_truncate` in the stated cases); refusal codes include `4004`, `4018`, `4029`, `4030`, `4009` (busy); a successful rewind returns `survivor_user_row_ids`.
- **Fresh local sessions through the gateway:** `session.create` with a stable `request_id`, `session.resume` to attach viewers, `prompt.submit` with a `submission_id`, and `approval.respond({session_id, execution_generation, prompt_id, choice})` on the canonical gateway surface (Gateway local sessions page).
- **Pi-style mapping** (documented): `steer` is `session.steer`, `abort` is `session.interrupt`, `compact` is `session.compress`, `get_state` is `session.status`, `get_messages` is `session.history`, `fork` is `session.branch`, `set_model` is `command.dispatch` with `/model ...`.

## OpenAI-compatible API server
> Sources: [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [gateway/platforms/api_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/api_server.py)

Enable and run (values are placeholders; the docs say `API_SERVER_KEY` is required for every deployment, including loopback):

```bash
# ~/.hermes/.env
API_SERVER_ENABLED=true
API_SERVER_KEY=<your-api-key>
# API_SERVER_CORS_ORIGINS=http://localhost:3000   # only if a browser calls Hermes directly
hermes gateway
```

Defaults per the docs: host `127.0.0.1`, port `8642`; `API_SERVER_PORT`, `API_SERVER_HOST`, `API_SERVER_MODEL_NAME` also exist. The same settings may live under `gateway.api_server:` in `config.yaml` (environment variables take precedence).

Authentication is a Bearer token in `Authorization: Bearer <key>`. With multi-profile routing (`gateway.multiplex_profiles`), `/p/<profile>/...` requires that profile's own `API_SERVER_KEY`; a default-profile key on a named prefix returns `401` (July 2026 breaking change per the docs), and run ids are scoped per profile (another profile's id returns `404`).

Endpoint groups (as documented; see the API Server page for request/response detail):

| Group | Endpoints |
|---|---|
| OpenAI-compatible | `POST /v1/chat/completions` (SSE streaming), `POST /v1/responses`, `GET/DELETE /v1/responses/{id}`, `GET /v1/models` |
| Runs | `POST /v1/runs` (202 with run id), `GET /v1/runs/{id}`, `GET /v1/runs/{id}/events` (SSE), `POST /v1/runs/{id}/approval`, `/clarify`, `/steer`, `/stop`, `/resolve-unknown` |
| Sessions | `GET/POST /api/sessions`, `GET/PATCH/DELETE /api/sessions/{id}`, `GET .../messages`, `POST .../fork`, `POST .../chat`, `POST .../chat/stream` |
| Jobs | `/api/jobs` list/create/get/patch/delete, `.../pause`, `.../resume`, `.../run` |
| Discovery and health | `GET /v1/capabilities`, `GET /api/model/options`, `GET /health`, `GET /health/detailed`; skills and toolsets discovery endpoints |

Headers and semantics worth knowing:

- `X-Hermes-Session-Id` continues a transcript; `X-Hermes-Session-Key` (max 256 chars) gives a stable per-channel scope for long-term memory; `Idempotency-Key` on chat completions and responses gives durable admission identity in canonical mode (conflicts return HTTP 409 `admission_conflict`).
- Per-request model selection uses `model`, `provider` and `model_options` fields. On `/v1/chat/completions` and `/v1/responses` a bare `model` without `provider` is ignored unless `gateway.platforms.api_server.direct_model_requests: true`. The docs do not document an `X-Hermes-Model` header. `GET /v1/models` stays minimal; use `/api/model/options` for the Hermes-aware picker.
- Terminal run status is derived from how the turn ended (`completed`, `cancelled`, `failed`, `interrupted`), with matching `run.<status>` events; steer text is queued, not guaranteed consumed (`pending_steer` on the terminal event).
- System prompts from clients are layered on top of Hermes' core prompt, not replacing it.
- Limitations: stored responses persist in SQLite (max 100, LRU); no file upload (inline images are supported); no browser-extension control protocol (removed; use `browser.cdp_url` or `/browser connect`).

### Concurrent-run cap
> Sources: [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [gateway/platforms/api_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/api_server.py)

`gateway.api_server.max_concurrent_runs` (default 10; `0` disables; negatives clamp to 0) limits agent runs started directly through the OpenAI-compatible, Runs and session-chat endpoints. Excess requests get HTTP 429 `Too many concurrent runs (max N)`. Cron-triggered runs follow cron's own limits. The docs show the key under `gateway.api_server` (the block is also accepted under `gateway.platforms.api_server` or top-level `platforms.api_server`) and do not document a `Retry-After` header for the 429.

```yaml
gateway:
  api_server:
    enabled: true
    port: 8642
    host: 127.0.0.1
    max_concurrent_runs: 10     # 0 disables the limit
```

## Python in-process embed
> Sources: [Using Hermes as a Python Library](https://hermes-agent.nousresearch.com/docs/guides/python-library) · [run_agent.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/run_agent.py)

```python
from run_agent import AIAgent

agent = AIAgent(model="<provider-model-id>", quiet_mode=True)
print(agent.chat("What is the capital of France?"))

result = agent.run_conversation(user_message="Explain quicksort", task_id="my-task-1")
print(result["final_response"], len(result["messages"]))
```

Documented guidance: set `quiet_mode=True` when embedding; use `enabled_toolsets` / `disabled_toolsets`, `skip_context_files`, `skip_memory` (recommended for stateless endpoints), `ephemeral_system_prompt`, `max_iterations` (documented default 500), `platform`; create one `AIAgent` per thread or task. The docs install this from a PM-prepared source checkout (`source ./activate`); Hermes does not publish a supported wheel for `requirements.txt` installs. Unlike the other three surfaces, the guide presents this as a direct in-process agent; the docs do not describe it attaching to the gateway.

## Model switching across surfaces
> Sources: [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [gateway/platforms/api_server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/platforms/api_server.py)

Mid-session switching is the `/model` slash command: CLI/TUI `/model <name>` or `/model <provider>:<model>`; TUI-gateway RPC `command.dispatch` with `{"command": "/model ..."}`; ACP sends the slash command as a prompt (and the service-tier picker is a session option exposed through `configOptions`); API server per-request `model` / `provider` / `model_options` fields as above. `/model reset` returns a pinned session to the configured default (v0.23.0 notes). Provider-aware resolution lives in `hermes_cli/model_switch.py`.

## Choosing a protocol
> Sources: [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) · [tui_gateway/server.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tui_gateway/server.py)

- IDE plugin whose host speaks ACP: ACP.
- Custom desktop/web/TUI host wanting slash commands, approvals, clarify, multi-agent and branching: TUI gateway (`hermes gateway attach --json`).
- OpenAI-compatible frontend, curl or CI: API server.
- Python script or service: import `AIAgent`.

Not covered: Group Chat and hosted-room RPC, the Webapp's sign-in and token model, the dashboard's `/api/ws` bridge, canonical OpenRPC contracts, and per-endpoint request schemas. See the cited pages.

---

# 24. Slash Commands Reference

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [Gateway command parity](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-command-parity) · [hermes_cli/commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/commands.py)

Hermes has two slash-command surfaces that are both driven by one central registry, `COMMAND_REGISTRY` in `hermes_cli/commands.py`:

- **Interactive CLI commands** are dispatched by `cli.py`; autocomplete and the command palette are generated from the registry.
- **Messaging commands** are dispatched by `gateway/run.py` (handlers live in `gateway/slash_commands_*.py` mixins); help text and platform command menus are generated from the registry.

This section summarises the groups the reference page uses. It is a map, not a full copy: for exact flags and wording always open the reference page. This guide does not state a command count; the reference and the parity table are generated from the registry and change between releases. To enumerate commands on your install, type `/help` (grouped), `/help <text>` (filter), `/palette` (CLI/TUI fuzzy palette, also Ctrl+P) or, in messaging, `/commands [page]`.

## How the registry works

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [Extending the CLI](https://hermes-agent.nousresearch.com/docs/developer-guide/extending-the-cli) · [hermes_cli/commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/commands.py)

Each command is one `CommandDef` entry. Fields visible in the source at this SHA include `name`, `description`, `category`, `aliases`, `args_hint`, `subcommands`, `cli_only`, `gateway_only`, `gateway_config_gate` (a config dotpath that, when truthy, overrides `cli_only` for the gateway), `busy_policy` / `busy_handler` (what the gateway does while an agent run is in progress: dispatch, reject, or interrupt-then-dispatch), `execute` (a key into `hermes_cli.slash_exec.EXECUTORS`) and desktop-composer hints (`argument_mode`, `desktop`, `desktop_subcommands`).

Resolution on the gateway (per the gateway internals page):

```
user input "/mod"
   -> resolve_command()            hermes_cli/commands.py (aliases, prefix match)
   -> canonical name in GATEWAY_KNOWN_COMMANDS ?
   -> _handle_message()            handler looked up by name: _handle_<name>_command
   -> gateway_config_gate ? / busy_policy check if an agent is running
```

Rules worth knowing (all from the reference page):

- Commands are case-insensitive and support **prefix matching**: `/h` resolves to `/help`. If a prefix is ambiguous the first match in registry order wins; full names and registered aliases always beat prefix matches.
- Installed skills are exposed as dynamic `/<skill-name>` commands on both surfaces. A skill whose name collides with a built-in command (or alias) never gets its own slash command; the built-in wins and the skill stays loadable via `/skill <name>`. `/plan` used to be a skill and is now a built-in.
- **Quick commands** (`quick_commands:` in `config.yaml`) map a short name to `type: exec` (a shell command) or `type: alias` (another slash command). String-only prompt shortcuts are not supported; put reusable prompts in a skill.
- **Custom model aliases** (`model_aliases:` full form, or `hermes config set model.aliases.<name> <provider>/<model>`) are reachable with `/model <alias>`; user aliases shadow built-in short names.
- **Slack threads**: Slack blocks native slash commands inside threads, so the gateway accepts a `!` prefix there (`!stop`, `!new`, `!status`).
- **Admin/user split**: on platforms with per-user allowlists, admins get every command while regular users get only `user_allowed_commands` plus the always-allowed `/help` and `/whoami`. If `allow_admin_from` is unset for a scope, that scope stays in unrestricted backward-compatible mode. `/whoami` shows your level.

## Interactive CLI commands by group

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [hermes_cli/commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/commands.py)

The reference page splits CLI commands into Session, Configuration, Tools & Skills, Info and Exit. Representative entries (not exhaustive):

### Session

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

| Command | What the docs say it does |
|---|---|
| `/new [name]` (alias `/reset`) | Fresh session ID and history; optional `[name]` sets the initial title. Append `now`, `--yes` or `-y` to skip the confirmation modal. |
| `/clear` | Clear the screen and start a new session. |
| `/history`, `/title`, `/save <json\|md\|html> [filename] [redact]` | Show history; set a title; export the session (`redact` scrubs keys and tokens and must come last). |
| `/retry`, `/undo` | Resend the last message; remove the last user/assistant exchange. |
| `/rewind [N \| fork N]` | Backtrack in the classic CLI (also Esc, Esc on an idle empty prompt): opens a transcript browser; Enter puts the chosen prompt back in the composer and cuts the session back to just before it (later messages archived, as with `/undo`); `f` forks into a new branched session instead and keeps the original. `/rewind N` edits prompt N directly; `/rewind fork N` forks before it. Listed as client-local (CLI-only) in the parity table. |
| `/compress [here [N] \| focus topic]` | Manual context compression; `here [N]` keeps the most recent N exchanges (default 2) verbatim. |
| `/rollback`, `/diff [staged\|all\|session]`, `/snapshot` (alias `/snap`) | Filesystem checkpoints; git diff views (`session` shows everything Hermes changed, needs checkpoints); config/state snapshots. |
| `/stop`, `/queue <prompt>` (`/q`), `/steer <prompt>`, `/busy` | Kill background processes; queue a prompt for the next turn; inject a note after the next tool call without interrupting; choose queue/steer/interrupt behaviour when you message mid-run. |
| `/goal`, `/subgoal`, `/heartbeat` (`/hb`), `/loop` (`/proactive`) | Standing goal judged after each turn (default budget 20 turns, `goals.max_turns`); extra criteria for the active goal; recurring in-session prompt; re-run a prompt or command on an interval. |
| `/refine`, `/review`, `/moa`, `/learn` | Run the memory/skill self-improvement review now; spawn an independent reviewer subagent; one-shot Mixture-of-Agents preset; distil a skill from a description. |
| `/resume [name]`, `/sessions`, `/branch [--here] [name]` (`/fork`), `/handoff <platform>`, `/bg`, `/btw`, `/agents` (`/tasks`) | Session navigation, branching, hand-off to a messaging platform (CLI-only), background and side-question runs, active agent list. |
| `/status`, `/context [all]` (`/ctx`), `/egress [status]` | Session info plus a local recap; context-window breakdown; Docker egress-proxy status (CLI, TUI, Desktop and gateway). |

### Configuration

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

| Command | What the docs say it does |
|---|---|
| `/config`, `/statusbar`, `/skin`, `/verbose`, `/focus`, `/footer`, `/timestamps`, `/indicator` | Show config; display toggles. `/focus` and `/verbose` share the `display.tool_progress` suppression path. |
| `/model [name]` | Show or switch the model. Forms include `provider:model`, `custom:model`, `custom:name:model`, `custom`, `@preset/<slug>` and user aliases. Flags: `--global` (persist), `--session`, `--once` (next turn only), `--refresh`, `--provider <name>`, `--reasoning <level>`. **`/model reset`** drops the session pick and returns to the `config.yaml` model without losing the conversation. A plain `/model <name>` is session-only unless `model.persist_switch_by_default: true`. Running `/model` bare opens a provider-to-model picker with type-to-filter. A mid-session model switch resets the prompt cache. |
| `/reasoning [level\|show\|hide\|full\|clamp] [--global]`, `/fast [normal\|fast\|auto\|cold\|status]`, `/personality`, `/codex-runtime` | Reasoning effort and display, fast mode, personality overlay, optional Codex app-server runtime. |
| `/yolo`, `/approvals [manual\|smart\|off]` | Toggle YOLO; show or set the persistent approval mode (see section 25). |
| `/export`, `/import` | CLI-only profile archive export/import; credentials (`auth.json`, `.env`) are stripped from exports. |
| `/voice`, `/wake`, `/battery`, `/autoprune` | Voice mode, wake word, battery read-out, automatic pruning of old ended sessions (off by default). |

### Tools and Skills

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

| Command | What the docs say it does |
|---|---|
| `/tools [list\|disable\|enable] [name...]`, `/toolsets` | List or toggle tools for the current session (disabling resets the session); list toolsets. |
| `/browser [connect\|disconnect\|status]` | Local Chromium-family CDP connection. |
| `/skills`, `/memory [pending\|approve\|reject\|approval]`, `/bundles`, `/curator`, `/reload-skills` | Skill hub plus the skill write-approval review surface; memory write-approval gate; skill bundles; background skill maintenance; re-scan skills. |
| `/plan [task]`, `/init [notes]` | Write a plan to `.hermes/plans/` (no execution); generate or update `AGENTS.md` from a repo scan. |
| `/cron`, `/suggestions` (`/suggest`), `/blueprint` (`/bp`), `/kanban <action>` | Scheduled tasks, suggested automations, cron blueprints, the collaboration board. |
| `/reload-mcp` (`/reload_mcp`), `/reload`, `/plugins` | Reload MCP servers; reload `.env`; list/update plugins. |

### Info and Exit

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands)

`/help`, `/palette`, `/version`, `/whoami`, `/usage`, `/insights`, `/topup`, `/login`, `/update`, `/platforms` (`/gateway`), `/profile`, `/debug` (uploads a debug report), plus clipboard helpers (`/paste`, `/copy`, `/image`). `/quit` (also `/exit`) leaves the CLI; `/exit --delete` or `/quit --delete` also deletes the session history.

## Messaging (gateway) commands

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/run_busy.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_busy.py)

The messaging table covers most of the session and configuration commands above plus gateway-specific ones:

| Command | Notes from the docs |
|---|---|
| `/start` | Platform handshake, acknowledged silently (no agent turn). |
| `/new [name]` (`/reset`), `/stop`, `/status`, `/retry`, `/undo`, `/queue`, `/steer` | Session control. `/stop` kills background processes and interrupts the running agent. |
| `/model [provider:model]` | Includes `/model reset`, which clears this chat's session override so `channel_overrides` / `config.yaml` decide the model again. `/model` only switches between already-configured providers; add providers with `hermes model` in a terminal. |
| `/approve [session\|always]`, `/deny`, `/yolo` | Answer a pending dangerous-command approval (`always` adds to the permanent allowlist); toggle YOLO. |
| `/sethome` (`/set-home`), `/restart`, `/pause [reason \| off]`, `/discard`, `/topic`, `/platform <list\|pause\|resume> [name]`, `/commands [page]` | Messaging-only operations: set the home channel, gracefully restart the gateway, global emergency hold on cron/kanban/gateway work, drop a turn lost in a restart, Telegram topic mode, per-adapter operations, paginated command browser. |
| `/sessions [all] [search <query>]`, `/resume`, `/branch`, `/save`, `/title`, `/bg`, `/btw`, `/goal`, `/loop`, `/heartbeat`, `/kanban`, `/learn`, `/plan`, `/init` | Work on messaging with the same argument surface as the CLI; `/branch` opens a sibling thread on thread-capable platforms. |
| `/<skill-name>` | Invoke any installed skill. |

Per-surface availability is spelled out in the reference "Notes" block: for example `/skin`, `/snapshot`, `/export`, `/import`, `/tools`, `/browser`, `/config`, `/cron`, `/handoff` and `/quit` are CLI-only (the parity table separately lists `/rewind` as client-local), while `/sethome`, `/restart`, `/approve`, `/deny`, `/topic`, `/platform`, `/pause`, `/discard` and `/commands` are messaging-only. `/verbose` is CLI-only unless `display.tool_progress_command: true` is set. The page also notes `/connectors` as TUI-only.

### Running-agent guard

> Sources: [Gateway Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-internals) · [gateway/run_busy.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/run_busy.py)

While an agent run is active for a chat, `_dispatch_busy_slash_command()` in `gateway/run_busy.py` routes each recognised command by its `busy_policy` / `busy_handler`: a mid-run variant if one exists, otherwise the normal handler when the policy allows, otherwise a reject message such as "Agent is running - `/model` can't run mid-turn". The docs list `/stop`, `/new`, `/approve`, `/deny`, `/queue` and `/status` as bypass commands with mid-run handlers.

## Destructive-command confirmation

> Sources: [Slash Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/slash-commands) · [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

The CLI asks before `/clear`, `/new` / `/reset`, `/undo` and `/exit --delete` / `/quit --delete`. The modal offers Approve Once, Always Approve (persists `approvals.destructive_slash_confirm: false`) or Cancel. Append `now`, `--yes` or `-y` to skip it for one invocation. Setting `approvals.destructive_slash_confirm: false` disables it globally.

## Client parity (CLI, TUI, Desktop, ACP)

> Sources: [Gateway command parity](https://hermes-agent.nousresearch.com/docs/developer-guide/gateway-command-parity) · [gateway/session_commands.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/session_commands.py)

Every local client (classic CLI, Ink TUI, Desktop, ACP) attaches to the one gateway that owns the session. A slash command either runs inside the client against a canonical verb (for example `session.mutate`), or goes to the owner as `slash.exec`, where `gateway/session_commands.py` runs only reviewed reads and refuses the rest (`unsupported_command`). The generated parity table (do not edit by hand; a test fails when it is stale) classifies each registry command per client as `read`, `control` (admission-fenced write, such as `/approve` and `/deny` resolving this session's waiting approval) or `refused`, and flags client-local (`cli_only`) commands. Consult that page when you need to know whether a given command works in the TUI, Desktop or an editor over ACP.

## Adding or changing a command

> Sources: [Extending the CLI](https://hermes-agent.nousresearch.com/docs/developer-guide/extending-the-cli) · [CLI Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cli-internals) · [Contributing](https://hermes-agent.nousresearch.com/docs/developer-guide/contributing)

Built-in commands are defined once in `COMMAND_REGISTRY`; handlers live in `cli.py` (`HermesCLI._handle_<name>_command`) and the gateway mixins. The extending-the-CLI page documents `process_command()` as a pre-existing hook for wrapper CLIs that want their own `/mycommand` handling (override it and call `super().process_command(cmd)` for everything else). The contributing page warns that CI (`tests/ci/test_docs_drift.py`) fails when a new slash command or `DEFAULT_CONFIG` key has no docs. Not covered here: the full per-command argument grammar; use the reference page.

---

# 25. Security Model

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [SECURITY.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/SECURITY.md) · [tools/approval.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval.py)

The Security page describes a defense-in-depth model with eight layers: user authorization, dangerous-command approval, file-write safety, container isolation, MCP credential filtering, context-file scanning, cross-session isolation, and input sanitization of terminal working directories. The repo-root `SECURITY.md` adds the formal trust model, which is the most important framing for this section: **the only security boundary against an adversarial LLM is the operating system.** The approval gate, output redaction, pattern scanners and tool allowlists are in-process heuristics (accident prevention), not containment.

## Trust model (SECURITY.md)

> Sources: [SECURITY.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/SECURITY.md) · [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

- Hermes is described as a single-tenant personal agent. The operator's user account defines the "trust envelope".
- **Two OS-level isolation postures**: (1) *terminal-backend isolation*, where LLM-emitted shell and the file tools run in a container, remote host or cloud sandbox (confines shell and file-tool actions only; code execution, MCP subprocesses, plugin loading, hook dispatch and skill loading still run in the agent process); and (2) *whole-process wrapping* (the official Docker image/Compose setup, or an external sandbox runtime), which subjects every code path to the same policy. The doc says whole-process wrapping is the supported posture when ingesting content from surfaces the operator does not control, and for production or shared deployments.
- **Credential scoping** strips provider keys and gateway tokens from shell, MCP, cron-script and code-execution children by default; this reduces casual exfiltration but is "not containment".
- **Skills and plugins** run with full agent privileges; the boundary is operator review before install (read the Python and scripts, not just `SKILL.md`).
- **External surfaces** (gateway adapters, network HTTP surfaces, ACP, the TUI gateway): authorization is required at every trust-crossing surface; every enabled network-exposed adapter needs an allowlist; session IDs are routing handles not authorization; all callers in the authorized set are equally trusted; binding a local-only surface to a non-loopback interface is a break-glass operator decision.
- **Deployment hardening** list: run as non-root, keep credentials in the credential file with tight permissions, do not expose the gateway or API to the public internet without a VPN or firewall, configure a caller allowlist for every network adapter, review third-party skills and plugins.

## Dangerous command approval

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [tools/approval.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval.py) · [tools/approval_detection.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval_detection.py)

Before executing a terminal command Hermes checks it against curated dangerous patterns. At this SHA the approval code is split: `tools/approval.py` owns the gate flow and per-session state and imports leaf modules (pattern detection in `approval_detection.py`, pre-gate floors in `approval_floors.py`, prompts in `approval_prompt.py`, the smart-mode guardian in `approval_smart.py`, and others). The patterns themselves (`DANGEROUS_PATTERNS`, `HARDLINE_PATTERNS`) are defined in `tools/approval_detection.py`.

### Approval modes and keys

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

```yaml
approvals:
  mode: smart                     # smart | manual | off
  timeout: 300                    # seconds to wait for a reply
  cron_mode: deny                 # deny | approve (headless cron)
  single_query_mode: deny         # deny | approve (hermes chat -q)
  unattended_mode: deny           # deny | approve (webhook / api_server sessions)
  mcp_reload_confirm: true
  destructive_slash_confirm: true
```

| Mode | Behavior per the docs |
|---|---|
| `smart` (default) | An auxiliary LLM assesses risk: low-risk commands are auto-approved for that command only, genuinely dangerous ones are auto-denied, uncertain cases escalate to a manual prompt. |
| `manual` | Always prompt on dangerous commands. |
| `off` | Disables approval checks (equivalent to `--yolo`), and also skips the `trust: untrusted` MCP write prompt. |

An approval that times out is denied (fail-closed); an expired prompt cannot be reopened, and the user must ask again to get a fresh card. In the CLI the prompt offers once / session / always / deny (default deny); "always" writes to `command_allowlist` in `config.yaml`. On messaging platforms the user replies yes/no (or uses `/approve` and `/deny`). The gateway sets `HERMES_EXEC_ASK=1`.

### YOLO mode

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

YOLO bypasses dangerous-command approval prompts. It is enabled with `hermes --yolo` / `hermes chat --yolo`, `/yolo` (a toggle, scoped to the session it was typed in and saved with it), or `HERMES_YOLO_MODE=1`. The CLI shows a red banner and a status-bar `YOLO` fragment while active. YOLO does **not** skip: the hardline blocklist, user deny rules, an MCP server's `spend_limit`, MCP elicitation questions, payment-card fill prompts from the credential vault, or MCP event subscriptions.

### Hardline blocklist and user deny rules

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [tools/approval_detection.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval_detection.py)

- **Hardline blocklist**: catastrophic commands (root-level recursive delete, fork bombs, formatting a mounted root device, zeroing a physical disk, piping untrusted URLs to a shell at the filesystem root) are refused regardless of YOLO, `approvals.mode: off`, headless-approve cron, or "allow always". There is no override flag; the docs say the list is non-exhaustive. The docs name `UNRECOVERABLE_BLOCKLIST` in `tools/approval.py`; at this SHA the hardline patterns are defined as `HARDLINE_PATTERNS` in `tools/approval_detection.py`. A command whose shell quoting cannot be parsed fails closed.
- **`approvals.deny`**: a user-editable list of case-insensitive fnmatch globs that block matching terminal commands unconditionally, before YOLO and `mode: off` are consulted. Matching runs over normalised variants and parses launchers such as `sudo`, `env`, `nohup`, `timeout` and shell `-c` payloads. Always quote patterns in YAML. The docs state this is a shell-command policy, not a complete interpreter or OS sandbox; use OS permissions and an isolated backend for containment.
- **Supervised-gateway lifecycle guard**: a non-overridable terminal-tool guard refuses commands that stop or restart the gateway from inside its own supervised process, including process killers aimed at the Python interpreter image. Approval, YOLO and `force=True` do not bypass it.

### What triggers approval

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [tools/approval_detection.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/approval_detection.py)

The page lists pattern classes rather than a single constant. Grouped:

| Class | Examples in the docs |
|---|---|
| Destructive filesystem | `rm -r` / `--recursive`, `find -exec rm` / `-delete`, `xargs rm`, recursive `chown` to root, world-writable `chmod`, `mkfs`, `dd if=`, writes to a block device |
| System config | writes via `>`, `tee`, `cp`/`mv`/`install` or `sed -i` to `/etc/`, `~/.ssh/`, `~/.hermes/.env` |
| SQL | `DROP TABLE/DATABASE`, `DELETE FROM` without `WHERE`, `TRUNCATE TABLE` |
| Processes and services | `kill -9 -1`, `pkill -9`, `systemctl stop/restart/disable/mask`, fork bombs, killing hermes/gateway, starting `gateway run` outside the service manager |
| Remote code | `curl ... \| sh`, `wget ... \| sh`, `bash <(curl ...)`, `bash -c` / `python -e` style inline execution |
| Containers | `docker stop/kill/restart`, `compose down`, daemon redirects (`docker -H`, `DOCKER_HOST=`, `docker context use`), Podman remote flags |
| Content-level | a `curl`/`wget` body carrying a secret or credential file; invisible or bidirectional Unicode control characters in a command |

Backends that are themselves the boundary (`docker`, `modal`, `daytona`, `vercel_sandbox`, and container plugin backends such as `singularity` or `sprites`) skip these checks. `execute_code` scripts get a whole-script approval in gateway/API sessions; local interactive sessions follow the CLI rule where only the script's `terminal()` calls ask (the docs list exceptions that keep whole-script approval). The earlier external scanner (tirith) was removed; upgrading drops `security.tirith_*` settings and enables nothing in its place.

### Permanent allowlist and mining history

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

`command_allowlist` (list of strings in `config.yaml`) holds exact commands, globs, or dangerous-pattern rule keys. It is read at startup; removing an entry does not revoke it from a running session. `hermes approvals suggest` mines past approvals from the session database into allowlist proposals: dry-run by default, `--apply N[,M]` to write, `--json`, `--days` (default 90), `--min-count` (default 2). Destructive classes and hardline classes are never proposed, and credentials in mined commands are masked.

## File write safety

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

`write_file` and `patch` check a denylist before touching disk; blocked writes return an error with **no approval prompt**. Always-blocked categories: OS credential stores (SSH keys and `authorized_keys`, cloud and Kubernetes credential directories, sudoers, netrc), Hermes secret stores (`.env`, OAuth token files, the vault, browser profile, `mcp-tokens/`, `pairing/`, secret-manager caches) and Windows NT/device-namespace paths (also blocked for reads). Project `.env*` files and `.envrc` are read-denied but writable. `~/.ssh/config` is the documented exception: it is approval-gated rather than blocked.

`HERMES_WRITE_SAFE_ROOT` optionally restricts writes to listed directory prefixes (colon-separated on Unix, semicolon on Windows); it is set automatically in the official Docker image. The docs call all of this defense in depth: the `terminal` tool runs as the same OS user and can still touch denied paths, so the denylist does not sandbox a hostile agent.

## User authorization (gateway)

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [gateway/pairing.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/gateway/pairing.py)

Authorization check order: per-platform allow-all flag, DM-pairing approved list, platform allowlist (for example `TELEGRAM_ALLOWED_USERS`), global `GATEWAY_ALLOWED_USERS`, global `GATEWAY_ALLOW_ALL_USERS`, then **deny by default**. With no allowlists configured every unauthorised user is denied and the gateway logs a warning.

DM pairing: an unknown user gets an 8-character one-time code that the owner approves with `hermes pairing approve <platform> <code>` (also `list`, `revoke`, `clear-pending`). The docs list: 1-hour code TTL, one request per user per 10 minutes, at most 3 pending codes per platform, 5 failed approvals trigger a 1-hour lockout, data files at mode `0600` under `~/.hermes/pairing/`, codes never logged. `unauthorized_dm_behavior` is `pair` (default for chat DM platforms), `ignore`, or `decline`; email defaults to `ignore`. For the Docker image, run pairing commands as the unprivileged `hermes` user (`docker exec -u hermes ...`) or the approval file is unreadable by the gateway.

For the slash-command admin/user tier see section 24.

## Container isolation and terminal backends

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Docker](https://hermes-agent.nousresearch.com/docs/user-guide/docker) · [tools/environments/docker.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/environments/docker.py)

The Docker terminal backend applies hardening flags defined in `tools/environments/docker.py`: all capabilities dropped with a minimal re-add set (`DAC_OVERRIDE`, `CHOWN`, `FOWNER`), `no-new-privileges`, a PID limit of 256, and size-limited tmpfs mounts (`/tmp` 512m; `/var/tmp` no-exec, 256m). `SETUID`/`SETGID` are added only when an init/entrypoint must drop privileges. Resource limits are set under `terminal:` (`container_cpu`, `container_memory`, `container_disk`, `container_persistent`).

| Backend | Isolation | Dangerous-command check |
|---|---|---|
| `local` | none (host) | yes |
| `ssh` | remote machine | yes |
| `docker`, `modal`, `daytona`, `vercel_sandbox`, plugin backends (`singularity`, `sprites`) | container / cloud sandbox | skipped (the sandbox is the boundary) |

Environment passthrough is explicit opt-in: skills declare `required_environment_variables` (registered as passthrough only when actually set) or you list names in `terminal.env_passthrough`; credential files go in `required_credential_files` / `terminal.credential_files` and are mounted read-only into Docker. Sandboxes strip variables whose names contain `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `CREDENTIAL`, `PASSWD` or `AUTH` unless declared. Anything in `terminal.docker_forward_env` is intentionally injected and can be exfiltrated by code in the container.

## Egress proxy and network isolation

> Sources: [Egress proxy](https://hermes-agent.nousresearch.com/docs/user-guide/egress/) · [Egress credential-injection proxy](https://hermes-agent.nousresearch.com/docs/user-guide/egress/iron-proxy) · [Network Egress Isolation (Docker)](https://hermes-agent.nousresearch.com/docs/user-guide/egress/network-isolation) · [Egress proxy internals](https://hermes-agent.nousresearch.com/docs/developer-guide/egress-internals) · [agent/proxy_sources/iron_proxy.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/proxy_sources/iron_proxy.py)

The optional egress proxy (`hermes egress install | setup | start | status | restart`, config under `proxy:`; `proxy.enabled` is `false` by default) is an outbound credential-injection firewall for the **Docker** terminal sandbox: the sandbox holds opaque proxy tokens, a host-side iron-proxy daemon terminates TLS with a local CA and swaps tokens for real credentials. The docs state it is wired into the Docker backend only (Modal, Daytona, SSH and Singularity do not receive proxy env vars or CA mounts) and that it does not cover the host process's own LLM calls. The internals page lists invariants such as `0700`/`0600` file modes under the proxy state directory, a minimised subprocess environment, default-deny upstream CIDRs (loopback, link-local including cloud metadata, private ranges, CGNAT), and a loopback-only or Docker-bridge bind. The separate network-isolation guide shows a two-network Docker Compose pattern (an `internal` network with no internet and an `egress` network with an allowlisting proxy) as a second layer against exfiltration via `curl`/`wget`. `/egress [status]` reports proxy state in chat.

## Secrets handling

> Sources: [Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) · [Bitwarden Secrets Manager](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/bitwarden) · [1Password](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/onepassword) · [Command helper](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/command) · [Building a Secret Source Plugin](https://hermes-agent.nousresearch.com/docs/developer-guide/secret-source-plugin) · [agent/secret_sources/base.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/secret_sources/base.py)

- Secrets belong in `~/.hermes/.env` (Security page: keep it `chmod 600`); `config.yaml` is for non-secret settings.
- Hermes can instead pull keys at process startup from external managers: Bitwarden Secrets Manager, 1Password (`op://` references) or a user-configured command helper (`keepassxc-cli`, `pass`, scripts printing `KEY=VALUE`). Precedence per env var: your `.env`/shell wins by default unless a source sets `override_existing: true`; mapped sources beat bulk sources; first source wins. `secrets.preserve_existing` and profile aliasing (`secrets.profile_alias`) make one vault safe across profiles.
- Third-party backends ship as plugins that subclass `agent.secret_sources.base.SecretSource` (one required method, `fetch`); the bundled set is deliberately closed.
- `security.redact_secrets` (on by default) redacts credential-shaped strings in tool output and logs; secret-bearing file reads are also masked. The credential vault feature lets the agent sign in to sites without the model seeing passwords ([Passwords & Logins](https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-vault)).
- Borrowed CLI logins can be disabled with `auth.adopt_external_logins: false`.

## Context file and skill scanning

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

- **Context files** (`AGENTS.md`, `.cursorrules`, project files) are scanned for prompt-injection heuristics (ignore-previous-instructions phrasing, hidden HTML comments, secret-reading attempts, curl exfiltration, invisible Unicode). A hit blocks the file with a visible `[BLOCKED: ...]` notice. Your own `SOUL.md` in `HERMES_HOME` is different: a hit logs a warning and `/context` flags it, but the file still loads. A `SOUL.md` owned by an installed profile distribution is still blocked on a hit.
- **Skills Hub installs** go through a scanner for data exfiltration, prompt injection, destructive commands and supply-chain signals. Trust levels: `builtin` (always trusted), `official` (the repo's `optional-skills/`), `trusted` (named trusted registries) and `community` (everything else). `--force` can override non-dangerous policy blocks but never a `dangerous` verdict. SECURITY.md stresses that Skills Guard is a review aid, not a boundary.
- Project skills discovered from a repository are scanned the same way and need a trusted project root (`skills.trusted_project_dirs`).

## MCP security

> Sources: [MCP](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp) · [MCP Config Reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference) · [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

- MCP stdio subprocesses receive a filtered environment: only `PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TERM`, `SHELL`, `TMPDIR` and `XDG_*`, plus variables explicitly in the server's `env`. MCP error text is redacted before reaching the model.
- `trust: full` (default) or `trust: untrusted` per server: on an untrusted server every write-capable tool (any tool without `readOnlyHint: true`) requires approval. YOLO and `approvals.mode: off` skip that prompt.
- `tool_costs` / `spend_limit` / `cost_unit` give a per-chat budget for paid tools; exceeding it asks even on a `trust: full` server and is refused where nobody can answer (cron, `chat -q`). The total is in memory and resets on restart.
- OAuth tokens for remote MCP servers are cached under `~/.hermes/mcp-tokens/<server>.json` with `0600` permissions; mutual TLS is supported through `client_cert` / `client_key`. The MCP page also documents sampling controls and tool filtering.

## Web access, SSRF and TLS

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)

URL-capable tools (web search/extract, vision, browser) validate URLs: private (RFC 1918) ranges, loopback, link-local (including cloud metadata), CGNAT and metadata hostnames are blocked, DNS failure is treated as blocked, and redirects are re-validated per hop. `security.allow_private_urls: true` is a global opt-out for LAN setups, but cloud-metadata endpoints stay blocked; `security.fake_ip_ranges` exempts a local proxy's fake-IP block. `security.website_blocklist` (`enabled`, `domains`, `shared_files`) blocks domains across web and browser tools. TLS trust uses the platform verifier (`truststore`); a custom provider can declare `ssl_ca_cert`, and `ssl_verify: false` is documented as unsafe for untrusted networks.

## Network exposure and extension trust

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [API Server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) · [Web Dashboard](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard) · [SECURITY.md](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/SECURITY.md)

- The API server binds to loopback by default (`API_SERVER_HOST`) and uses bearer authentication with `API_SERVER_KEY`. The dashboard defaults to a loopback bind; a non-loopback bind engages its auth gate, and the old `--insecure` flag is a deprecated no-op that no longer disables authentication.
- Gateway event hooks (`<profile home>/hooks/<name>/`) are "trusted by placement": placing the directory is the opt-in, with no `plugins.enabled` entry or prompt, and they run in-process with gateway privileges. Treat `~/.hermes/hooks/` like `config.yaml`. General plugins need `plugins.enabled`, and shell hooks need first-use approval.
- Supply chain: a built-in advisory scanner (`hermes_cli/security_advisories.py`) flags known-compromised package versions at CLI startup, in `hermes doctor` and at gateway startup; dismiss with `hermes doctor --ack <id>`. `security.allow_lazy_installs: false` disables on-demand installation of optional extras.
- Profiles isolate config, secrets, memory, sessions and the gateway process; see section 5.

The old guide's sweeping statements ("never use" a specific tunnel product, "the LLM never has access to raw secrets") are not asserted here; the docs state the narrower facts above. Not covered here: per-platform allowlist variables (see the messaging pages), the full Docker image hardening notes, and Kanban/worker-lane trust. Start from the Security page for those.

---

# 26. File Dependency Chain & Import Order

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [model_tools.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/model_tools.py)

The docs do not publish a full import order for the codebase. What they state is a short tool-registration chain, plus a subsystem ownership map. This section reproduces those and records what was checked against source at the SHA.

## Documented tool-registration chain

> Sources: [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) · [Tools Runtime](https://hermes-agent.nousresearch.com/docs/developer-guide/tools-runtime) · [tools/registry.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/registry.py)

The architecture page's chain:

```text
tools/registry.py   (documented as: no deps, imported by all tool files)
       ^
tools/*.py          (each calls registry.register() at import time)
       ^
model_tools.py      (imports tools/registry + triggers tool discovery)
       ^
run_agent.py, cli.py, batch_runner.py, environments/
```

Checked against source at this SHA:

- `tools/registry.py` defines `discover_builtin_tools()`. It scans `tools/` with an AST check for **top-level** `registry.register()` calls, skips `__init__.py`, `registry.py` and `mcp_tool.py`, caches the verdict on disk keyed by file mtime and size, and imports each matching module (import failures are logged, not fatal). A safe-worker mode restricts discovery to a short reviewed list (file, terminal and process-registry tools).
- `registry.py` is not strictly dependency-free: it imports `hermes_constants` and, inside `discover_builtin_tools`, `agent.safe_worker_policy`. Treat "no deps" as "does not import the tool modules".
- `model_tools.py` imports `tools.registry` helpers, `toolsets` and a few tool modules (`tools.arg_coercion`, `tools.todo_tool`), and calls `discover_builtin_tools()` at module import time, so registration happens before any agent instance exists.
- `run_agent.py` imports `from model_tools import get_toolset_for_tool`; `batch_runner.py` imports `TOOL_TO_TOOLSET_MAP` from `model_tools`; `cli.py` imports `model_tools` functions lazily inside functions. Many other entry points (gateway turn code, cron scheduler, ACP adapter, TUI gateway, API server, curator) import `AIAgent` from `run_agent` inside functions rather than at module top, so loading those modules does not by itself construct an agent.
- The architecture page lists `environments/` as a consumer. At this SHA there is no top-level `environments/` directory in the checkout; terminal execution backends live in `tools/environments/` (`local.py`, `docker.py`, `ssh.py`, `modal.py`, `daytona.py`, `vercel_sandbox.py` and helpers). Treat the architecture page's `environments/` entry as legacy wording.

## Subsystem ownership

> Sources: [Codebase Ownership Map](https://hermes-agent.nousresearch.com/docs/developer-guide/codebase-ownership) · [Architecture](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)

The ownership page maps subsystems to directories. Selected rows:

| Subsystem | Source directories |
|---|---|
| Agent core (loop, transports, compression) | `agent/`, `run_agent.py` |
| Prompt assembly | `agent/prompt_builder.py`, `agent/system_prompt.py` |
| Built-in tools | `tools/` |
| Messaging gateway | `gateway/`, `plugins/platforms/` |
| CLI | `hermes_cli/` |
| Plugins | `plugins/` |
| Cron | `cron/` |
| Session storage | `hermes_state.py`, `hermes_state_*.py` |
| Egress firewall | `agent/proxy_sources/iron_proxy.py` |
| ACP | `acp_adapter/` |
| TUI | `ui-tui/`, `tui_gateway/` |

Conventions from the page: keep changes inside their subsystem, run the mirror test directory for every source directory touched, and prefer a fix in an adapter or plugin over a branch in the agent core ("the core is a narrow waist"). The architecture page's design principle "Loose coupling" says optional subsystems (MCP, plugins, memory providers) use registry patterns and `check_fn` gating rather than hard dependencies.

## System prompt build order

> Sources: [Prompt Assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly) · [agent/system_prompt.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/system_prompt.py) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

The prompt-assembly page describes the cached system prompt as three ordered tiers built in `agent/system_prompt.py`, joined stable then context then volatile:

```text
stable   : identity (SOUL.md or built-in fallback), tool/model guidance, coding operating brief
context  : caller-supplied system_message, project context file (.hermes.md / AGENTS.md /
           CLAUDE.md / .cursorrules, first match wins), git workspace snapshot,
           operator instructions, platform hints
volatile : skills index, MEMORY.md snapshot, USER.md snapshot, external memory-provider block,
           timestamp/session/model/provider line, runtime environment hints (ends the tier)
```

This differs from the older chain in this guide, which placed the skills index in the stable tier. See section 7 for the full layer description.

## Configuration precedence

> Sources: [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [hermes_cli/config.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config.py)

The configuration page gives the precedence (highest first): CLI arguments, then `~/.hermes/config.yaml`, then `~/.hermes/.env`, then built-in defaults. Secrets go in `.env`; non-secret settings go in `config.yaml`, which wins when both are set. The older ordering in this guide (environment variables and `.env` overriding `config.yaml`) is not what the page states. An administrator can pin values through a managed-scope directory (see the Managed Scope page); `.env` loading at agent import time goes through `hermes_cli/env_loader.py` (`load_hermes_dotenv`), called from `run_agent.py`.

---

# 27. Key Constants & Defaults Reference

> Sources: [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example) · [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)

Only values that could be cited to the docs, `cli-config.yaml.example`, or a named source constant at SHA `9f23c4311a4c4837078081db4f3cd8deab23c0fe` are listed. Defaults change between releases; verify on your install with `hermes config` or the Configuration page. Where two upstream sources disagree this is stated in the row. This section deliberately omits several values from the older guide that could not be re-verified or were found changed.

## Agent and loop

> Sources: [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [cli-config.yaml.example](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cli-config.yaml.example) · [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)

| Setting | Value | Source |
|---|---|---|
| `agent.max_turns` | The Configuration page says unlimited by default (`none`; a positive integer caps it). The comment in `cli-config.yaml.example` still says "default: 500". The two sources conflict at this SHA; treat the Configuration page as authoritative and set it explicitly if you rely on a cap. | Configuration page; cli-config.yaml.example |
| `agent.api_max_retries` | 3 | Configuration page |
| `delegation.max_iterations` | 250 per child | cli-config.yaml.example; Delegation page ("default: 250") |
| `goals.max_turns` | 20 continuation turns | Configuration page; Goals page |
| `api_server.max_concurrent_runs` | 10 in the docs example (`0` disables the limit) | API Server page |

## Context compression

> Sources: [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) · [Context Compression & Caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching) · [agent/context_compressor.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/context_compressor.py)

| Setting | Value | Source |
|---|---|---|
| `compression.enabled` | `true` | cli-config.yaml.example |
| `compression.threshold` | `0.50` of the model context. The example notes models with windows below 512K are floored at 0.75 (raise-only). | cli-config.yaml.example |
| `compression.target_ratio` | `0.20` (range 0.10 to 0.80); tail budget = target_ratio x threshold x context length | cli-config.yaml.example |
| `compression.protect_last_n` | `20` messages | cli-config.yaml.example; constructor default in `context_compressor.py` |
| `compression.protect_first_n` | `3` (configurable; the example describes it as system prompt plus the first three non-system head messages) | cli-config.yaml.example |
| `compression.idle_compact_after_seconds` | `0` (disabled) | cli-config.yaml.example |
| `compression.hygiene_max_turn_hold_seconds` | `10` | cli-config.yaml.example |
| `compression.hygiene_hard_message_limit` | `5000` (gateway-only safety valve) | Configuration page |
| `_SUMMARY_RATIO` | `0.20` | `agent/context_compressor.py` |
| `_MIN_SUMMARY_TOKENS` | `2000` | `agent/context_compressor.py` |
| `_SUMMARY_TOKENS_CEILING` | `10_000` in source; the config example comment separately mentions a 12K-token cap on summary output, so the two figures should not be assumed equal | `agent/context_compressor.py`; cli-config.yaml.example |

## Tool output and prompt size limits

> Sources: [Context Files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) · [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) · [tools/budget_config.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/budget_config.py) · [agent/prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/agent/prompt_builder.py)

| Constant | Value | Source |
|---|---|---|
| `DEFAULT_RESULT_SIZE_CHARS` | 100,000 (scaled down for small context windows by the budget code) | `tools/budget_config.py` |
| `DEFAULT_TURN_BUDGET_CHARS` | 200,000 (same scaling) | `tools/budget_config.py` |
| `DEFAULT_PREVIEW_SIZE_CHARS` | 1,500 | `tools/budget_config.py` |
| `CONTEXT_FILE_MAX_CHARS` | 20,000 floor; without an explicit `context_file_max_chars` the cap scales with the model window (fraction 0.06 of window chars) up to a 500,000 ceiling | `agent/prompt_builder.py`; Context Files page |
| Context file truncation | 70% head / 20% tail with a marker | `agent/prompt_builder.py` (`CONTEXT_TRUNCATE_HEAD_RATIO`, `CONTEXT_TRUNCATE_TAIL_RATIO`); Context Files page |
| `MEMORY.md` limit | 2,200 characters | Persistent Memory page |
| `USER.md` limit | 1,375 characters | Persistent Memory page |

## Approvals, pairing and containers

> Sources: [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) · [tools/environments/docker.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/tools/environments/docker.py)

| Setting | Value | Source |
|---|---|---|
| `approvals.mode` | `smart` | Security page |
| `approvals.timeout` | 300 seconds (timeout means deny) | Security page |
| `approvals.cron_mode`, `single_query_mode`, `unattended_mode` | each `deny` | Security page |
| `approvals.mcp_reload_confirm`, `destructive_slash_confirm` | each `true` | Security page |
| `security.redact_secrets` | `true` | Configuration page |
| `security.allow_private_urls` | `false` | Security page; cli-config.yaml.example |
| Pairing code | 8 characters, 1-hour TTL, 3 pending per platform, 5 failures then 1-hour lockout | Security page |
| Docker `--pids-limit` | 256 | Security page; `tools/environments/docker.py` |
| Docker tmpfs | `/tmp` 512m, `/var/tmp` 256m (no-exec) | Security page |
| `terminal.container_cpu` / `container_memory` / `container_disk` | 1 core / 5120 MB / 51200 MB (as shown in the docs example) | Security page |
| `proxy.enabled`, `proxy.tunnel_port` | `false`, `9090` | Egress proxy page |

## Cron

> Sources: [Scheduled Tasks (Cron)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) · [Cron Internals](https://hermes-agent.nousresearch.com/docs/developer-guide/cron-internals) · [cron/scheduler.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/cron/scheduler.py)

| Setting | Value | Source |
|---|---|---|
| Scheduler tick | every 60 seconds, called from the gateway's background thread | Cron Internals page; `cron/scheduler.py` docstring |
| `cron.script_timeout_seconds` | 3600 seconds default (1 hour); resolution order is env `HERMES_CRON_SCRIPT_TIMEOUT`, then config, then default. The older guide's 120 s is not what the Cron page states. | Cron page |
| `cron.wrap_response` | the Cron page shows `false` as the opt-out for the response wrapper | Cron page |
| Job store / output | `~/.hermes/cron/jobs.json`, `~/.hermes/cron/output/<job_id>/` | Cron page |

## Session database write contention

> Sources: [Session Storage](https://hermes-agent.nousresearch.com/docs/developer-guide/session-storage) · [hermes_state.py](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_state.py)

The session-database code is split across `hermes_state.py` and many `hermes_state_*.py` modules at this SHA. Constants seen in `hermes_state.py` (class attributes): `_WRITE_RETRY_MIN_S` 0.020 and `_WRITE_RETRY_MAX_S` 0.150 (fast jitter), a slow phase after `_WRITE_RETRY_SLOW_AFTER_S` = 2.0 s with 0.250 to 1.000 s jitter, and a WAL checkpoint every `_CHECKPOINT_EVERY_N_WRITES` = 50 writes. The older guide's retry count of 15 and 1-second timeout were not re-verified and are omitted.

## Skill template tokens and key paths

> Sources: [Creating Skills](https://hermes-agent.nousresearch.com/docs/developer-guide/creating-skills) · [Which File Does What?](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what) · [Sessions](https://hermes-agent.nousresearch.com/docs/user-guide/sessions)

| Token / path | Meaning | Source |
|---|---|---|
| `${HERMES_SKILL_DIR}` | Absolute path to the skill directory | Creating Skills page |
| `${HERMES_SESSION_ID}` | Active session id (left in place if none) | Creating Skills page |
| `~/.hermes/config.yaml` | Non-secret configuration | Configuration page |
| `~/.hermes/.env` | Secrets and API keys | Configuration page |
| `~/.hermes/SOUL.md` | Agent identity | Which File Does What? |
| `~/.hermes/memories/` | `MEMORY.md` and `USER.md` (not at the home root as the older guide stated) | Persistent Memory page |
| `~/.hermes/state.db` | SQLite session database | Sessions page |
| `~/.hermes/pairing/` | Pairing data files | Security page |
| `~/.hermes/mcp-tokens/<server>.json` | MCP OAuth tokens | MCP page |
| `~/.hermes/proxy/` | Egress proxy CA, config, mappings | Egress proxy page |
| `~/.hermes/hooks/` | Gateway event hooks | Security page |

Not listed because not re-verified: gateway PID file location, plugin directory layout (see section 2 and section 20), and tool, toolset or platform counts (see the tools and toolsets reference pages).
