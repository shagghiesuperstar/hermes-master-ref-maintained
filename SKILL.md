---
name: hermes-architecture-reference
description: Use when troubleshooting, designing, implementing, optimizing, maintaining, or reviewing Hermes Agent architecture, internals, config, tools, profiles, memory, MCP, gateway, cron, plugins, skills, provider routing, context compression, or security behavior. Treat official Hermes docs and the NousResearch/hermes-agent repo as authoritative sources; this skill is an offline architecture reference that must be kept in sync.
version: 2.0.0
last_drift_review: 2026-11-06
repo_commit: 9f23c4311a4c4837078081db4f3cd8deab23c0fe
author: Hermes Architecture Reference maintainers
license: MIT
metadata:
  hermes:
    tags: [hermes, architecture, troubleshooting, maintenance, optimization, implementation, internals, mcp, gateway, memory, skills, cron]
    related_skills: [hermes-agent]
---

# Hermes Architecture Reference

## Purpose

This skill is the local/offline architecture reference for Hermes Agent. Load it for any Hermes Agent troubleshooting, architecture decision, implementation, optimization, maintenance, or drift-review work.

Authoritative sources:

1. Official documentation: <https://hermes-agent.nousresearch.com/docs/>
2. Official repository: <https://github.com/NousResearch/hermes-agent>

The main reference ([Hermes_Architecture.md](./Hermes_Architecture.md)) was rebuilt on 2026-11-06 from those two sources, anchored to upstream commit `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (latest release at that time: [v0.23.0](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0), published 2026-11-05). **Live docs and the live repository win if they differ.** Community sources (upstream issues and PRs, HermesHub at <https://hermeshub.com>) are secondary: they may be cited for context but must be labelled as community/secondary, never presented as official behavior. See [README.md](./README.md) for the repository overview.

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

- [Hermes_Architecture.md](./Hermes_Architecture.md)

Use targeted reads/searches instead of loading the full file into the chat when possible. It is large.

## Companion files

All files live in the repository root (flat layout).

- [README.md](./README.md) — repository purpose, reading order, update cadence, contribution and privacy rules, source authority.
- [drift-review-runbook.md](./drift-review-runbook.md) — mechanical execution details for the periodic drift review: anchor probes, commit-type concentration, feat-only inventory, architecture area buckets, spot-verification commands, frontmatter bump recipe.
- [approvals-mode-quirks.md](./approvals-mode-quirks.md) — `approvals.mode: off` write/repair playbook (covers `hermes config set` YAML 1.1 coercion, `config get` behavior, `config show` not displaying `approvals:`, and file-tool guards that block direct writes; verified 2026-06-28, so re-check against current docs).
- [citation-verification.md](./citation-verification.md) — protocol for verifying user-supplied or shared procedures and docstrings before executing them.
- [config-migration-and-provider-routing.md](./config-migration-and-provider-routing.md) — provider migration playbook: structured (non-`sed`) edits, `model.provider` / `model.default` namespace-collision pitfall, live OpenRouter slug validation, stale-provider purge, worker-profile creation, routing-role vs runnable-profile distinction, and a pre-completion verification checklist.

## Changelog index (historical snapshots)

Each dated changelog below is a **historical snapshot** of one drift-review pass. It records what the reviewer observed against the upstream head of that day. It is not current truth, and any staleness or "commits behind" figure inside it describes that day only. Use the changelogs for provenance and for finding the commit SHAs behind a change; use [Hermes_Architecture.md](./Hermes_Architecture.md), the official docs, and the upstream repo for current behavior. The 2026-11-06 entry describes the rebuild that produced the current reference.

- [CHANGELOG-2026-11-06.md](./CHANGELOG-2026-11-06.md) — rebuild of the main reference against official docs and upstream `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (v0.23.0), privacy scrub, changelog date corrections, new README.
- [CHANGELOG-2026-11-01.md](./CHANGELOG-2026-11-01.md) — snapshot: remote config plane (`HERMES_CONFIG_BACKEND=remote` and the `ConfigBackend` seam), GitHub Copilot moved out of core into a labs plugin, `hermes cron cancel`, `sessions recover`, `webapp`, `hooks`, `pre_llm_call` able to block a turn, `/model reset`, `compression.prepare_ahead`, per-job cron timezone and missed-run policy; corrects `SCHEMA_VERSION` figures in earlier snapshots.
- [CHANGELOG-2026-10-25.md](./CHANGELOG-2026-10-25.md) — snapshot (originally filed under a mis-dated name, corrected 2026-11-06): gateway hardening, webapp server from the gateway, desktop plugin SDK expansion, vault OS-keychain opt-in, plugin catalog growth. Its schema-version statement was corrected by the 2026-11-01 snapshot.
- [CHANGELOG-2026-10-18.md](./CHANGELOG-2026-10-18.md) — snapshot (originally filed under a mis-dated name, corrected 2026-11-06): plugin system maturation (new hooks, public API stabilization), desktop onboarding i18n, gateway route-bound model keys, plugin catalog growth, local-model onboarding UX, NVIDIA NIM defaults. Its schema-version statement was corrected by the 2026-11-01 snapshot.
- [CHANGELOG-2026-10-14.md](./CHANGELOG-2026-10-14.md) — snapshot: prior summary retained for chain integrity.
- [CHANGELOG-2026-10-04.md](./CHANGELOG-2026-10-04.md) — snapshot: schema v30 to v31 (`messages.message_uid` and sibling identity columns), gpt-6.1 family, `plugins.isolation: host`, `'external'` skill provenance tier, layered i18n language packs, per-job cron `interpreter`, `authserv_id` requirement for authenticated-sender email.
- [CHANGELOG-2026-09-27.md](./CHANGELOG-2026-09-27.md) — snapshot: desktop plugin SDK expansion, release pipeline rework, PM subsystem, plugin catalog growth, Docker image axis `[slim, desktop]`, agent streaming fixes, email subsystem rewrite, transcript persistence rewrite.
- [CHANGELOG-2026-09-20.md](./CHANGELOG-2026-09-20.md) — snapshot: model-provider plugin system maturity, managed local llama.cpp runtime, cron reliability suite, voice/STT/TTS changes, security fixes, profile and clone changes, gateway profile routing.
- [CHANGELOG-2026-09-13.md](./CHANGELOG-2026-09-13.md) — snapshot (v0.21.0 to v0.21.2): multiplex gateway serving every live profile (breaking), `hermes gateway migrate`, IMDS endpoint approval, OpenRouter OAuth PKCE login, cron bounded re-runs, HEIF/HEIC/AVIF vision decode, full-duplex voice mode, Honcho peer-model rework.
- [CHANGELOG-2026-09-06.md](./CHANGELOG-2026-09-06.md) — snapshot (v0.21.0): schema 26 to 30, managed llama.cpp runtime, OpenRouter per-model `provider_routing`, core-tool deferral, delegation completion groups, cron doctor, opt-in shared-metrics telemetry, new model catalogs, Perplexity and Tavily web backends.
- [CHANGELOG-2026-08-30.md](./CHANGELOG-2026-08-30.md) — snapshot (v0.20.6): lean tail retention default, prompt and tool-schema rebuild at compaction boundary, shipped-skills slimming, `skill_manage.operations[]`, cross-profile write guard retired, cron durable incidents, gateway pause-for-update socket.
- [CHANGELOG-2026-08-23.md](./CHANGELOG-2026-08-23.md) — snapshot (v0.20.2 to v0.20.5): `agent.max_turns` unlimited default, keyless-auth policy, MCP SDK 2.x, gateway control socket, Bot Mode `message_agent`, update receipts and `--plan`, browser control broker, `plugin_guard` on install.
- [CHANGELOG-2026-08-16.md](./CHANGELOG-2026-08-16.md) — snapshot (v0.20.1): schema v26, compression watermark commit and lean `tail_mode`, `model_overrides`, `key_cmd` credential source, delegation defaults, `/loop`, plugin manifest v2, multi-connection desktop registry, `pre_tool_call` `modify` hooks.
- [CHANGELOG-2026-08-09.md](./CHANGELOG-2026-08-09.md) — snapshot (v0.19.0 to v0.20.0): schema v23 to v25, tiered `tool_search` disclosure, micro-compaction, MCP schema cache and trust tiers, `hermes pause`/`resume`, cron notepad and monitor mode, outbound webhooks, A2A platform plugin, portable Agent Plugins v1, `hermes import-agent`, `hermes doctor --live`.
- [CHANGELOG-2026-07-26.md](./CHANGELOG-2026-07-26.md) — snapshot: schema v22, `api_content` sidecar for prompt-cache replay, durable delivery-obligation ledger, per-session turn lease, profile-based routing for inbound messages, `hermes config get`/`unset`, unified credential stores, kanban attachments, cron execution ledger.
- [CHANGELOG-2026-07-19.md](./CHANGELOG-2026-07-19.md) — snapshot: smart approvals default, gpt-5.6 family, Fireworks AI provider, reasoning effort levels `max`/`ultra`, OIDC client-credentials relay, webhook payload filters, PTY keep-alive and reattach.
- [CHANGELOG-2026-07-12.md](./CHANGELOG-2026-07-12.md) — snapshot: state.db schema v19 (`gateway_routing` table), `approvals.deny`, pluggable `secrets` subsystem, webhook body-size limits, TTFT reductions, durable one-shot cron run-claim; two reverts.
- [CHANGELOG-2026-07-05.md](./CHANGELOG-2026-07-05.md) — snapshot (v0.17.0 to v0.18.0): per-provider `extra_headers`, persisted per-session `/model`, MoA `fanout: user_turn`, stacked `/skill` invocations, MCP GUI tab, cron profile-secret scope.

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

Check the table of contents of [Hermes_Architecture.md](./Hermes_Architecture.md) for the exact current section list.

## Drift review protocol

The maintainer reviews this skill after each upstream release and at least weekly, and updates it if it drifts from the authoritative sources. The mechanical commands live in [drift-review-runbook.md](./drift-review-runbook.md).

### Review checklist

1. **Establish anchors.** Record these facts up front; they go in the changelog header verbatim:
   - Local Hermes commit (`hermes --version` plus `git -C <hermes-agent-checkout> rev-parse HEAD`)
   - Upstream HEAD (`git rev-parse origin/main` after `git fetch origin main`), plus the latest release tag
   - Local-behind-upstream commit count (`git rev-list --count HEAD..origin/main`). A non-zero count is normal; a local install can trail upstream. Do not conflate "I have not pulled" with "I have not reviewed."
   - Prior reference anchor (the `repo_commit` line in this file's frontmatter)
2. **List the delta.** `git log --oneline <prior_anchor>..<upstream>`; note the total and a rough feat/fix/refactor breakdown.
3. **Bucket the delta by architecture area, not by commit**, using the area list in the runbook (it mirrors the section structure of [Hermes_Architecture.md](./Hermes_Architecture.md) so the next merge has a 1:1 map). Call out reverts by PR number.
4. **Spot-verify high-impact commits before quoting.** For each `feat(...)`, `refactor(...)`, or revert, run `git show --stat <sha>` and confirm the body matches the message.
5. **Decide: rebuild or changelog-only.** Use the decision table below.
6. **Write the dated changelog** (`CHANGELOG-YYYY-MM-DD.md` in the repository root) even for small windows. The chain of changelogs is the audit trail. Never overwrite a prior changelog, and never date a note in the future.
7. **Bump the frontmatter** of this file with the mechanical edits:
   - `version` (semver minor bump on every review; major bump on a full rebuild)
   - `last_drift_review: YYYY-MM-DD`
   - `repo_commit`: the full 40-character upstream SHA the reference was verified against. If a pass was changelog-only, keep the last fully synced SHA here and describe the pending delta in the changelog, not in the frontmatter value.
   - Add the new changelog to the index above, labelled as a historical snapshot.
8. **Do not partially edit [Hermes_Architecture.md](./Hermes_Architecture.md)** unless doing a full rebuild; partial edits leave the reference looking current but inconsistent.

### Rebuild vs changelog-only decision table

| Condition | Action |
|---|---|
| Prior anchor under one week old and a very small delta | Inline report only; patch `last_drift_review`. |
| Moderate delta in a single window | Changelog-only: write `CHANGELOG-<date>.md`, leave the main reference untouched. |
| Large delta, a long-unreviewed anchor, a new upstream release with architectural changes, or an explicit maintainer request | Full rebuild of [Hermes_Architecture.md](./Hermes_Architecture.md) from the official docs and upstream source, then a changelog describing the new baseline. |

### Changelog-only pattern for a large window

When a window is too large to rebuild in one pass, write a dated changelog instead of a partial edit:

1. Record both anchors and the commit count.
2. Group deltas by architecture area, not by commit.
3. Attach short SHAs to every delta so the next rebuild can run `git show <sha>` instead of trusting the summary.
4. End with an "Action items for next full rebuild" list of concrete per-section edits.
5. Document known limitations of the pass (docs not fetched, local install behind upstream, nothing executed).
6. Bump the frontmatter as above and add the changelog to the index with a historical-snapshot label.
7. Name the file `CHANGELOG-<YYYY-MM-DD>.md` so passes form a chain.

## Maintenance commands

Suggested local probes:

```bash
hermes --version
hermes config path
hermes config check
hermes tools list
hermes mcp list
python3 -m py_compile <hermes-agent-checkout>/run_agent.py
```

Suggested docs/repo checks:

```bash
git -C <hermes-agent-checkout> fetch origin main
git -C <hermes-agent-checkout> log --oneline -20 origin/main
```

Locate the checkout before running repo commands; its path differs between machines.

## Pitfalls

- This skill is not a replacement for official docs. It is an offline reference.
- Hermes profile isolation matters. Always run `hermes config path` before claiming which config is active.
- Config edits must use official Hermes CLI commands unless the operator explicitly authorizes direct file edits.
- **Config may not be a local file.** Per the 2026-11-01 snapshot ([CHANGELOG-2026-11-01.md](./CHANGELOG-2026-11-01.md)), upstream added a `ConfigBackend` seam: with `HERMES_CONFIG_BACKEND=remote` each profile's config is served from a config plane, `config.yaml` is neither read nor written, and boot fetch fails closed. Several commands (`hermes config edit`, profile clone/rename, backup/import, snapshot restore) are refused in that mode and `hermes config set` is the supported path. Confirm current behavior in the official docs, and check the active backend (`hermes doctor`) before applying any `config.yaml`-based procedure.
- **GitHub Copilot may not be a core provider.** Per the same snapshot it ships as the `github-copilot` labs plugin (`hermes plugins install github-copilot`). Verify against the current provider docs before relying on `copilot` routing.
- Avoid hardcoded user paths when writing reusable procedures. Use `hermes config path`, `hermes config env-path`, and profile-aware locations.
- If a review finds drift, patch the skill promptly rather than only reporting it.
- **Local Hermes can be behind upstream.** Probe `git rev-list --count HEAD..origin/main` at the top of a review. The drift count that matters is `<prior_anchor>..<upstream>`, not `<local>..<upstream>`.
- **Spot-verify before quoting.** Never put a commit's claims into a changelog without running `git show --stat <sha>` first. Commit messages can mislead (reverts, salvage merges, follow-up fixes).
- **Changelogs are snapshots.** Any figure in an archived changelog (commit counts, schema numbers, "N commits behind", staleness) is true only for its own date. Re-derive current figures from the docs and upstream source.
- **Never date a changelog after the day it was written.** Dates are part of the audit trail.

### Config-migration workflow pitfalls

- Prefer structured edits (Python with `pathlib`/`yaml`/`re`) or the official `hermes config` CLI over ad-hoc shell text tools when editing config files.
- When given a numbered sequence of steps to run end to end, execute the full sequence and verify each step. Do not declare completion without readback (`hermes config check`, `hermes moa list`, active-model readback, file diff).
- File-write tools may refuse to edit `config.yaml` (security guard). Fall back to a structured Python edit or the CLI.
- `model.default` must be a real model slug, never the literal `"default"`; `model.provider` should be a real provider such as `openrouter` (not `moa`) for routine chat, since `provider: moa` routes every turn through the council at several times the cost.
- Validate every OpenRouter slug against the live catalog (`https://openrouter.ai/api/v1/models`) before writing it to config or a routing doc. The `:nitro` suffix is a routing modifier on a real base slug.
- A routing role in a routing doc is not a runnable profile. Only directories with a `config.yaml` are real profiles; empty stubs are labels. Never report "created a profile" when only a doc row was added.
- When cloning a profile for a worker, rewrite the inherited `SOUL.md` so it drops the orchestrator identity; `--no-skills` is mutually exclusive with `--clone-from`.
- See [config-migration-and-provider-routing.md](./config-migration-and-provider-routing.md) for the full checklist, code snippets, and examples.

## Verification

After creating or updating this skill:

```bash
hermes skills list | grep hermes-architecture-reference
```

Then load the skill and confirm the reference file appears under linked files.
