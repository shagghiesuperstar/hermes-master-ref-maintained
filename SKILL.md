---
name: hermes-architecture-reference
description: Use when troubleshooting, designing, implementing, optimizing, maintaining, or reviewing Hermes Agent architecture, internals, config, tools, profiles, memory, MCP, gateway, cron, plugins, skills, provider routing, context compression, or security behavior. Treat official Hermes docs and the NousResearch/hermes-agent repo as authoritative sources; this skill is an offline architecture reference that must be kept in sync.
version: 1.4.0
last_drift_review: 2026-07-05
repo_commit: 5b04a024a (origin/main, v2026.7.1)
author: ss
license: MIT
metadata:
  hermes:
    tags: [hermes, architecture, troubleshooting, maintenance, optimization, implementation, internals, mcp, gateway, memory, skills, cron]
    related_skills: [hermes-agent, memory-stack-snapshot, hindsight]
---

# Hermes Architecture Reference

## Purpose

This skill is the local/offline architecture reference for Hermes Agent. Load it for **any** Hermes Agent troubleshooting, architecture decision, implementation, optimization, maintenance, or drift-review work.

Authoritative sources remain:

1. Official documentation: `https://hermes-agent.nousresearch.com/docs/`
2. Official repository: `https://github.com/NousResearch/hermes-agent`

The attached reference file in this skill was compiled from those two sources and should be treated as a high-value offline SSOT, but **live docs/repo win if they differ**.

## Required usage rule

When the task involves Hermes Agent internals or operations, use this skill together with `hermes-agent`:

1. Load `hermes-agent` for live CLI commands, current operational conventions, and known pitfalls.
2. Load this skill for architecture, file dependency chains, code paths, and design-level reasoning.
3. If a claim affects config, commands, implementation, or troubleshooting, verify against either:
   - official docs, or
   - the local/upstream Hermes repo.

Do not guess Hermes CLI flags, config keys, tool names, provider semantics, MCP schema, or memory behavior.

## Main reference

Read the full architecture reference here:

```text
references/Hermes_Architecture.md
```

Use targeted reads/searches instead of loading the full file into the chat when possible. It is large.

## Targeted quirks

- `references/approvals-mode-quirks.md` — `approvals.mode: off` write/repair playbook (covers `hermes config set` YAML 1.1 coercion, `config get` not existing, `config show` not displaying `approvals:`, and `patch`/`write_file` tool guards that block direct writes — verified 2026-06-28).
- `references/citation-verification.md` — protocol for verifying user/shared docstrings before execution. Lessons from 2026-06-28 where a Shag-supplied procedure failed two CLI calls before any step ran.
- `references/config-migration-and-provider-routing.md` — provider migration playbook: no-`sed` Python-only edits, `model.provider: moa` / `model.default: default` namespace-collision bug, live OpenRouter slug validation, stale-provider purge, native-provider worker-profile creation (`--clone-from` + SOUL rewrite), routing-role vs runnable-profile distinction, AGENTS.md roster hygiene, and a pre-completion verification checklist.
- `references/CHANGELOG-2026-07-05.md` — drift deltas v0.17.0 → v0.18.0 (1098 commits). High-impact architecture changes since the last full reference re-compile: `extra_headers` per provider, persisted per-session `/model`, MoA `fanout: user_turn`, stacked `/skill` invocations, MCP GUI tab, pre-tool-call approval gate, cron profile-secret scope, and 20+ more. **Read this before assuming any section of the main reference is current.**

## Fast lookup map

The reference covers:

- System overview and design principles
- Source directory structure
- `config.yaml` and `.env` semantics
- `HERMES_HOME` and profile isolation
- `SOUL.md`, `MEMORY.md`, `USER.md`
- Prompt assembly and context-file discovery
- `AIAgent` loop internals
- Tool registry, toolsets, tool dispatch, error wrapping
- Built-in tools and platform toolsets
- Skills system and SKILL.md format
- Memory system and provider plugin lifecycle
- Compression, prompt caching, session storage
- MCP integration, naming, filtering, OAuth, reloads
- Provider runtime resolution and API modes
- Gateway internals and messaging platforms
- Cron internals
- Plugin system and memory-provider development
- Custom tools
- ACP / TUI / API server integration
- Slash commands
- Security model
- File dependency chain and import order
- Key constants and defaults

## Drift review protocol

Hercules must review this skill weekly and update it if it drifts from the authoritative sources.

Weekly review checklist:

1. Fetch or inspect the current `NousResearch/hermes-agent` repo.
2. Read the current official docs at `https://hermes-agent.nousresearch.com/docs/`.
3. Compare changed architecture-relevant areas against `references/Hermes_Architecture.md`:
   - CLI command changes
   - config schema/default changes
   - tool/toolset changes
   - MCP behavior
   - memory provider behavior
   - gateway behavior
   - cron behavior
   - skills format
   - provider routing/API modes
   - context compression/LCM behavior
   - security/approval changes
4. Patch `references/Hermes_Architecture.md` and this SKILL.md if needed.
5. Record a short report with:
   - date/time
   - sources checked
   - changes found
   - files updated
   - verification commands/results

## Maintenance commands

Suggested local repo probes:

```bash
hermes --version
hermes config path
hermes config check
hermes tools list
hermes mcp list
python3 -m py_compile ~/.hermes/hermes-agent/run_agent.py
```

Suggested docs/repo checks:

```bash
git -C ~/.hermes/hermes-agent fetch origin main
git -C ~/.hermes/hermes-agent log --oneline -20 origin/main
```

If the local repo path differs, locate it before running repo commands. Do not assume `~/.hermes/hermes-agent` on other machines.

## Pitfalls

- This skill is not a replacement for official docs. It is an offline reference.
- Hermes profile isolation matters. Always run `hermes config path` before claiming which config is active.
- Config edits must use official Hermes CLI commands unless the operator explicitly authorizes direct file edits.
- Avoid hardcoded user paths when writing reusable procedures. Use `hermes config path`, `hermes config env-path`, and profile-aware locations.
- If the weekly review finds drift, patch the skill immediately rather than only reporting it.

### Config-migration workflow pitfalls (operator-mandated, durable)

- **No `sed`/`awk`/shell text tools for file edits on this macOS box.** Use Python (`pathlib`/`yaml`/`re`) or official `hermes config` CLI. Operator directive — treat as binding for this class of work.
- **When the operator gives a numbered sequence with "do not stop / do not ask permission," execute the full sequence and verify each step.** Do not drip one action per turn; do not declare completion without readback (`hermes config check`, `hermes moa list`, active-model readback, file diff). Overclaiming completion drew repeated explicit frustration.
- **`patch`/`write_file` refuse to edit `config.yaml`** (security guard). Fall back to Python structural edits in `execute_code`, not shell.
- **`model.default` must be a real model slug**, never the literal `"default"`, and `model.provider` should be `openrouter` (not `moa`) for routine chat — `provider: moa` forces every turn through the council at ~4x cost, and `default: default` makes the mobile picker fall back to a stale cached label.
- **Validate every OpenRouter slug against the live catalog** (`https://openrouter.ai/api/v1/models`) before writing it to `config.yaml` OR `model-routing.md`. The `:nitro` suffix is a routing modifier on a real base slug.
- **A routing role in `model-routing.md` is not a runnable profile.** Only dirs with a `config.yaml` are real profiles; empty `hermes-*` stubs are kanban labels. Never report "created a profile" when only a doc row was added.
- **When cloning a profile for a worker, rewrite the inherited `SOUL.md`** so it drops the orchestrator identity; `--no-skills` is mutually exclusive with `--clone-from`.
- See `references/config-migration-and-provider-routing.md` for the full checklist, code snippets, and examples.

## Verification

After creating or updating this skill:

```bash
hermes skills list | grep hermes-architecture-reference
```

Then load the skill and confirm the reference file appears under linked files.
