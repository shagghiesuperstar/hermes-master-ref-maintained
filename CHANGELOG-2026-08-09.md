# Hermes Architecture Reference — Drift Changelog 2026-08-09

## Scope of this review

- **Local Hermes version:** v0.19.1 (2026.7.30) — local `3f497e2b4` (on `main`, **1,483 commits behind** `origin/main`)
- **Upstream HEAD reviewed:** `f28469349` (origin/main as of 2026-08-09)
- **Prior SSOT anchor (last drift review upstream HEAD):** `65b73eb1e` (CHANGELOG-2026-07-26)
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **5,196** (~557 feat, ~2,687 fix, rest chore/test/refactor/perf/docs)
- **Window:** 2026-07-26 → 2026-08-09 (~2 weeks; prior weekly cron appears to have missed one cycle)
- **Releases in window:** `v0.19.0` (2026.7.20) · `v0.19.1` (2026.7.30) · `v0.20.0` (2026.8.3)
- **Full SSOT drift (cdcbc3a31 → origin/main):** **6,986** commits pending
- **Methodology:** lightweight-local model (per operator directive) + git-log area-bucketing + spot-verification (`git show --stat` + body) on ~50 high-impact commits. **Changelog-as-scaffolding** per the drift review decision tree (5,196 commits ≫ 500; operator said "lightweight local model only"). The main reference (`Hermes_Architecture.md`) is left untouched.

> **Why a changelog and not a full re-compile.** Delta is an order of magnitude above the 500-commit re-compile threshold, the full SSOT is now ~7 weeks / ~7k commits stale, and the operator asked for lightweight-local only. This pass continues the changelog chain (07-05 → 07-12 → 07-19 → 07-26 → **08-09**). **A full re-compile is now overdue** and should be authorized on the next non-lightweight pass — the scaffold chain is long enough that partial edits would be worse than leaving the reference frozen.

## High-impact architecture deltas (`65b73eb1e` → `f28469349`)

### Schema versions & state.db migrations

- **SCHEMA_VERSION advanced 22 → 23 → 24 → 25.**
  - `v23` — FTS storage-layout / activity foundations (independent FTS layout versioning continues).
  - `v24` — `53a5983af` activity-tracking columns (`last_activity_at` / `description` / `provenance`) already present in `SCHEMA_SQL` + column reconciler; version stamp advanced so upgrade/downgrade tooling sees the layout.
  - `v25` — further state layout advance on current `origin/main` (spot-check: `SCHEMA_VERSION = 25` in `hermes_state_common.py`).
- **SessionDB god-file split.** `21c7ae856` — mechanical extraction of ~2.9K LOC out of `hermes_state.py` into mixins: `hermes_state_common.py`, `hermes_state_schema.py`, `hermes_state_search.py`, `hermes_state_portability.py`. Import surface for operators is unchanged; internal module map in the architecture reference is now stale.
- **`hermes doctor` state.db health stats.** `64c342c1c` — read-only URI connection reports logical size, pages/freelist, WAL size, message/session counts, journal mode, holder count, FTS presence + deferred-rebuild status. Advisory warnings at >1 GiB and >256 MiB WAL.
- **Doctor WAL-reset / journal-mode exposure.** `658329708` + `a96a4621f` — per-database journal mode + repair command for exposed databases.
- **Sessions carry read/unread state.** `ec0c8d9c2`.
- **Config-gated transcript safety limits.** `f0794640f`.
- **`hermes sessions clean-markers`.** `e1a273969` (#78148) — permanently purge stale tool-call markers; backs up `state.db` by default before writes (`e18c040c3`).

### Configuration & provider routing

- **Config migration registry table-driven.** `326764e25` — 17 if-blocks → `(version, fn)` table in new `hermes_cli/config_migrations.py` (byte-identical semantics).
- **Config auto-migration support floor raised to v12** + deprecated shim retirement. `4b33e5663`.
- **Default `agent.max_turns` / tool-calling iteration limit: 90 → 500.** `1b081e489` — applied across `AIAgent` constructor, `DEFAULT_CONFIG`, CLI resolution, gateway env bridge, cron scheduler, TUI gateway. Explicit user config preserved (no `_config_version` bump). **Main reference still says 90 — stale.**
- **`display.personality` resolves as ephemeral system-prompt overlay.** `da6f0030a` — keeps `agent.system_prompt` user-owned; named personalities are ephemeral, not persisted into the owned prompt.
- **Reset-aware primary restore.** `6611d8700` — `CredentialPool.next_available_at()` gates restore-to-primary so subscription-window rate limits (Claude 5h / Codex weekly) do not burn two provider switches + two prompt-cache invalidations per turn while the window is still closed. Fail-open on missing reset info.
- **Actual Computer inference provider.** `a9acb400b` + `e79f16cab` — full provider plugin under `plugins/model-providers/actual/`, env-var metadata, config-driven local no-auth, reasoning-effort clamp.
- **qwen3.8-max replaces qwen3.7-max** in Nous portal + OpenRouter catalogs. `3c3ae7428`.
- **Transport: imply `prompt_cache_key` capability for `api.openai.com`.** `78e2987e2`.

### Compression / context

- **Per-turn micro-compaction.** `186cad02f` + cadence knob `9ca4ee72c`.
  - After completed turns, fold the oldest un-absorbed exchange into a rolling summary (amortize batch compaction stalls).
  - Config (current `origin/main` defaults): `compression.micro_compact: false` (**opt-in**), `compression.micro_compact_every_n_turns: 1`, `compression.micro_compact_defrag_threshold_tokens: 2000`.
  - Only newest summary marker kept (cumulative) — measured 4104→2572 tokens vs 4104→4797 without marker collapse.
  - **Note:** original commit message said "default on"; current `config_defaults.py` has `False` (opt-in). Trust live defaults.
- **Native OpenAI Responses server-side compaction for gpt-5.6.** `5e1b50115`.
  - Opt-in `compression.codex_responses_native: false`.
  - Hard-gated: gpt-5.6 family only + direct OpenAI/Codex routes only (xAI / Copilot / OpenRouter / relays / local never see the field).
  - Native threshold clamped ~8K below local trigger; structured rejection disables native for the session and retries once without it.
  - Reuses `codex_reasoning_items` sidecar — zero new state.
- **Session activity watchdog + stall notify + compress timeout.** `c2088efe9` (#72424; briefly reverted as PR #72817 then re-landed).
  - Mid-turn activity heartbeats → `SessionDB`.
  - Stall watchdog: `agent.session_stall_timeout` default **300s** — notify-only, does not kill the turn.
  - Compaction host budget: `compression.context_timeout_seconds` default 120 idle, `compression.context_total_ceiling_seconds` default 600 ceiling; on timeout cancel via commit fence and continue without dropping messages.
- **Pool-saturated compression-attempt telemetry.** `a0e700c4c`.

### Tools / tool_search progressive disclosure

- **Tiered tool disclosure (major architecture change).** `e869accc1` → `0986ac393` → `e9fe060eb`.
  - **Tier 0:** no MCP/plugin tools → everything eager.
  - **Tier 1:** deferred tools whose catalog listing fits `min(threshold_pct% of context, listing_max_tokens)` → bridge + skills-style listing (degrades to names-only over budget).
  - **Tier 2:** listing over budget even names-only (Cloudflare-scale ~3,320 tools) → bare bridge + **per-server one-line summary** (`cloudflare (3320 tools)`); discovery via `tool_search` only.
  - Activation is driven by **deferrable-tool presence**, not schema-token share.
  - Current defaults: `tools.tool_search.threshold_pct: 5`, `listing_max_tokens: 4000` (cap raised earlier to 20000 then partially walked back — trust live config_defaults).
  - Listing degradation is **per server, largest first** (mixed form: Linear fully listed + Cloudflare summarized).
  - Byte-stable deterministic ordering → prompt-prefix cache safe.
- **Self-repo git mutation hard-block in `terminal_tool`.** `206531a1e` + `ecbe6ef0d` — detects git mutations targeting the running source checkout; `force=True` cannot bypass; local backend only. Redirects to worktree/temp clone.
- **Protected agent-instruction file writes always require approval.** `fe66596df` — `AGENTS.md` / `CLAUDE.md` / `SOUL.md` / `.cursorrules` / project-local `.hermes/*` always prompt even under `--yolo`; fail closed with no human channel. Config: `security.protected_instruction_files` (default true) + `security.protected_instruction_extra_patterns`.
- **`env_passthrough` allowlist for command-provider secret scrub.** `e251e78df`.
- **Vision optional region zoom crop.** `e166159f2`.
- **`read_file` document extraction widened** (PDF/legacy Office/ODF/RTF/EPUB via optional anydoc) — later schema honesty fix `ff3793fdf` stopped promising anydoc in the tool schema when unavailable.
- **`read_window_below` tool** — which OS window is underneath the desktop app. `406501fd9`.
- **Media upscaling** default-on for sub-2MP image models (FAL + Krea). `66ea4e686` / `137960c9a`.

### MCP

- **Fingerprint-keyed on-disk MCP tool-schema cache.** `135a29452` — `~/.hermes/mcp_schema_cache.json`, keyed by server name + connection-defining config fingerprint.
- **Lazy server startup from schema cache.** `1d5ecad56` — `mcp_servers.<name>.lazy: true` (default OFF) registers tools from cache without spawning; first tool use connects. Resource/prompt handlers also connect-on-first-use.
- **Trust-tier gating for write-capable MCP tools.** `c8369e37f` — `mcp_servers.<name>.trust: full|untrusted`. Untrusted servers route non-`readOnlyHint=True` tools through approval; missing annotations = write-capable (fail closed); unrecognized trust = untrusted; missing key = full (compat). Schema cache persists `readOnlyHint`.
- **Collapse const-only anyOf/oneOf unions to property enums.** `37cc99992`.
- **fnmatch glob support in `tools.include`/`exclude` filters.** `e7172ab1b`.
- **Warn on hidden whitespace in MCP config values.** `89f920901`.
- **Comfy Cloud MCP catalog entry** (remote HTTP + native OAuth 2.1). `fee0eae6d`.
- **Per-server MCP identity header + kanban orphaned-card reconciliation.** `9fad45fcd`.

### Approvals & security

- **`approvals.smart_policy`** — operator-customizable smart-approval policy text. `bd1db5460`.
- **Consecutive-denial circuit breaker** for smart approvals. `a0112ef26` — after N consecutive guardian denials, escalate to hard-stop instruction.
- **`hermes approvals suggest`** — mine approval history into allowlist proposals. `db90e3620`.
- **`hermes approvals test`** — dry-run approval verdict CLI (exit 0 allow / 2 ask / 3 deny); uses real runtime detectors including de-obfuscation variants. `563f0a6fd`.
- **Cross-surface `/approvals` mode command.** `f9cd57791`.
- **Docker/podman daemon-redirect commands require approval.** `643770122`.
- **Global emergency stop: `hermes pause` / `hermes resume`.** `5db1b72b1` — resumable ESTOP sentinel at `$HERMES_HOME/ESTOP`; halts NEW work only (cron dispatch, kanban spawn, new gateway turns); in-flight work finishes; `hermes status` shows PAUSED banner. Never kills running turns.
- **Self-repo mutation block** — see Tools above.
- **Protected instruction files** — see Tools above.

### CLI

- **`hermes pause` / `hermes resume`** — see Approvals.
- **`hermes approvals test` / `hermes approvals suggest` / cross-surface mode** — see Approvals.
- **`hermes doctor --live`** — opt-in real-call backend probes (Firecrawl credits, FAL models, browser about:blank, MCP initialize+tools/list, TTS/STT voices). Bounded, read-only, default off. `1006faa6f`.
- **`hermes import-agent`** — import Claude Code and Codex CLI setups (CLAUDE.md/AGENTS.md, permission allowlists, MCP, skills, memories). `24c3c27ba`.
- **`!` shell mode** — run a command without spending a model turn. `9704ed86c`.
- **`/init`** — generate or update `AGENTS.md` from a project scan. `95b7ea5e5`.
- **`/export` + `/import`** slash commands for profile sharing. `bde8c4e10`.
- **`/refine`** — on-demand memory/skill self-improvement review (fires background review fork immediately; optional focus). `8f2712725`.
- **`/heartbeat`** — recurring session re-entry prompt when idle. `6518aa184`. Session-scoped + in-process; durable schedules remain cron's job.
- **`/goal gate add|list|remove|clear`** — deterministic quality-gate commands that must pass before `/goal` completes (run before LLM judge; unchanged-workspace skip; bounded retries). `6e041d524`.
- **`/focus`** — reduced-output view. `d6fa2709d`.
- **`/context` unified** visual context-usage breakdown (CLI + gateway). `07370a9db` / `4fe2fecf5` / `a0112ef26`-adjacent.
- **`/diff`** cross-surface with staged/all/session modes. `0fa5e41c8`.
- **Double-ESC discards draft** even mid-stream. `c1ec39416`.
- **Session titles in status bars.** `5a16635f4`.
- **`hermes sessions retitle-skills`.** `16042b0c4`.
- **`--resume latest` keyword and `--in DIR` launch flag.** `c5f5fa40c`.

### Cron

- **Per-job durable notepad** (KV scratchpad in `cron/notepad.db`). `04e8a661f` — `hermes cron notepad <job_id> [get|set|delete|list]`; injected into job prompt when non-empty; size caps 16KB/value, 64KB/job.
- **Monitor-mode jobs** — hash-suppressed change detection. `6dff2109a` — `monitor_script` or `monitor_url` runs before any agent machinery; unchanged → silent `no_change` (no LLM); changed → diff block injected; source failure → ERROR alert (hash untouched).
- **Pre-dispatch configuration validation (`blocked_config` + alert-once).** `ed903f953` — missing API key / skill not ready / delivery platform unknown checked BEFORE AIAgent construction; alert delivered exactly once across ticks; `cron.preflight` default **true**.
- **User-owned model pins + `cron.model` fleet default.** `d464ae365`.
- **`usage_audit.jsonl` logger** for cron token-leak instrumentation. `15927c1d2`.
- **`skip_background_review=True` on cron AIAgent** + title-generation non-presence. `d3e3c6234` / `eaeba6474`.

### Gateway

- **Session activity watchdog / stall notify / compress timeout** — see Compression (`c2088efe9`).
- **Session workspace move RPC.** `28b3b0dd1` — `session.workspace.move` re-homes a stored session's cwd + replaces git meta so project-tree grouping follows; refuses mid-turn ("session busy").
- **iMessage-style message reactions** — storage, RPC, agent tool, model context. `7d92056c4`.
- **Opt-in `latency` runtime footer field.** `ad345a99d`.
- **Key-addressed `plugins.manage` rows + portable MCP toolset fold-in.** `a60b492e0`.
- **Discord auto-thread sessions keyed on `prospective_thread_id`.** `a041526ef` (#76513).
- **Group unplaced sessions into a Home bucket** in the project tree. `60b6ea237`.
- **Generalized change watcher** (pet/cron/sessions broadcasts; was skin-only). `3378e528e`.
- **Simplex channel enumeration + `hermes send --list` platforms.** `7483745da`.
- **Process-registry scope identity bound to pid** + CLI workers off controlling tty. `aa32e8114` / `ff5dfdece` (near HEAD).

### Plugins / portable Agent Plugins / A2A

- **Portable agent component load + validate.** `c5117655b` / `ca78c6d7a` — Agent Plugins v1 packages load skills/MCP into native runtime.
- **streamable-http entries mapped into native MCP runtime.** `471baea52` — absolute http(s) only; strict cross-origin redirect header stripping; legacy `sse` still skipped.
- **A2A (Agent-to-Agent) protocol plugin** under `plugins/platforms/a2a/` (zero core edits). `837003b1e` closes #514 — outbound `a2a_discover`/`a2a_call`/`a2a_list`; inbound Agent Card + JSON-RPC into live gateway session; bearer auth; stdlib only.
- **Outbound webhooks (hooks.outbound).** `3829e34e2` — signed lifecycle events (HMAC-SHA256 `X-Hermes-Signature-256`) to external HTTP endpoints; fire-and-forget bounded queue; `HERMES_SAFE_MODE` skips registration.
- **Public subagent lifecycle API for plugins.** `1865fb5fc`.
- **Buzz (Block/Nostr) platform adapter plugin.** `66fc2e2a9` + NIP-42 auth / BIP-340 signing (`07e931fcb` / `21c7b806a`).
- **Profile REST export/import + `extra_files` overlay.** `d1196750c` — desktop bundles theme/layout/skills as portable profile archive.

### Profiles / sessions

- **Profile shareable bundles** (desktop + CLI `/export`/`/import`). See CLI + Plugins.
- **Name a session the moment it starts.** `f726090d4`.
- **Sessions list carries pin flag; pinned rows never dropped.** `3290807e6`.
- **`hermes sessions clean-markers`** — see Schema.

### Kanban

- **Talk to a running worker without restart** — comment-thread → OUT-OF-BAND steer channel. `901205420`.
- **Per-task model + thinking-depth pins** from the board + REST. `602fc5f9f` / `f0ed0aebb` / `0b69a6ac0`.
- **Task effort estimate via auxiliary model.** `346149c4f`.
- **Boards scoped to a project.** `027ef381a`.
- **Orphaned-card reconciliation.** `9fad45fcd`.
- **Desktop Kanban dashboard-parity board plugin on the SDK.** `79e7adae2`.

### Delegation

- **Optional structured-output schema on `delegate_task`.** `d6ee58b58` — per-task `output_schema`; child gets OUTPUT CONTRACT block; parent validates with one bounded retry; result gains `schema_valid` only when schema requested.
- **Per-delegation cost in result entry.** `d7635e43b`.
- **Batch task quality validation before spawn.** `94bc3194b`.
- **Structured stall metadata + live per-child status in `/agents`.** `b792bd052`.
- **Redacted child tool history exposure.** `e369d6ea3`.

### Skills

- **Advisory SKILL.md convention linter on create.** `fce314eab` — `tools/skill_linter.py`; findings attached as `lint_warnings` (soft); CLI exits 1 only on ERROR severity. CI-enforced authoring standards: `55982159d`.
- **Clean-room MIT rewrite of office document skills** (docx/xlsx/powerpoint/pdf) replacing Anthropic proprietary LICENSE. `51570f4da` + parity extension `fad88cf13`.
- **Dedup repeat `skill_view` calls** with unchanged-content stub. `2a3a7e6f5`.
- **Org skills editable in place; local edits survive org updates.** `981feb673`.
- **`/learn` expansive knowledge-base skills** for books/large corpora. `32e7fb07a`.
- Many new bundled/optional skills: competitor-news-monitor, social-media-content-calendar, weekly-review-planning, product-price-monitor, meeting-action-items, google-workspace-daily-brief, github-issue-to-pr, email-inbox-triage, document-to-action-items, grounded-citations (+ fact-checking mode), actual-setup.

### Providers / Auth / Voice / STT / TTS

- **Actual Computer provider** — see Configuration.
- **Wake word ("Hey Hermes")** + profile-routed wake ("hey \<profile\>") + remote desktop client-mic streaming. `5f43452e9` / `2a35c8f0b` / `7e1624182` (#79491).
- **STT idle unload** for local whisper. `7b006ea6e`.
- **STT pre-upload silence trim** for cloud providers. `a683ef95d`.
- **Streaming TTS** — Gemini SSE + xAI WebSocket; `tts.streaming.provider` knob; gateway streaming TTS adapter contract. `bc4dcb1b0` / `3a4aa2f8f`.
- Full STT configurability in `hermes tools` + GUI. `96bf65a6f`.

### Desktop / TUI (architecture-adjacent only)

Desktop absorbed the plurality of feats (167 `feat(desktop)`). Architecture-relevant:

- **HUD mode** (Spotlight bar + fading chat band + session handoff). `7b0dbd224` / `e8b83f37c` / related.
- **Agent Plugins in Settings → Plugins** + descriptions + folder open. `c86da8397` / `44790bc9c`.
- **`ctx.os` curated OS door for plugins** (notifications, links, files, clipboard). `6acae3075` / `e8ccb4a2e`.
- **Share profile as portable bundle.** `6e7eafc7e`.
- **SSH in Gateway settings / Cloud-aware connection model / secure Desktop SSH bootstrap** (earlier in window, ~2026-07-15 cluster).
- **Multiple cron delivery targets** from desktop. `67927808b`.
- Electron rolled back to 40.10.2 (`bb8280b75`).

### Observability

- **NeMo Relay runtime + shared metrics** was added then **reverted** (`841a5a744` / PR #73053). Do not document as current.
- Partial observability surface remains (model/provider usage, bounded tool metrics, skill metrics) from the broader cluster — treat as unstable until a non-reverted design lands.

### Reverts (call out by identity)

| SHA / PR | What |
|---|---|
| `206f74baa` + `0647bf988` / #81891 | **DCP context engine** fully reverted (feat + rewire fix) |
| `841a5a744` / #73053 | **NeMo Relay runtime + shared metrics** observability integration |
| `2c1809e6c` / #72858 | Temporary revert of session activity watchdog (#72817); **later re-landed** as `c2088efe9` |
| `ad12df6ba` | Restored **Vercel AI Gateway + Vercel Sandbox** (undo of prior removal #33067) |
| `bb8280b75` | Electron rolled back to 40.10.2 |
| `7f87b6724` | Streaming-backfill gate (broke E2E invariant) |
| `279be8211` | AttributeError circuit-break from commit-splice |
| Several desktop/installer cosmetic reverts | Non-architectural |

### Salvaged / notable ports

- Micro-compaction + `/refine` + `/heartbeat` + `/goal gate` concepts adapted from Prime Intellect Prime-Agent (Continual Harness) — Hermes-native durable state (memory/skills/SessionDB).
- MCP trust gating ported from cloudflare-os `classifyTool()` (Apache-2.0 idea+pattern).
- Protected instruction writes ported from RooCodeInc/Roo-Code `RooProtectedController` (Apache-2.0).
- ESTOP pattern from gastownhall/gastown `estop.go` (MIT).
- Cron notepad inspired by Amp (idea-level).
- `hermes approvals test` inspired by Amp `permissions test` (idea-level).
- A2A plugin closes long-running #514 cluster on pure plugin surface.
- Native Responses compaction direction credit PR #76950 (@laryhorb); minimal reimplementation on current main.
- Session activity watchdog cherry-picked from PR #72424 (@fangliquanflq).

## Action items for next full reference re-compile

Concrete per-section edits for `references/Hermes_Architecture.md` (do NOT apply until a full re-compile pass):

1. **Agent loop / defaults** — `max_turns` / `max_iterations` default **500** (was 90). Update every mention + diagrams.
2. **state.db** — document SCHEMA_VERSION **25**, activity-tracking columns, SessionDB mixin split (`hermes_state_{common,schema,search,portability}.py`), read/unread, pin flag, clean-markers, doctor health stats.
3. **Compression** — new subsection for micro-compaction + native gpt-5.6 Responses compaction + stall watchdog / compress timeout knobs. Update TTFT/cache interaction notes (micro-compact breaks prefix on cadence).
4. **Tool assembly** — replace binary tool_search activation with **tiered disclosure (0/1/2)** + per-server mixed listing + current defaults (`threshold_pct: 5`).
5. **MCP** — schema cache path, lazy start (`lazy: true`), trust tiers (`trust: full|untrusted`), `readOnlyHint` call-time gating, streamable-http portable plugin mapping, Comfy Cloud catalog.
6. **Approvals** — `smart_policy`, denial circuit breaker, `hermes approvals test|suggest`, cross-surface mode command, protected instruction files, self-repo git hard-block, docker/podman daemon-redirect approval, ESTOP (`hermes pause`/`resume`).
7. **Cron** — notepad.db, monitor-mode, preflight `blocked_config`, `cron.model` fleet default, `skip_background_review`, usage_audit.jsonl.
8. **Gateway** — delivery ledger + turn lease (from 07-26 changelog — still missing from main ref), activity watchdog, workspace.move, reactions, profile_routes (07-26), plugins.manage key-addressing.
9. **Plugins** — Agent Plugins v1 portable packages, A2A platform plugin, outbound webhooks (`hooks.outbound`), Buzz/Nostr adapter, `ctx.os` desktop plugin door.
10. **CLI surface** — `/refine`, `/heartbeat`, `/goal gate`, `/init`, `/export`/`/import`, `/focus`, `/context`, `/diff`, `!` shell, `hermes import-agent`, `hermes doctor --live`, `hermes pause`/`resume`.
11. **Delegation** — `output_schema` on `delegate_task`, cost surfacing, batch quality gate.
12. **Kanban** — live comment steer, per-task model/reasoning pins, project-scoped boards, orphan reconciliation.
13. **Providers** — Actual Computer, qwen3.8-max, reset-aware primary restore, Vercel AI Gateway restored.
14. **Skills** — advisory linter, clean-room MIT office skills, org-skill editability.
15. **Config migrations** — table-driven registry in `config_migrations.py`; support floor v12.
16. **Remove / do not document** — DCP context engine (reverted), NeMo Relay shared metrics (reverted).
17. **Consume the full changelog chain** as merge map: `CHANGELOG-2026-07-05.md` → `07-12` → `07-19` → `07-26` → **`08-09`**.

## Known limitations of this pass

- **Docs site not re-fetched** end-to-end; claims grounded in `git show` bodies + `origin/main` `config_defaults.py` probes, not live docs HTML.
- **Local install is 1,483 commits behind upstream** (pinned at v0.19.1 / `3f497e2b4`). Runtime behavior on this machine ≠ upstream HEAD. Changelog describes **upstream**, not local.
- **~5.2k-commit window** — only ~50 commits fully spot-verified with body reads; remaining feats summarized from subject lines + area filters. Next re-compile must `git show` any claim before promoting it into `Hermes_Architecture.md`.
- **`micro_compact` default discrepancy** — commit message said default-on; live `config_defaults.py` on origin/main is `False`. Recorded as opt-in; re-verify at re-compile.
- **SCHEMA_VERSION 25** exact migration contents not fully expanded (only stamp confirmed). Re-compile should read `hermes_state_schema.py` migrations table end-to-end.
- **Desktop HUD / wake-word / streaming TTS** documented at architecture-relevance only; full desktop surface belongs in desktop docs, not the core architecture SSOT.
- **Prior weekly cron gap** — last review 2026-07-26; this pass covers ~2 weeks. No intermediate changelog exists for 2026-08-02.
- **Public mirror** will be synced with this changelog + updated `SKILL.md` in the same pass.

## Verification trail

```text
Local version:     Hermes Agent v0.19.1 (2026.7.30) · 3f497e2b4
Upstream HEAD:     f28469349 (2026-08-09)
Prior review HEAD: 65b73eb1e (CHANGELOG-2026-07-26)
Full SSOT anchor:  cdcbc3a31 (Hermes_Architecture.md, 2026-06-21)
Window commits:    5,196 (65b73eb1e..f28469349)
Full SSOT debt:    6,986 (cdcbc3a31..f28469349)
Local behind:      1,483
Decision:          changelog-as-scaffolding (no Hermes_Architecture.md edit)
Skill version:     1.7.0 → 1.8.0
```
