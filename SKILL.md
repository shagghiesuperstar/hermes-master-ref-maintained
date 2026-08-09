---
name: hermes-architecture-reference
description: Use when troubleshooting, designing, implementing, optimizing, maintaining, or reviewing Hermes Agent architecture, internals, config, tools, profiles, memory, MCP, gateway, cron, plugins, skills, provider routing, context compression, or security behavior. Treat official Hermes docs and the NousResearch/hermes-agent repo as authoritative sources; this skill is an offline architecture reference that must be kept in sync.
version: 1.8.0
last_drift_review: 2026-08-09
repo_commit: cdcbc3a31 (last fully-synced anchor; upstream now f28469349, +6986 commits pending; SSOT untouched per skill protocol)
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
- `references/CHANGELOG-2026-07-12.md` — weekly drift review (214 commits since `cdcbc3a31`). Major additions: state.db schema v19 (`gateway_routing` table, sessions.json demoted to legacy mirror), `approvals.deny` (pre-yolo deny list), `secrets` subsystem (pluggable `SecretSource` + 1Password `op://`), webhook body-size limits on 5 platforms, ~80% TTFT cut (TTFT round 2), Codex gpt-5.4 autoraise, durable one-shot cron run-claim. Reverts: pre-tool-call approval gate (#58698), `dynamic-workflow` skill. **Read this before assuming any section of the main reference is current.**
- `references/drift-review-runbook.md` — mechanical execution details for the weekly drift review: anchor probes, commit-type concentration command, feat-only inventory, canonical architecture area buckets (1:1 with Hermes_Architecture.md sections), spot-verification commands, frontmatter bump recipe, public-mirror sync.
- `references/neo4j-graphiti-bolt-pool.md` — Neo4j 2026.05.0 (Homebrew) bolt pool tuning, Graphiti MCP keepalive / CancelledError root cause, verified config keys, and `brew services stop && start` recovery for launchctl bootstrap I/O error 5. Lessons from 2026-07-08 session where a third-party skill doc had the wrong config key.
- `references/CHANGELOG-2026-07-19.md` — weekly drift review (502 commits since `aaeba213d`). Major additions: smart approvals now default (`approvals.mode: smart`), gpt-5.6 family (6 slugs, 272K compaction auto-raise), Fireworks AI as preferred provider, grok-4.5 GA, reasoning effort levels `max`/`ultra`, Hermes Cloud connection mode in desktop, OIDC client-credentials relay, webhook payload filters, PTY keep-alive/reattach/session-registry, Tencent Hy3 GA swap. **Read this before assuming any section of the main reference is current.**
- `references/CHANGELOG-2026-07-26.md` — weekly drift review (1,074 commits since `7b5ba2054`). Major additions: **schema v22** (`session_model_usage` PK dimension + table rebuild for per-task auxiliary accounting), **`api_content` sidecar** for exact-prompt-cache replay (27.9s → 2.4-5.8s first-call TTFT fix), **durable delivery-obligation ledger** (gateway final-response loss fix), **per-session turn lease** (closes concurrent-turn alternation wedge), **profile-based routing for inbound messages** (Discord guilds/channels/threads → profiles), **`hermes config get`/`unset`**, **unknown top-level key warnings + `hermes doctor` deprecated-key/env reporting**, **per-model reasoning_effort overrides**, **per-task auxiliary reasoning_effort**, **MoA per-slot reasoning_effort**, **`/model --once` one-turn override**, **`/topup` + `/subscription` Nous plan UX**, **MCP exact version pins** + Blender catalog entry + Unreal MCP live-verify, **Upstage Solar** + **DeepInfra** providers, **unified 3-store credentials** (`.env` + `auth.json` + `config.yaml`), **Blender companion skill** + **MCP-OAuth-remote-gateway skill**, **computer-use verify → escalate ladder** with `delivery_mode` foreground/background, **Honcho config schema** + `surface=declared` routing + `honcho_host_block` storage, **kanban attachments toolset + CLI** + `kanban_attach_url` SSRF guard + modal create-task dialog + board `default_workdir` settings, **cron truthful execution ledger**, **profile-aware approval mode control**, **inline `/reasoning`/`/fast` pickers** on Telegram/Discord/Matrix, **model-picker reverts-in-existing-threads fix**. Reverts: provider-actions extension point (memory); E2E manual-approval test force. **Read this before assuming any section of the main reference is current.**
- `references/CHANGELOG-2026-08-09.md` — biweekly drift review (5,196 commits since `65b73eb1e`; releases v0.19.0 → v0.20.0). Major additions: **schema v23–v25** + SessionDB mixin split, **`agent.max_turns` default 90→500**, **tiered tool_search disclosure (0/1/2)** + per-server mixed listings, **micro-compaction** + **native gpt-5.6 Responses server-side compaction**, **MCP schema cache + lazy start + trust tiers (`readOnlyHint`)**, **ESTOP `hermes pause`/`resume`**, **protected instruction-file writes**, **self-repo git mutation hard-block**, **cron notepad + monitor-mode + preflight `blocked_config`**, **outbound webhooks**, **A2A platform plugin**, **portable Agent Plugins v1** (incl. streamable-http), **delegation `output_schema`**, **`/refine` `/heartbeat` `/goal gate` `/init` `!` shell**, **`hermes import-agent`**, **`hermes doctor --live` + state.db health stats**, **kanban live comment steer + per-task model/reasoning pins**, **reset-aware primary restore**, **Actual Computer provider**, **wake word + client-mic streaming**, clean-room MIT office skills + advisory skill linter. Reverts: **DCP context engine**, **NeMo Relay shared metrics**. Full SSOT now **+6,986 commits / ~7 weeks stale — re-compile overdue**. **Read this before assuming any section of the main reference is current.**

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

### Weekly review checklist (canonical, post-2026-07-12)

1. **Establish anchors.** Record four facts up front — they go in the changelog header verbatim:
   - Local Hermes commit (`hermes --version` + `git -C ~/.hermes/hermes-agent rev-parse HEAD`)
   - Upstream HEAD (`git rev-parse origin/main` after `git fetch origin main`)
   - Local-behind-upstream commit count (`git rev-list --count HEAD..origin/main`) — **non-zero is normal**; the local install can trail upstream by weeks. Don't conflate "I haven't pulled" with "I haven't reviewed."
   - Prior SSOT anchor (the `repo_commit` line in this SKILL.md frontmatter)
2. **List the delta.** `git log --oneline <prior_anchor>..<upstream>` — note the total count and rough feat/fix/refactor breakdown.
3. **Bucket the delta by architecture area, NOT by commit.** Canonical area list (matches the section structure of `references/Hermes_Architecture.md` so the next re-compile has a 1:1 merge map):
   - Configuration & provider routing
   - Schema versions & state.db migrations
   - Approvals & security
   - Skills (format, disabled-gate, preload)
   - MCP (auth, parked-revival, preflight)
   - Sessions & state
   - Compression / context (TTFT, autoraise, native paths)
   - CLI (new commands, removed commands, behavior changes)
   - Cron (run-claim, scheduling, past-timestamp rejection)
   - Gateway (routing, pairing, session-key guards, multiplex)
   - Platform adapters — Discord, Telegram, WhatsApp, Feishu, MS Graph, SMS, Mattermost, Slack (per-platform entries)
   - Computer-use (cua-driver, env sanitization)
   - Providers / Auth (OAuth, Z.AI, Codex, Nous, Anthropic, OpenRouter, etc.)
   - Web / search providers (Firecrawl, exa, parallel, tavily, brave-free)
   - Docker / Photon
   - Desktop / TUI
   - Secrets (SecretSource plugin system, Bitwarden, 1Password)
   - Reverts (worth a dedicated section — call them out by PR number)
   - Salvaged PRs (cherry-picks preserving authorship; mention `AUTHOR_MAP` updates)
4. **Spot-verify high-impact commits before quoting.** For each `feat(...)`, `refactor(...)`, or revert, run `git show --stat <sha>` and confirm the body matches the commit message. Drift reviews fail when the changelog claims things the commit doesn't actually do.
5. **Decide: re-compile OR changelog-as-scaffolding.** Use the decision tree below.
6. **Write the dated changelog** (`references/CHANGELOG-YYYY-MM-DD.md`) — even for sub-threshold windows. The chain of changelogs is the audit trail and the scaffold the next re-compile consumes. See the protocol subsection below for the 8-step format.
7. **Bump the SKILL.md frontmatter** with the exact 4 mechanical edits:
   - `version` (semver minor bump on every drift review)
   - `last_drift_review: YYYY-MM-DD`
   - `repo_commit` — format as `<anchor> (last fully-synced anchor; upstream now <head>, +<n> commits pending; SSOT untouched per skill protocol)` when changelog-only, or `<head>` when full re-compile lands.
   - Add a pointer under "Targeted quirks" to the new changelog with the explicit "Read this before assuming any section of the main reference is current" warning.
8. **Do NOT edit `references/Hermes_Architecture.md` unless doing a full re-compile** (see the pitfall — partial edits leave the reference looking current but inconsistent).

### Re-compile vs changelog-as-scaffolding — decision tree

| Condition | Action |
|---|---|
| Prior SSOT anchor < 1 week old AND delta < 50 commits | Inline report only — no changelog file needed. Patch `last_drift_review` + log a session_search note. |
| Delta 50-500 commits in a single window (typical weekly) | **Changelog-as-scaffolding.** Write `CHANGELOG-<date>.md`. Do NOT touch `Hermes_Architecture.md`. |
| Delta > 500 commits OR prior anchor > 4 weeks old OR operator explicitly requests | **Full re-compile.** Re-author `references/Hermes_Architecture.md` from upstream, then write a changelog describing the new baseline. |
| Operator says "lightweight local model only" or cron has extended time | Follow operator's directive; default to changelog-as-scaffolding unless explicitly told to re-compile. |

**Ambiguous zone clarification (post-2026-07-12):** when the prior text said "default to changelog-as-scaffolding if >500 commits or >4 weeks" AND "full re-compile only if (a) operator requests, (b) <300 commits, or (c) extended time," there was an overlapping range where both rules applied differently. The decision tree above resolves it: **for weekly cadences with 50-500 commits, always changelog-as-scaffolding**, regardless of operator hints about re-compile scope, because the SSOT was just refreshed in the most recent full re-compile (1,098 commits on 2026-07-05) and incremental churn doesn't justify re-doing it.

### Large drift window — changelog-as-scaffolding (verified 2026-07-05, 1098-commit pass)

When the drift window between the prior SSOT anchor and the current upstream HEAD exceeds what a single cron pass can re-compile (~300+ commits, multi-week windows, or "lightweight local model only" directives), do NOT attempt a full re-compile. Use the **changelog-as-scaffolding** pattern instead:

1. **Anchor bounds.** Record both anchors explicitly: `git log --oneline <prior>..<current>` and the commit count. State the active Hermes version + commit + upstream HEAD.
2. **Group deltas by area**, not by commit. Areas: config & provider routing, skills/CLI, delegation, MoA, gateway, compression/caching, memory, MCP, tools, webhooks, cron, security, web tools, models, desktop, platform adapters (WhatsApp/Telegram/Discord/Slack), provider routing/runtime. This matches the section structure of `Hermes_Architecture.md` so the next re-compile has a 1:1 merge map.
3. **Attach source SHAs** to every delta. `b98baa303` — short SHA, descriptive label. The next re-compile uses these to pull `git show <sha>` for verification, not to trust your summary.
4. **End with a "Action items for next full reference re-compile" list** — concrete per-section edits to make. This is the actual scaffold the next pass consumes.
5. **Document known limitations** of the pass (e.g. docs not fetched, local Hermes itself behind upstream) so the next pass knows what gaps remain.
6. **Bump skill frontmatter**: `version`, `last_drift_review`, `repo_commit`. Add the changelog under "Targeted quirks" with an explicit "read this before assuming any section of the main reference is current" warning.
7. **Mirror to public repo** (`shagghiesuperstar/hermes-master-ref-maintained`). The public mirror is distribution for other agents — push the same changelog so they see the drift markers without re-deriving them.
8. **Name the file `CHANGELOG-<YYYY-MM-DD>.md`** so a sequence of passes forms a chain. Never overwrite a prior changelog — the chain is the audit trail.

Threshold heuristic: if the prior SSOT anchor is more than 4 weeks old, or the delta exceeds 500 commits, default to changelog-as-scaffolding. A full re-compile is only worth attempting in a single pass if (a) the operator explicitly requests it, (b) the drift is <300 commits, or (c) the cron has explicit extended time.

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
- **Local Hermes can be behind upstream.** Always probe `git rev-list --count HEAD..origin/main` at the top of a drift review. A non-zero count is normal (the local branch is often pinned to a feature branch); don't confuse "local is stale" with "drift is zero." The drift count that matters is `<prior_anchor>..<upstream>`, not `<local>..<upstream>`.
- **Spot-verify before quoting.** Never put a commit's claims into the changelog without running `git show --stat <sha>` first. Commit messages can mislead (reverts, salvage merges, follow-up fixes). The 30-second spot-check is what catches "this commit doesn't actually do what the message says."
- **Threshold heuristic contradictions.** Earlier versions of this skill had overlapping re-compile vs changelog conditions that conflicted for the 50-500 commit weekly range. Follow the decision tree above, not the legacy prose.
- **When a drift window is too large to re-compile in a single pass, write a dated changelog reference (`CHANGELOG-<YYYY-MM-DD>.md`) grouped by architecture area with source SHAs, NOT a free-form report or a partial reference edit.** A partial edit is worse than no edit — it leaves the reference in a state that looks current but isn't. The changelog is the durable scaffolding for the next full re-compile; the reference stays untouched until a full re-compile consumes the scaffold. See the "Large drift window — changelog-as-scaffolding" subsection in the Drift review protocol above for the full pattern.

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
