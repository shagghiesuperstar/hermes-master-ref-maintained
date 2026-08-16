# Hermes Architecture Reference — Drift Changelog 2026-08-16

## Scope of this review

- **Local Hermes version:** v0.20.0 (2026.8.3) — local `eac1e2512` (on `main`, **1,267 commits behind** `origin/main`)
- **Upstream HEAD reviewed:** `7095e23eb` (origin/main as of 2026-08-16)
- **Prior SSOT anchor (last drift review upstream HEAD):** `f28469349` (CHANGELOG-2026-08-09)
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **1,643** (~188 feat, ~983 fix, rest chore/test/refactor/docs/style; **2** reverts)
- **Window:** 2026-08-09 → 2026-08-16 (~1 week)
- **Releases in window:** `v0.20.1` (2026.8.13)
- **Full SSOT drift (`cdcbc3a31` → origin/main):** **8,629** commits pending
- **Methodology:** lightweight-local model (per operator/cron directive) + git-log area-bucketing + spot-verification (`git show --stat` + body) on ~30 high-impact commits. Official docs spot-check (configuration + MoA pages). **Changelog-as-scaffolding** per the drift review decision tree (1,643 commits ≫ 500; operator said "lightweight local model only"). The main reference (`Hermes_Architecture.md`) is left untouched.

> **Why a changelog and not a full re-compile.** Delta is above the 500-commit re-compile threshold, the full SSOT is now ~8 weeks / ~8.6k commits stale, and the cron directive is lightweight-local only. This pass continues the changelog chain (07-05 → 07-12 → 07-19 → 07-26 → 08-09 → **08-16**). **A full re-compile remains overdue** and should be authorized on the next non-lightweight pass.

## High-impact architecture deltas (`f28469349` → `7095e23eb`)

### Schema versions & state.db migrations

- **SCHEMA_VERSION advanced 25 → 26.** `e89532d97` — sessions table gains `git_metadata_generation INTEGER NOT NULL DEFAULT 0` (async desktop git-metadata ordering). FTS layout versioning remains independent of `SCHEMA_VERSION`.
- **Compression watermark commit path.** `21d3e6370` (+ follow-ups `652f5c2eb`, `406c5daf0`) — redesign of the #75316 class:
  - Ordinary transcript appends no longer check `compression_locks` (lock only serializes concurrent compressions).
  - Watermark = `MAX(id)` of active rows at compression start (DB, not in-memory dicts).
  - `archive_and_compact(watermark=, lock_holder=)` re-sequences concurrent tail (`id > watermark`) via pure-SQL column clone (byte-exact sidecars including `api_content`).
  - Removes append-side compression fence + busy-wait family of failures (`session_persistence_failed` during slow summaries).
- **state.db repair loop bounded.** `0d00ebef7` (#86747) — stop dead-backup accumulation.
- **Schema surgery serialized across processes.** `923d86e09`.
- **Eager schema migration at backend startup** + stop swallowing locked ALTERs. `f45813ea7`.
- **Canonical writes available when FTS is corrupt.** `1527a81b5`.
- **Generic session `hidden` flag.** `fbaea9bdd` (#86797) — sidebar-hide, still resumable.

### Configuration & provider routing

- **`compression.tail_mode` config surface.** `1d5fc2bc0` (#87326 wiring) — `legacy` (default, 0.20×window verbatim tail) | `lean` (clamped 2.5%-of-window tail, 10K–25K). Docs en/zh.
- **`model_overrides` unified metadata overrides.** `dafdba324` (+ simplify/fill-gap follow-ups `67ab2f296`, `de47d19f1`):
  - Resolution: `model_overrides.<provider>.<model_id>` → `.<provider>._default` → `._default` → catalog.
  - Fields: `context_window`, `max_output_tokens`, capabilities, cost, family.
  - Unknown model ids get safe defaults then patch (self-unblock for custom/local / wrong-catalog cases).
- **`providers.<name>.key_cmd` credential source.** `6efab2872` (+ bound no-TTL cache `4ef56cef4`):
  - Command prints a short-lived bearer; wrapped as zero-arg callable invoked per request.
  - Cached until shortly before advertised expiry (`expires_in` or absolute ISO deadline); distinct from one-shot `secrets.command`.
  - Covers enterprise SSO/OIDC/IAM gateways without rewriting `.env` mid-session.
- **Unified model selection-time guards registry.** `916653094` — `hermes_cli/model_selection_guards.py` centralizes cost + data-training-tier warnings across CLI/TUI/gateway/dashboard/Telegram/Discord pickers.
- **Data-training tier warn at model selection.** `a06f1d761` (fires via the unified registry).
- **Delegation defaults raised (with migrations):**
  - `delegation.max_iterations` 50 → **250** (`50d98fc1f`, `_config_version` 35→36; migration lifts pinned-at-50 only).
  - `delegation.max_concurrent_children` 3 → **10** (`ce996d405`, `_config_version` 36→37; migration lifts pinned-at-3 only).
- **Delegation truncation marker.** `dc2fe99ec` (#86641) — max_iterations-truncated subagent results flagged for parent.
- **Profile-scoped provider auto-detection keys.** `8ff5f13c0` / `275f8d41b` (#86917/#86918).
- **Custom provider `default_model` when `--provider` set.** `d6688adcc` / `3cbd86aac`.
- **Sequential tool deadline via `timeouts.tools.sequential_call`.** `367f0c21e` (#85125).
- **Docs-confirmed (live site):** `hermes config get`/`unset` present; provider `request_timeout_seconds` / `stale_timeout_seconds`; Cursor-style `${env:VAR}` SecretRef; `runtime.nofile_soft_limit`; terminal backends include `vercel_sandbox`; update knobs under `updates.*`.

### Compression / context

- **Lean compaction mode (major).** `8fe9025ab`:
  - `tail_mode='lean'`: tail budget = clamp(2.5% of window, 10K, 25K); stale tail tool results → session_search recovery stubs; chunked identifier-preserving digests; verbatim user messages in summary; deterministic session_search recovery footer.
  - Eval arm gains `+recovery` (one simulated session_search round-trip).
- **Mechanical anchor index + region-scoping tripwire.** `c4bbb14e5` — regex-harvests PR/issue/SHA/path/error/URL anchors from compacted region (LLM-free); eval proves summarizer input is region-only.
- **Digest noise filter + FTS5 recovery sim + digest-aware query hints.** `7a82457ed`.
- **Field-proven summarizer prompt upgrades.** `31ca1200e` — anti-injection preamble, verbatim security-constraint preservation, Errors & Fixes with user-correction quoting.
- **Compaction recall eval harness.** `33242d5ee`.
- **Watermark commit** — see Schema section (architecture-level fix for concurrent-append-during-compaction).
- **Hygiene idle timeouts must not block in-agent compression.** `2cb838196`.

### Tools / hooks / computer-use

- **`pre_tool_call` content transformation via `modify` directive.** `d083b8559` (salvaged #28953):
  - Hooks may return `{action: modify, args: {...}}` (or Claude Code-compatible `{decision: modify, tool_input: {...}}`); shallow-merge into original args before execution.
  - Dispatch sites: `model_tools.py`, `tool_executor.py`, `agent_runtime_helpers.py`.
- **cua-driver browser attachment authorization.** `48dd9c87c` (+ contract align `20cf326bd` for cua-driver 0.19.3):
  - `hermes computer-use browser-approve` mints five-minute single-use attachment token (user is token source, never the model).
  - `approval_token` on `cua_browser_prepare` for `existing_profile` only.
  - `computer_use.permission_mode: bounded` + capability_manifest; `unrestricted` deliberately not a config value.
- **Computer-use hardenings:** wrong-window refuse on app mismatch (`b7f628025`); zero-display capture doctor flag (`460d34564`); merge refs+content_refs for cua-driver 0.17 (`bbd3462e2`).
- **Named `browser_exec` sessions compose with every backend.** `bb4f680f2` / pin-to-own-tab `c9a806e9d`.

### MCP

- **Per-profile MCP server lifecycle RPCs.** `d275b96bf` (#86473) — `mcp.servers.list/add/set_api_key/test/remove/oauth.start/poll` via profile-scoped `set_hermes_home_override` (mirrors `skills.manage`).
- **`profiles.create share_auth` + MCP in describe/configure.** `4bbc6f258` — share auth.json via global-root fallback (single refresh pool); `enabled_mcp_servers` replace semantics on configure.
- **Desktop MCP fleet cost/usage overlay** (schema token estimates + 30-day usage). `e22fa9076`.
- **`hermes://` deep link MCP install with explicit confirmation.** `56eafcff3`.
- **Background MCP health checks + re-auth nudges.** `1c3fbd21a`.
- **Paste-anything MCP server import (desktop).** `1e54e0522`.
- **Unify MCP Servers + Catalog into one list.** `37445d6dc` / suggestion directory into catalog `fc8ebff6d`.
- **MCP suggestion directory → 18 official hosted remotes.** `8c8d55bd0`.
- **Prefer server-native tool over generated utility on name collision.** `d6f18cd7d` (#87112).
- **DCR clients with secrets across all OAuth paths.** `9975180a5`.
- **SDK export of route-decoupled `McpTab` + `ToolsetConfigPanel`.** `3671529e9` (#86896).

### Approvals & security

- **Deterministic `approvals.single_query_mode` for `-q` sessions.** `1596148ff`.
- **Linear webhook HMAC auth via `linear-signature`.** `f13f3401a` (#87348).
- (Prior ESTOP / protected-instruction / self-repo-block surface from 08-09 remains; no further ESTOP redesign this window.)

### Skills

- **Desktop Skills tab hub browser + full-skill detail pane** (drop Browse Hub tab; SkillsView SDK export). `d5773bfc3`.
- **Capabilities-wide profile scoping + one-click hub installs.** `9c58a78a7`.
- **Bundled Box productivity skill.** `e450b09dd`.

### Sessions & state

- **Import and resume Claude Code / Codex CLI sessions.** `04c61f294` — new `hermes_cli/foreign_sessions.py` + docs.
- **Per-terminal `--continue` via terminal breadcrumbs.** `d6f02e349`.
- **Session picker lifecycle status + delete.** `933ef6947`.
- **`/save` multi-format export (json/md/html) on all platforms.** `26aa12337` (+ bare `/save` usage card `ca5c04818`; earlier `/export` `23ad35b7f`).
- **Persisted unread/read watermark (desktop).** `c75d83555`.
- **Archive current session** (hotkey + option-shift-click). `89f9375bc`.
- **Serialize turns across processes.** `6e929a969`.
- **Reset conversations stay listable; resume walker fenced at reset boundaries.** `ce89afa59` / `91c7a67f4`.
- **Prune filter derivation aligned; surface open sessions skipped by prune.** `0a42bc711` / `29dfbf2d6`.
- **session.history ships durable `row_id` stamps.** `2e9dcb7c5`.
- **Reject unstamped durable ordinal rewinds.** `79b7d969d`.

### CLI / slash commands

- **`/loop` — recurring in-session wakeups (Claude Code parity).** `f79440e0f` (+ gateway tick completion / isolation fixes `fccf2b718`, `9219cd394`, `2f6bbfbcb`):
  - `/loop [interval] <prompt>`; self-paced mode when interval omitted (local digest comparison, exponential backoff).
  - Stop: `LOOP_COMPLETE` marker, `--times N`, `--until <condition>` (goal_judge), `/loop stop`, `loops.max_ticks` backstop.
  - Persisted in SessionDB `state_meta` (`loop:<sid>`); survives `/resume` and compression like `/goal`.
  - Surfaces: CLI, Gateway (loop_wakeup_watcher), TUI/dashboard/desktop.
- **Sub-400ms warm CLI startup.** `55f9e472a` — probe-mode `check_fn`s, lazy MCP SDK, banner snapshot, parallel worktree add.

### Cron

- **`cron.allow_agent_scheduling` (default false).** `6e76c2698` — config-gated; only `cronjob` leaves cron-context denylist when enabled.
- **Resolve `origin` delivery at create time for cron-context job creation.** `a297edf3c` — never store dangling `origin` for follow-up jobs from scheduled agents.
- **Optional `profile` param on `cron.manage` RPC.** `d2672a349` (#86796).
- **Forward `repeat` through `cron.manage` add.** `a364390da` (#85602).
- **Stale execution-claim reap before one-shot `hermes cron run`.** `22e638db7` / stop orphaning one-shot CLI run `0fc2a10d8`.
- **NUL-bearing script path rejection before Path calls.** `40586082e` (#76762 class).
- **Windows uv-venv script jobs: process `.pth` files.** `029a0d8c7` (#86567).
- **Epochs not treated as unsafe lifecycle scripts.** `9ac1e65b0` (#86753).

### Kanban

- **Explicit notify/wake delivery modes with faithful wake session routing.** `6e81ce273` (salvage #37865) — `delivery_mode`: `notify` / `notify+wake` / `wake`; persists `chat_type` + `user_id_alt` for real session-key reconstruction.
- **GC stale done-task notify subscriptions.** `c69231270` — `kanban.done_sub_retention_days` (default 30; 0 disables).
- **Kanban worker-lifecycle / task-mutation / dispatch-tick plugin observers.** `5e1035168`.

### Gateway

- **Multi-connection / multi-source agent registry (desktop schema v2).** `b54b0521d` → end-to-end `aff19d025` (sockets, roster, SDK, fan-out updates) → discoverable Connections UI `34271eb06`. Docs: multi-gateway setup guide.
- **Route backends by `(connection, profile)`** — registry-scoped pool keys. `883b57cbd`.
- **Disk-usage telemetry + dashboard disk-pressure banner (NS-656).** `6977d21fa` — sibling of memory-pressure; critical `<256MB free or ≥95%`; elevated `<512MB free`.
- **Memory pressure + suspected-OOM restarts surfaced to users.** `e11d1ddc7` (NS-656).
- **Dump wedged worker stacks when turn reaper fires.** `a90d5369f`.
- **Concise background process notifications by default.** `1c971769e`.
- **Session-finalize plugin hooks off-loop and bounded.** `763b10c32`.
- **Persisted model routes kept consistent.** `095d25c61`.
- **Hygiene streak / failure cooldown persistence.** `b3df99083` / `39d2d858f`.
- **launchd supervisor marker preservation + Windows Ready scheduled-task as supervisor.** `7008fb81b` / `c69a0872e` / `730c5fc5d` / `2795b2ab9`.
- **Foreign `XDG_RUNTIME_DIR` must not crash user-systemd preflight.** `2d7c9ef6b` (#86558).
- **`share_auth` / MCP RPCs** — see MCP section.
- **Agent-to-agent message cards + collapse inter-agent replies (desktop).** `7a9634568` / `bd22451f0`.
- **Sender-side delivery notices.** `084265538`.

### Plugins

- **Manifest v2.** `bd6dcd4bd` (#64165) — `manifest_version` (file format) split from `api_version` (runtime API gen); `requires_plugins` (advisory + topo load order); `python_dependencies` (surfaced, never auto-installed); `config_schema` warnings; `hermes plugins doctor` v2 checks.
- **Plugin packs** (declarative shareable sets). `46e20083d` (#64166).
- **Community plugin index + `hermes plugins search`.** `2e0183169` (#64181).
- **Inter-plugin event bus** (`emits`/`listens`). `17030939d`.
- **Ownership ledger** unload lifecycle + widen to all registration surfaces. `22af80bcf` / `221974799`.
- **Redaction pattern registry** (vendor token formats as plugins) + ReDoS reject at registration. `fdd45323b` / `50f12e6ad` / `22002b1d3`.
- **Hooks:** `pre_command` observer + capability-gated `ctx.call_mcp` (`11310068c`); `pre_transcription` + STT prompt threading (`52eb8eb53`); streaming output observer hooks (`00f4da01e`); `classify_api_error` / `transform_api_error_classification` (`1d93b549c` / `c7c687aa4`); custom `@`-prefix context references (`b85e5bb4b`); capability-gated `ctx.platform_actions` (`e3983f91e`); plugins inject session messages (`f46c600a5`).
- **Desktop plugin SDK:** Dialog/ConfirmDialog/Toast (`7448e7a50`); focused-session state atoms (`68bd8befd`); busy turn flags (`0a9337d2b`); `host.deleteProfile` teardown-routed (`a15de3454`); load desktop half from unified agent-plugin packages (`4c1365b6c`).
- **`profiles.list` / `profiles.create` ws RPC + plugin session-navigation doors.** `89a84e1ae` (#85093).
- **`image.generate` ws RPC for plugin surfaces.** `ccce6976e` (#85183).

### Providers / Auth

- **`key_cmd`** — see Configuration.
- **Codex OAuth provider label → "ChatGPT or Codex Subscription".** `94be91941`.
- **Loopback custom-provider pool credentials exempt from usable-secret floor.** `08baf9653`.
- **Bedrock API key through named provider (CLI).** `a7253a6c0`.
- **Dashboard-auth: RFC 8252 native sign-in extended to password providers.** `56f1afc83` (#75808).

### Webhooks

- **Per-route toolset overrides for webhook agent runs.** `e4aeb6559` — route-level `toolsets` replaces platform-level resolution for that route only; deliberately NOT exposed via `hermes webhook subscribe` (manual config edit only — no self-grant of terminal).
- **Linear HMAC authentication.** `f13f3401a` (#87348).

### Memory / Hindsight / observability

- **Hindsight provider improvements.** `34c727c5c` (salvage #74379):
  - opt-in `recall_sync` (in-turn recall vs next-turn prefetch)
  - default `retain_source='hermes'`
  - starter memory template during `hermes memory setup`
  - unavailable-provider warning; local_embedded missing → actionable install hint
  - deterministic "recalled N memories" / "saving to memory" indicators
- **Langfuse tracing widened** to errors, sessions, subagents, MoA fan-out. `e665300d6`.
- **Relay session-span segmentation for continuous sessions.** `11c74beff`.
- **Nemo relay: bound plugin Relay marks** so wedged native pipeline cannot stall agent. `fe0a56ed1`.
- **Background-review usage attribution + cost controls.** `7095e23eb` (HEAD at review time).
- **Skip automatic background review inside delegation subagents.** `ece3bc7d5`.

### Desktop / TUI

- Concentration of window activity: **238 fix(desktop) + 57 feat(desktop)** — multi-connection registry, MCP fleet UX, Skills hub, session density/archive/unread, multi-source agents, Connections discoverability, open-in-terminal, transcript timestamps, thinking auto-collapse, per-profile Capabilities scope, cron+blueprint recipes in nav rail, setup_mcp consent cards, recurrence-to-cron suggestions.
- **TUI:** auto-collapse reasoning when phase ends (`2b0b4a219`); INFO prompt-accepted / turn-finished records (`be083358b`); settle session close against active turns (`ad85feec4`).
- **Desktop multi-gateway docs.** `ea2daa093` / `12859e9eb` / `647949332`.

### Platform adapters

- **Telegram:** keep `/loop` and synthetic sends in active DM topic (`25fabcf8e`).
- (No large adapter redesign this window; most platform work was gateway/desktop-side.)

### Reverts

- **`4ea2a0e54` Revert "Inspired by Perplexity Computer: Model Council mode for Mixture of Agents"** — reverts `8d9e18d40`. Removes Model Council mode / council-style MoA UX. Live MoA docs still describe preset-as-provider model (fanout, privacy_filter, per-slot reasoning) — council mode specifically is gone.
- **`3863de315` revert(gateway): keep profile truncation routing out of scope** — undoes profile-DB truncation routing attempt (`0640fe711` lineage).

### Salvaged PRs (authorship preserved)

- `d083b8559` — pre_tool_call `modify` (PR #28953)
- `6e81ce273` — kanban notify/wake modes (PR #37865 / @verybigdog)
- `34c727c5c` — Hindsight improvements (PR #74379 / @benfrank241)
- Multiple desktop/plugin salvages with AUTHOR_MAP updates throughout the window

## Docs / live-site corroboration (spot-check 2026-08-16)

Official docs at https://hermes-agent.nousresearch.com/docs/ corroborate several of the above and also surface items that may pre-date this window but remain absent from the frozen main reference:

- `hermes config get` / `unset`
- Provider per-model timeouts + stale timeouts
- `${env:VAR}` SecretRef parity with Cursor
- `runtime.nofile_soft_limit` (default 4096)
- Terminal backends: local | docker | ssh | modal | daytona | **vercel_sandbox** | singularity
- `updates.pre_update_backup` / `non_interactive_local_changes`
- MoA as normal provider (not a toolset); `/moa` one-shot sugar only; `fanout: user_turn|per_iteration|every_n:N`; `privacy_filter`; `reference_max_tokens`; per-slot `reasoning_effort`
- Machine-readable docs entry points: `/llms.txt`, `/llms-full.txt`

## Action items for next full reference re-compile

1. **Schema:** document SCHEMA_VERSION=26 + `git_metadata_generation`; SessionDB mixin map; watermark commit API (`archive_and_compact(watermark=)`); independent FTS layout versioning.
2. **Compression:** full rewrite of compaction section — lean vs legacy tail_mode, micro_compact, native gpt-5.6 Responses compaction, anchor index, digest pipeline, watermark concurrent-tail clone, eval harness.
3. **Delegation defaults:** max_iterations=250, max_concurrent_children=10, truncation markers, sequential tool deadline.
4. **Config surface inventory:** `model_overrides`, `key_cmd`, `compression.tail_mode`, `cron.allow_agent_scheduling`, `kanban.done_sub_retention_days`, `computer_use.permission_mode`, `loops.*`, `timeouts.tools.sequential_call`, `updates.*`, `runtime.nofile_soft_limit`.
5. **Slash commands:** `/loop`, `/save` multi-format, foreign-session import (`hermes sessions` Claude/Codex import).
6. **Gateway multi-connection registry** + `(connection, profile)` pool keys + disk/memory pressure telemetry + per-profile MCP/cron RPCs + `share_auth`.
7. **Plugin manifest v2** + packs + event bus + ownership ledger + redaction registry + hook catalog (`modify`, `pre_command`, `classify_api_error`, stream observers).
8. **MCP:** lazy start + schema cache + trust tiers (from 08-09) + per-profile lifecycle RPCs + desktop fleet UX.
9. **Kanban notify/wake delivery modes** + done-sub GC.
10. **Webhooks:** per-route toolsets (manual config only).
11. **Hindsight:** recall_sync, retain_source, indicators, setup templates.
12. **Computer-use:** browser-approve token flow + bounded capability_manifest.
13. **Remove/annotate:** Model Council MoA mode (reverted); profile truncation routing (reverted); pre-tool-call approval gate (reverted 07-12); DCP context engine / NeMo shared metrics (reverted 08-09).
14. **Defaults audit:** `agent.max_turns=500` (still wrong in main reference if it says 90).

## Known limitations of this pass

- Lightweight-local only — no full `Hermes_Architecture.md` rewrite.
- Spot-verified ~30 commits; desktop/TUI churn (300+ commits) summarized by theme, not per-commit.
- Official docs middle sections not exhaustively re-read (configuration page is ~190k chars); relied on head/TOC + targeted MoA page.
- Local install remains on v0.20.0 / `eac1e2512` (1,267 behind upstream); runtime behavior on this host may lag changelog claims that only exist on `origin/main`.
- Public mirror sync attempted after local skill patch; see job report for push status.
- Full SSOT now **+8,629 commits / ~8 weeks stale — re-compile critically overdue**.
