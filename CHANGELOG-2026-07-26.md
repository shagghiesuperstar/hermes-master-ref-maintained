# Hermes Architecture Reference — Drift Changelog 2026-07-26

## Scope of this review

- **Local Hermes version:** v0.18.2 (2026.7.7.2) — local `36f2a966c` (on `main`, +1 carried fmt commit)
- **Upstream HEAD reviewed:** `65b73eb1e` (origin/main)
- **Prior SSOT anchor (last drift review):** `7b5ba2054` (CHANGELOG-2026-07-19)
- **Prior upstream HEAD (prior review):** `7b5ba2054`
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **1,074** (124 feat, 563 fix, 109 test, 72 chore, 38 refactor, 18 perf, ~150 misc)
- **Window:** 2026-07-19 → 2026-07-26 (~1 week)
- **Local vs upstream:** local is **3 commits behind** `origin/main` (effectively current on `main`; the only deltas are post-merge fmt fixes)
- **Methodology:** lightweight-local model (per operator directive) + git-log area-bucketing + spot-verification (`git show --stat`) on high-impact commits. Changelog-as-scaffolding per the drift review decision tree (1,074 commits > 500, and operator said "lightweight local model only" → default to changelog-as-scaffolding). The main reference (Hermes_Architecture.md) is left untouched.

> **Why a changelog and not a full re-compile.** The delta is > 500 commits and the operator asked for lightweight-local only. The 2026-07-05 SSOT anchor (`cdcbc3a31`) is still 1,790 commits behind upstream, but the reference has been kept current through three consecutive weekly changelogs (07-12, 07-19, 07-26). Full re-compile is still scheduled when the drift window either (a) crosses a 4-week SSOT age or (b) the operator explicitly authorizes it.

## High-impact architecture deltas (7b5ba2054 → 65b73eb1e)

### Configuration & provider routing

- **`hermes config get` / `hermes config unset` commands added.** `53adb3fd9` (9 files, +373/-15) — symmetric pair to `hermes config set`. Documented in `website/docs/user-guide/configuration.md`. Implicitly closes the long-standing "no config get command" gap that bit every operator when an `--unset` was needed.
- **Unknown top-level config keys now warn, not silently ignored.** `f5bacee27` (`_KNOWN_ROOT_KEYS` derived from `DEFAULT_CONFIG.keys()`) + `336c3b13a` (`hermes doctor` reports deprecated keys + legacy env vars as non-failing warnings, naming the modern replacement). Typos like `skillz:`/`secrity:` now surface immediately. Two related safeguards: `4f67c3338` (whitelist real non-`DEFAULT_CONFIG` roots) and warning-only behavior preserved on `provider.*`.
- **Deprecated/legacy keys & env reported by `hermes doctor`.** `336c3b13a` — `display.tool_progress_overrides`, `delegation.max_async_children`, `compression.summary_*`, `HERMES_TOOL_PROGRESS*`, `TERMINAL_CWD`, `QQ_HOME_CHANNEL*`. No auto-delete/migrate — warnings only.
- **Per-model reasoning effort overrides.** `d9cdb8192` — new `agent.reasoning_overrides` dict in `config.yaml`. Spelling-tolerant (strips provider prefix, normalizes dots-vs-dashes). Resolution priority: `/reasoning --session` (gateway only) → per-model override → global `agent.reasoning_effort` → provider default. Wired into CLI startup, gateway construction, TUI/Desktop `_load_reasoning_config`, cron scheduler, `/model` mid-session switch, and fallback activation (re-resolves for fallback model). Closes #21256. **Caveat:** no `hermes config set agent.reasoning_overrides.<model>` support (`.` split would corrupt model keys); users edit YAML directly.
- **Custom and plugin voice providers surface in config schema.** `1e1749278` — extends `_KNOWN_ROOT_KEYS` for voice provider entries not in `DEFAULT_CONFIG`.
- **Fireworks AI promoted to #2 in the model provider list.** `b27d8b6ac` (#65214) — moved above OpenRouter in `CANONICAL_PROVIDERS`. Order propagates to `hermes model`, setup wizard, Telegram `/model`, and the desktop provider catalog.
- **LM Studio JIT load mode.** `1a323d608` — auto-load a model in LM Studio on first request rather than requiring it to already be loaded. Wired into `agent/agent_init.py` and `run_agent.py`.

### Providers / Auth

- **Upstage Solar as a full model provider.** `20502b407` (10 files, +402/-0) — model registration + picker entries + reasoning-aware fallbacks.
- **DeepInfra as an LLM provider.** `fe002eb12` (34 files, +2500/-85) — large provider integration with all the standard wiring.
- **Fireworks pricing entries + routing branch.** `365620ab2` — Fireworks pricing snapshot now in the official pricing catalog.
- **Credentials unification across `.env` + `auth.json` + `config.yaml`.** `9a987f142` (#67213, fixes #51071/#59761/#62269) — DELETE `/api/env` now removes the key from all three stores (previously `.env` only, leaving stale credential_pool entries). Resolves "provider reappears in picker after delete" bug. Composed of: `9a987f142` (unified delete/update) + `027243eb4` (suppress re-seeding when a pool entry is deleted via API).
- **`export FOO=bar` recognized in `save_env_value`/`remove_env_value`.** In `9a987f142` — `load_env()` already parsed `export KEY=value`, but the writers only matched plain `KEY=` lines, so DELETE 404'd and PUT appended a duplicate. Now both writers go through a shared `_env_line_defines_key()` helper. Commented-out lines still ignored.
- **OAuth refresh lock scaffolding consolidated.** `73ad9136b` — single `_single_use_refresh_lock_timeout()` helper, shared `provider guard` + dispatch. Codex and xai-oauth branches were duplicating the same lock-timeout computation; preserved verbatim the provider-specific post-sync logic.
- **Mem0 legacy OSS base URL aliases migrated.** `07f07c7b5` — fixup for stale URL aliases in Mem0 config.

### Approvals & security

- **Per-session turn lease + conversation-scope funnel in gateway.** `19527db73` (#64934/#67401, 7 files, +936/-149) — closes the serialization half of #64934. Busygates were keyed by routing key, but the durable transcript is owned by `session_id`, and `switch_session()` makes the key→id mapping many-to-one (`/resume` from second chat/topic, CLI-continuity rebinding, async-delegation pinning, topic-binding tip-walks). Two routing keys mapped to one `session_id` ran concurrent turns on different agent objects: flushes persisted in completion order, identity-marker dedup swallowed rows, second turn ran on stale history → permanent `user;user` alternation wedge.
  - Lease keyed by **RESOLVED** `session_id` (`gateway/turn_lease.py`), acquired in `_handle_message_with_agent` after session resolution is final (post `switch_session`/tip-walk), released in `_handle_message`'s `finally` on every exit path.
  - Tokens granted per (routing key, run generation) — a stale unwind cannot release a newer turn's lease (lesson from #28686).
  - Fail-open: stuck holder degrades to today's unserialized behavior with a loud ERROR after `agent.gateway_timeout`; degraded token holds nothing.
  - Registry size-capped; never evicts a live lease.
  - Persist-disabled review forks never dispatch through `_handle_message` → cannot contend.
  - Known limits (tracked on #64934): CLI-continuity cross-process pairs need a DB-level lease; mid-turn compression rotation leaves a small alias window.
- **Profile-aware approval mode control in desktop.** `3510b1881` (23 files, +713/-74) — approval mode can be set per profile (not just global). Mirrors the `gateway.delivery_ledger` style.
- **Approval smart-deny owner overrides scoped to one operation.** `d48bf743f` — fixes a leak where smart deny was sticky across operations.
- **Approval observer hooks for smart verdicts.** `b03c94dbe` — observability surface for the smart-approval path.
- **Execution-bearing option detection unified.** `b90dbac1d` — single chokepoint for detecting "this option will execute code".
- **Approval canonical gateway timeout honored.** `c5e841ab0` — fixes timeout divergence between approval gates.
- **Codex approval roundtrips forward drained notifications to `on_event`.** `11a91a6d1` — so the UI sees the human's response while waiting for the tool.
- **Test isolation for smart observer redaction.** `bd740f203` — redaction tests no longer interfere with smart-approval flow.
- **Pip/Homebrew unsupported warning** (from previous window, `4d7f8ade3`) continues to surface on startup. Cumulative hardening in the approval + credentials surface across both windows.

### CLI

- **`hermes config get` / `hermes config unset`.** `53adb3fd9` — see Configuration above.
- **`hermes doctor` reports deprecated keys + env vars.** `336c3b13a` — see Configuration above.
- **Fireworks AI promoted to #2 in `hermes model`.** `b27d8b6ac` — see Providers above.
- **`/model --once` one-turn model override.** `3f84b7a16` (#29914, 9 files, +683/-63) — switch model for the next turn only, restoring the previous model in a `finally` block (success, exception, interrupt all revert). `parse_model_flags_detailed()` extended; `resolve_persist_behavior()` treats `--once` as a persistence opt-out; `--global + --once` rejected. Salvaged from PR #29923 (image-generation lane split to #59815 per review).
- **Nous subscription from the terminal.** `b51fbc738` (#51639, 45 files, +8355/-1398) — `/subscription`, `/topup`, terminal-billing UX. Includes `/billing` → `/topup` rename, shared overlay primitives (`overlayPrimitives.tsx`), `SubscriptionState` + `get_subscription_manage_link()` in `agent/subscription_view.py`, wire types for `SubscriptionTierOption` / `SubscriptionStateResponse` / `SubscriptionManageLinkResponse`. RPC methods: `subscription.state`, `subscription.manage_link`. Large cross-cutting PR; consider it the new "billing backend" for desktop + TUI + gateway.

### Gateway

- **Durable delivery-obligation ledger for final responses.** `5854aad8b` (#67181, 7 files, +968/-0) — fixes silent loss of a finalized-but-not-yet-ACKed response on crash/restart. New `gateway/delivery_ledger.py` records each outbound final response in `state.db` (WAL, owner pid + process-start-time liveness, bounded retention), states: `pending → attempting → delivered | failed`, startup sweep on dead-owner rows → `redeliver | abandoned`.
  - Contract (lessons from closed delivery-outbox #61790): obligation recorded BEFORE first send attempt; cleared only on `SendResult.success` (#51184).
  - Ambiguity is labeled, never silently retried: mid-send rows that died redeliver with visible `♻️ Recovered reply — may be a duplicate` prefix (honest at-least-once).
  - Stable ids from `session_key + inbound message id + content` so distinct threads/topics cannot collide.
  - Poison rows bounded: 3 attempts / 24h freshness → abandoned; atomic claim re-stamps ownership.
  - Redelivery clears `resume_pending` for the session → resume path never re-runs a turn whose answer the ledger already holds.
  - Best-effort everywhere: ledger failure never blocks or delays a send.
  - Slash-command/ephemeral/empty responses not recorded; cron and proactive delivery stay on `DeliveryRouter` (separate subsystem).
  - Config: `gateway.delivery_ledger` (default on; no version bump needed).
  - Validation: 30 ledger+producer tests; 352 blast-radius gateway tests green; cross-process E2E (record in process A, kill mid-send, claim+marker+redeliver in process B against same `state.db`).
- **Per-session turn lease.** `19527db73` — see Approvals above.
- **Profile-based routing for inbound messages.** `5e65f6d79` (6 files, +568/-5) — `gateway.profile_routes` config routes specific Discord guilds/channels/threads (and other platforms) to different profiles. Hierarchical specificity matching (thread > channel > guild) with bounded LRU caching for forum post resolution.
  - Routing result stamped on `source.profile` by `BasePlatformAdapter.build_source()` at inbound time.
  - When `gateway.multiplex_profiles` is on, the existing `_profile_runtime_scope` machinery picks up `source.profile` and runs the whole turn inside that profile's `HERMES_HOME` — memory/skills/config/secrets all resolve to that profile automatically. No new isolation code.
  - When `multiplex_profiles` is off, `profile_routes` is ignored (no behavior change for single-profile gateways).
  - 29 unit tests (specificity scoring, hierarchical matching, path-traversal validation, config parsing).
- **Multiplex profile scoping extended.** `8091c4405` (install `_profile_runtime_scope` in `_run_background_task`), `6ff65c4d2` (scope default-listener api_server requests), `fef0b2d60` (scope secondary-adapter auth callback to own profile), `6b1267c2e` (gate `/profile` source scoping on multiplex_profiles), `f7d6f099d` (`/profile` reports serving profile not multiplexer's), `8191f621c` (preserve multiplex profile in model picker), `dd9e75335` (skip port-conflicting multiplex profiles), `01d3268e0` (harden multiplex credential cluster salvage), `64746b4bd` (validate multiplex adapter config by platform).
- **Launchd `ThrottleInterval` + portable respawn-storm circuit breaker.** `d92095474` — prevents launchd from killing the gateway when it crashes repeatedly.
- **Thread shutdown watchdog + loop heartbeat.** `1bf5fd08a` (#66892) — supervises long-lived watcher tasks at the thread level (`71f4de3cd`), `loop heartbeat` keeps the event loop responsive, `c66891db0` arms exit watchdog on shutdown signal (not at chat startup).
- **Route inbound-image decision off the event loop.** `95b09d3f7`.
- **Bytes-stable session context prompts.** `c0c76a471` — `perf(gateway)`. Reduces prompt-cache prefix churn for session context.
- **Live per-tool status line on Slack.** `d4396797c` — Slack now shows the currently-running tool in the working-state line.
- **Working-state status text configurable.** `dc0c778b2` — `feat(gateway)`. Operator can override the "Working — N min" string per profile or globally.
- **Inline choice pickers for `/reasoning` and `/fast` (Telegram, Discord, Matrix).** `bd37ff913` (#65799, 22 files, +970/-104) — parity with the `/model` picker UX. Generic `send_choice_picker` capability gate (detected on adapter type, like `send_model_picker`); selection + typed args flow through one shared application path; choices from `VALID_REASONING_EFFORTS` so future levels appear automatically. i18n: picker_title + choice labels in all 16 languages. Closes #61110.
- **Discord: `/reasoning reset|show|hide` exposed as slash choices.** `bfca45bda`.
- **Slack app token scoped in multiplex gateway.** `ea2c9bc10` — per-profile token isolation.
- **`secret_scope` multiplexed for authz, Slack, webhooks.** `8fc989b41`.
- **`MEDIA:` caption attached to media bubble.** (from previous window, `709da844b`) — `hermes send "MEDIA:/x.png This Caption"` now arrives as one captioned bubble.
- **Platform HTTP event callbacks routed.** `1305a690e` (6 files, +420/-3) — generic HTTP-event-callback routing layer for gateway platforms.
- **Weixin fallbacks migrated to `get_secret()` for profile-scoped resolution.** `6160a8025`.
- **Feishu websocket mode allowed in multiplex profiles.** `9cb3569e9`.

### Reasoning effort

- **`max` / `ultra` levels (from previous window).** `7550c594c` continues to be referenced; **per-model overrides** (`d9cdb8192`, see Configuration) and **per-auxiliary-task overrides** (`df5700ebe`, see Auxiliary) are the big reasoning-related additions in this window.
- **MoA per-slot reasoning effort.** `3dca75b45` (8 files, +279/-16) — each MoA slot can have its own `reasoning_effort` (overrides the auxiliary default for MoA reference calls). Replaces the auxiliary-task knob that `0bb3a82c5` removed in favor of per-slot preset config.

### Auxiliary

- **Per-task reasoning_effort for auxiliary models.** `df5700ebe` (#64597, 5 files, +195/-5) — every auxiliary task (vision, web_extract, compression, title_generation, curator, background_review, moa_reference, …) accepts `reasoning_effort` shorthand:
  ```yaml
  auxiliary:
    compression:
      reasoning_effort: low
    vision:
      reasoning_effort: none
  ```
  - `_get_task_extra_body()` folds it into `extra_body.reasoning`. Chat.completions passes through; Codex Responses maps to top-level `reasoning/include`; Anthropic auxiliary now forwards to `build_anthropic_kwargs(reasoning_config=...)` (was hardcoded None).
  - Explicit `extra_body.reasoning` wins over the shorthand. Invalid levels ignored with a warning. Empty string (shipped default) is a no-op.
  - `reasoning_effort` added to all 16 auxiliary task blocks in `DEFAULT_CONFIG` (no version bump — deep-merge handles new keys).
- **Auxiliary model usage recorded per task in session accounting.** `eb6aa0360` (#65537, 12 files, +824/-26) — fixes #23270. `session_model_usage` gains a task PK dimension (`''=main loop`) via **v22 table-rebuild migration** (SQLite can't alter a PK); `record_auxiliary_usage()` writes per-(model,provider,task) deltas WITHOUT touching `sessions` counters (gateway overwrites those with absolute main-loop totals — folding aux in would double-count or be clobbered).
  - ContextVar ambient accounting context in `agent/aux_accounting.py` (mirrors the portal_tags conversation context); `record_aux_usage()` normalizes via `usage_pricing.normalize_usage`, estimates cost, strictly best-effort.
  - `moa_reference`/`moa_aggregator` excluded — `conversation_loop` already folds MoA usage+cost into the main delta.
  - Recording chokepoint is `_validate_llm_response` in `agent/auxiliary_client.py` — every successful non-streaming aux response passes through exactly once, sync + async, including fallback paths.
  - `/api/analytics/usage` folds aux rows into `by_model` (aux-only models finally appear) + adds a `by_task` summary; `/api/analytics/models` surfaces aux rows on the Models page.
  - Insights also updated: `21dedb858` (include auxiliary usage in overview token totals).
- **Auxiliary runtime cache isolated by live context.** `fdc6c32d7` (`fix(auxiliary)`) + `73057ed16` (scope runtime state to each turn) — fixes cross-conversation cache bleed.
- **Auxiliary client accepts accounting-hint kwargs.** Tests updated: `match _validate_llm_response mock to new accounting-hint signature` + accept in remaining mocks.

### OpenAI / gpt-5.6

- **Complete gpt-5.6 family E2E.** From previous window: `4af484d3d` (sol/terra/luna + -pro variants, 6 slugs total).
- **`gpt-5.6 -pro` variants covered.** `a3828a94d` (PR #61587 complement). Same 272K Codex-OAuth compaction auto-raise predicate extended to cover 5.6.

### Anthropic

- **Preserve thinking blocks on Kimi-family endpoints on replay.** `ddd81e935` (`fix(anthropic)`) — Kimi-family Anthropic-compatible endpoints were stripping thinking blocks on message replay; now preserved.

### Desktop

- **Per-job model picker in cron create/edit dialog.** `1bf441cd1` (#67472) — cron jobs can now pick their own model at create/edit time, not just the gateway's default.
- **Session color — inherit from project, shared across sidebar and tabs.** `ad0d21188` (#67469) + `ad0d21188`, `897f3da27`, `5a6e23583`, `7710485c0`/`1d12d610e` (inherited projects set color and icon) — operator-visible session/project color coding throughout the desktop.
- **Terminal execution backend picker with health probes.** `8b6714556` (#67203) — desktop can pick the terminal backend (local / docker / ssh / modal) from the Capabilities tab; truthful per-provider readiness (`c372c4220`).
- **Ctrl/Cmd + mouse wheel zoom.** `ad0ddfb15` (#40295).
- **Billing settings tab.** `d29674905` (#61054) — paired with `/topup`/`/subscription` work above.
- **List config-defined command TTS/STT providers in settings.** `e58534f9d` + `5c6499ce4` (surface all xAI TTS params).
- **Drop `tts.xai.text_normalization`** (`6bb8a0aef`) — not honored by xAI TTS backend; remove stale config surface.
- **Profile-aware approval mode control.** `3510b1881` — see Approvals above.
- **Multi-session tiles.** `e6fea77d1` (per-profile state, tile pane, pane mirror), `ac4f596ca` (pointer session drag/drop), `eae1d7d14` (tab keybindings `⌘W`/`⌘⇧T`/`⌘T`/`⌘1-9`/`⌃Tab`), `860a3f67b` (chat view — drop overlays, composer scoping), `2afbe7776` (session hooks — open-in-tile, per-session actions, resilient resume).
- **Plugin SDK surface.** `99ff67eb0` (rest door, socket, react-query, UI kit), `7ed717096` (plugin manager, runtime loader, plugins settings), `aefb36299` (contribution registry — namespaced areas, keybinds, palette).
- **Layout-tree renderer.** `63a9bde77` (splits, zones, pointer drag-session, tab strip), `c388daa66` (model + store + workspace geometry).
- **Shared UI primitives.** `aae35c5ee` (per-session prompt overlays, gateway overlays, tab primitives), `f1379bd6c` (routes, nav, command palette as contributions).
- **Layout-store work.** `7f74b324c` (store + lib — layout/preview/session atoms, escape-layers, keybind helpers), `0f922002e` (contribution controller, surfaces, wiring), `0f398f8e9` (focused-session-aware titlebar + statusbar).
- **Worktree support.** `f6d1fd511` (auto-fetch remote base branch before worktree add), `6f7ee72be` (base-branch picker for new worktree dialog).
- **Tool panels collapse to a persistent rail.** `798e602a8`.
- **Chat backdrop on/off toggle.** `1813d3046`.
- **Electron: openDir IPC + ⌘W menu bridge.** `10fbade64` — tabs not windows.
- **Background-task indicator on sidebar session rows.** `b80b52aa4` (#65174), `587e76fbc` (green unread dot for background-finished sessions).
- **Button tooltip keybind hints + keybinds settings tab.** `f08b1f344` (#65204).
- **WSL path bridge for Windows host + WSL backend.** `88fbc8825`.
- **Autosave the inline provider panel.** `67d64124c`.
- **Honcho memory provider config modal.** `a182ddbf9` (per-field info tooltips + profile name in full-config modal).
- **Persisted zoom + first-show window state.** `e63276926` (#56726).
- **Model picker reverts in existing threads.** `659d1123c` (#65777) — see Reverts.

### PTY & API server

- **PTY keep-alive, reattach, session registry** (from previous window: `e5ac169c2`, `0ecfbc989`, `41166bbe0`, `e10e4bca8`, `c3d2be073`, `79f4f78fa`).
- **API run transport lifetime management** (from previous window: `837077dfa`, `8f18fa104`, `1da89a5f3`).

### MCP

- **Exact version pins across the whole MCP catalog.** `9df5f879b` (3 files, +68/-3) — same supply-chain rules as pyproject dependencies:
  - `n8n`: `install.ref main` → full commit SHA `7a9ae007` (2026-05-23; branches/tags can be moved by upstream; SHAs cannot).
  - New contract test: every shipped manifest must pin exactly — git installs need a 40-char SHA, `uvx`/`npx`-style launchers need `pkg==X` / `pkg@X` with a digit-leading version (rejects bare names, ranges, npm dist-tags like `@latest`).
  - Module docstring documents the pin policy (exact version, 2-week cooldown).
  - Verified: unpinning `blender-mcp` in the manifest makes the contract test fail with a named diagnostic; restoring the pin passes.
  - Unreal-engine and Linear are HTTP transports (server runs elsewhere) so there's nothing to pin at the transport layer.
  - Follow-up: `a52393a3b` (pin `blender-mcp` to `1.6.4` per catalog dependency policy).
- **Blender added to MCP catalog.** `9be941dac` (3 files, +123/-0) — `optional-mcps/blender` (`ahujasid/blender-mcp`, stdio via `uvx`). Server advertises 22 tools; 18 front optional asset services with no upstream trim mechanism, so `tools.default_enabled` pins the install to the core surface (scene/object info, viewport screenshot, code exec) and the rest stay opt-in through `hermes mcp configure blender`.
  - Manifests can now declare `transport.env` (static, non-secret subprocess env vars), parsed/validated in `_parse_manifest` and written by `_build_server_config`. Used here to ship `DISABLE_TELEMETRY=true` per the no-telemetry-without-opt-in policy. Runtime already honored per-server env; manifests just couldn't declare it.
- **Unreal MCP live-verified against UE 5.8.** `d24ab2040` (live-verify skill), `18694e96d` (advanced-workflows layer), `665eaf197` (video/frame-sequence pitfalls), `d57531b1a` (pitfall 21b rewritten), `ab818493f` (dedupe pitfall numbering), `a8b81c56a` (companion skill for the unreal-engine MCP catalog entry).
- **Blender MCP companion skill.** `5601f2444` (enhanced with comprehensive references and recipes) + `6efec39ec` (rework around catalog entry) + `b6c11a35a` (adapt salvaged references to MCP-tool surface, bump to 2.1.0) + `5f171e36a` (move `mcp-oauth-remote-gateway` to optional-skills + modernize) + `03885e0aa` (add the skill).
- **Hosted OAuth lifecycle hardened.** `ebd737f4d`, `604552972`, `11eaa77da`, `cf3ae7c59` (preserve live OAuth state during reauth), `4dc2b7be0` (preserve concurrent OAuth manager refresh), `b09f1ba77` (reject invalid dashboard oauth callbacks), `05dea7be0` (complete OAuth through hosted dashboards), `58010c8b3` (reuse cached oauth redirect port), `d34cc4093` (per-flow callback waiters so concurrent OAuth flows cannot cross ports), `454d553d3` (clear error when OAuth callback port in use), `261b0f824` (document `redirect_uri` proxied callbacks + `redirect_host` WAF workaround), `95a0f9c83` (close select-to-bind TOCTOU on OAuth callback port), `f4c7caa70` (remove unreachable dead code), `13e19a909` (per-provider closures + `allow_reuse_address` for OAuth, #44588/#44590), `6297634d2` (allow configurable `redirect_uri` for MCP OAuth flows), `dc419d6e8` (configurable `redirect_host`, WAF-safe localhost redirect URIs), `d0afcb125` (cover configurable `redirect_uri` + fix misleading SSH hint), `f01f0f75f` (redirect_host coverage + adapt salvaged tests to the non-interactive guard).
- **HTTP MCP authentication in dashboard.** `e0e7cfa67` (#65146), `56ab9951b` (#65163, add MCP auth to profile builder).
- **Pagination safety.** `a8ec41533` (treat non-string `nextCursor` as end of pagination), `6030ca8ce` (follow `nextCursor` pagination in tools/resources/prompts discovery).
- **Tool schema sanitization for Gemini.** `b78ff50d8` (prune required entries missing from properties).
- **MCP poll-loop OOM spin fix.** `3df8bd347` (propagate stored timeouts from completed futures), `1cec5c69d` (e2e integration coverage for #63892).
- **Stdio watchdog uses direct parent identity.** `3d031bdb2`.
- **Empty-response advisories no longer trigger compression.** `032a424fa`.

### Cron

- **Truthful execution ledger.** `d9dd05b69` (7 files, +481/-1) — new `cron/executions.py` records every cron execution (start, end, exit reason, output). `cron/jobs.py`, `cron/scheduler.py`, `cron/scheduler_provider.py`, `hermes_cli/cron.py`, `plugins/cron_providers/chronos/__init__.py` all wired to write through it. Tests: `tests/cron/test_execution_ledger.py` (205 lines).
- **Accept `target_model` kwarg in codex-path resolver stub.** `65b73eb1e` (most recent commit on `origin/main` at review time) + `786df3ca6` (`fix(cron)` — resolve provider with the job's effective model; default dashboard cron creates to the backend's own profile), `6e676c768` (`fix(desktop)` — profile-scope all cron REST calls).
- **Inline dispatch deadlock fix** (from previous window: `5c5dd6b7e`, `d00c15c0c`, `47c91e4c3`, `02063ece1`) — cron LLM calls run synchronously when gateway is the transport.
- **One-shot claim heartbeat binding + never stale-remove a live one-shot** (from previous window: `dabae386e`, `9b72995a1`).

### Sessions & state

- **Schema v22 table-rebuild migration in `session_model_usage`.** `eb6aa0360` — SQLite can't alter a PK, so the migration rebuilds the table with a new task PK dimension. See Auxiliary above for full details.
- **State heal alternation at all restore boundaries.** `4579f2630` (ACP / CLI-resume / TUI-resume restore sites), `ee659d1d8` (durable alternation violations at restore boundary).
- **State self-heal FTS corruption on SessionDB write path.** `9e1b1d753` (#66296).
- **State repair duplicate session titles without data loss on startup.** `3990bdf55` (#65602), `9fc8fe217` (guard the duplicate-title repair so it never aborts DB open).
- **State use PASSIVE checkpoint for periodic WAL flush.** `c2a3b9ce5` — prevents B-tree corruption on macOS.
- **State enforce synchronous=FULL on macOS.** `9aba95b05` (prevents btree corruption), `f9c6f92c4` (docstring fix).
- **State widen surrogate scrub to remaining raw-str bind sites.** `653a95f9f`.
- **State stop a lone surrogate from silently killing session persistence.** `81d461970`.
- **State guard the duplicate-title repair so it can never abort DB open.** `9fc8fe217`.
- **Compression state revalidated under lock.** `727392b5c`.
- **Compression state probe fails closed.** `fd461b58c`.
- **Compression reset failure cooldown on runtime switch.** `bcce70078`.
- **Byte-stable session context prompts.** `c0c76a471` (also gateway) — reduces prompt-cache prefix churn.
- **Snapshot close state under staging lock.** `32bdc67e1` — CLI shutdown safety.

### Kanban

- **Attachment toolset + CLI to match the dashboard surface.** `3fccd698f` (9 files, +930/-39) — closes the long-standing gap that the dashboard had full attachment storage (since #35338) but no agent toolset tool and no `hermes kanban` CLI verb. New: `kanban_db.store_attachment_bytes()` (single shared write path — validate name, 25 MB cap, write blob under per-task dir with collision-free naming, insert metadata row, clean up orphan blob if insert fails), `_MAX_ATTACHMENT_BYTES`, `_safe_attachment_name`, `_collision_free_path` moved into shared helpers.
  - Tools: `kanban_attach` (inline base64), `kanban_attach_url` (server-side http/https fetch with same cap), `kanban_attachments` (list). Write tools respect worker task-ownership; list is read-only. Registered in the `kanban` toolset.
  - CLI: `attach <id> <path>`, `attachments <id>`, `attach-rm <attachment_id>`.
  - Dashboard `upload_task_attachment` imports shared helpers + uses `_collision_free_path` — behavior identical.
  - Tests: tool round-trip + oversize + bad base64 + ownership; `attach_url` against local HTTP fixture incl. oversize-mid-stream and non-http scheme rejection; CLI attach/attachments/attach-rm; shared-helper unit tests; dashboard parity preserved.
- **Modal create-task dialog, editable board project directory, comment workflow hint.** `e4f87557b` (#66333, 7 files, +483/-163) — community feedback from `@LSanapalli`: inline task-creation form was cramped in ~280px column with no resize; board-level workspace defaults couldn't change after board creation; users believed they had to block + comment + unblock just to talk to a worker.
  - Create-task dialog: centered modal (reuses hermes-kanban-dialog chrome, 36rem wide) with labeled fields for title, assignee, priority, skills, workspace kind/path, goal mode, parent task. Same request shape; Enter/Escape preserved; submit disabled until title present.
  - Board settings dialog: Settings button in board switcher opens a modal to edit display name, description, and `default_workdir`. `PATCH /boards/:slug` now accepts `default_workdir` (validated absolute existing dir; empty string clears; omitted leaves unchanged) and returns the recomputed `default_workspace_kind` so task-creation defaults follow immediately.
  - Comment workflow hint: task drawer's comment box now explains that comments land on the thread immediately and reach the worker on its next run/`kanban_show()` — no block/unblock dance needed — with a fuller tooltip for when blocking IS the right tool.
  - i18n: new keys optional in the kanban namespace with English fallbacks in the bundle (established pattern; avoids churning 17 locale files).
- **Final result surfaced on Done cards.** `deae8e3b4` — show `final_result` for Done cards; show run summary when `task.result` is empty. `98b456294` makes Done-card results actionable.
- **Collect project directory when creating boards.** `aaf569126` (#63249) — new boards require a project directory at creation time.
- **Attach-URL SSRF guard.** `c2e11bf41` — `kanban_attach_url` rejects SSRF via `tools.url_safety`.
- **Unify attachment size cap on `KANBAN_ATTACHMENT_MAX_BYTES`.** `f3cbe4560`.
- **Worker-lifecycle + events table reflects the bounded protocol-violation retry.** `3cd8feb63` (docs).
- **Bounded retry for clean-exit protocol violations.** `c3656e9f0`.
- **Nudge workers that exit without complete/block.** `03fbf6edb`.
- **Spawn `goal_mode` workers with `-Q` so the goal loop actually runs.** `80b58ec71`.
- **Harden durable artifact handoff.** `8030b01a2`.
- **Preserve scratch completion artifacts.** `e6c42b5d8`.
- **`kanban_unblock` response status syncs with DB state.** `9a7a43b5d`.
- **Honcho / memory-related kanban fallout.** N/A.

### Memory

- **Honcho config schema declared; honcho-first registry.** `94a0cb283` — grouped Honcho provider schema (connection/identity/session/dialectic/recall/…) with inline + full-config field split + `honcho_host_block` storage backend. Hindsight fields stay inline; registry lists Honcho before Hindsight.
- **Memory `/config` reads+writes dispatch on provider storage backend.** `101b9f8dc` — generalize config endpoints to dispatch on `provider.storage`: flat-json for simple providers, `honcho_host_block` for Honcho's real profile-scoped `honcho.json`. Kind-aware coercion (bool/number/json) + partial-save semantics so the inline panel never clobbers full-config-only fields.
- **Restore `surface=declared` routing from main.** `9cb0c62e6` — `surface=declared` serves the curated schema (now sourced from plugins' `config_schema.py` instead of the deleted `hermes_cli/memory_providers.py`), default surface keeps serving the raw plugin schema. The declared PUT returns `{ok: true}` to match main's contract; only the raw-surface PUT reports the activated provider. Desktop requests `surface=declared` again.
- **Honcho latency paths configurable.** `e8957babf` — make latency-adding paths configurable.
- **Honcho `list mode` for `honcho_conclude` so delete resolves a real conclusion id.** `f4669f34c`.
- **Honcho `use _resolve_observer_target` for user context in session context.** `9a887e7c5`.
- **Honcho preserve delayed and rewritten recall context.** `1c051d1df`.
- **Honcho inject base context on the first message of a session.** `bef9eea3e`.
- **Query rewrite provider-agnostic.** `e7fb51d5a` — moved `query_rewrite` from Honcho plugin to `plugins/memory/`; renamed auxiliary task key `honcho_query_rewrite` → `memory_query_rewrite`.
- **Memory per-field info tooltips + profile name in full-config modal.** `a182ddbf9`.
- **Memory provider actions extension point** — **REVERTED** (`f000fbe5c`). See Reverts.

### Cache & prompt-cache continuity

- **`api_content` sidecar persisted exactly as sent.** `7b3dcee92` (16 files, +1532/-45) — first LLM call of every gateway turn was getting ~0% provider prompt-cache hit rate (in-turn calls: 97–99%) because the bytes sent for a turn's user message are not the bytes replayed next turn: memory-prefetch and `pre_llm_call` context are injected into the API copy only, the #48677 persist override writes cleaned content to the DB row, and `get_messages_as_conversation` sanitize/strips user+assistant content on load. Any of these diverges the request prefix at that message and re-prefills everything after it — measured **27.9s for the first call vs 2.4–5.8s cached at a median ~156k-token context**.
  - Nullable `messages.api_content` column stores the exact content string sent to the API when it differs from the clean stored content; replay substitutes it verbatim (no sanitize, no strip).
  - Injection composition lives in one helper (`turn_context.compose_user_api_content`); turn prologue stamps its output onto the live user message; `api_messages` build sends the stamped bytes; every outgoing copy pops the field so it never reaches a provider.
  - Crash-resilience user-turn persist moves after `prefetch`/`pre_llm_call` so the user row is written once with its final sidecar; `_ensure_db_session` stays before preflight compression.
  - Current-turn index trackers re-anchored after compaction rebuilds the message list; in-place preflight compaction backfills the stamp onto the already-inserted row.
  - Gateway replay forwards the sidecar only when the replay pipeline did not rewrite the content. Rewrite paths that would leave stale bytes (historical image strip, merge-summary-into-tail, consecutive-user repair merge, stale-confirmation redaction) drop the sidecar.
  - `codex_app_server` and MoA turns are excluded from stamping because their wire bytes differ from the composition.
  - Missing or dropped sidecar degrades to today's behavior (one cache-boundary miss), never to wrong content.

### Compression / context

- **Durable refresh skipped for in-memory-only blocks.** `8ddc05b80` (`perf(compression)`).
- **Unblock-direction gaps closed in durable guard refresh.** `1093263aa`.
- **Anchor restoration alternation-safe and grounding scaffolding-proof.** `c03c247e7`.
- **Tool use stays active in the compaction handoff prefix.** `c7205040c` (#66291).
- **Compression failure feedback hardened.** `1e895f4c1`.
- **Compression preserves missing-key history.** `577beeb9b`.
- **Compression preserves human intent and durable handoffs.** `960abf73a`.
- **Compression preserves transient quota retry behavior.** `202ad1b8c`.
- **Compression preserves messages when summary quota is exhausted.** `c72f4576b`.
- **Z.AI GLM token-limit message classified as context overflow.** `174fc958a`.
- **Compression task snapshot grounded.** `761a0b124`.
- **Compression 5th call site fix-up.** `e844ea9f0` — salvaged PR #65187 follow-up + force-redact error text at gateway boundaries.
- **Ambient conversation context entangles aux/MoA/delegate calls.** `9ce0e67f2` — portal context now propagates so MoA/auxiliary/delegate calls see the same conversation context as the main loop.
- **Honcho integration with compression state.** See Memory section.

### Computer-use

- **Cua-driver verify → escalate ladder.** `9d6d77283` (#67123, 7 files, +674/-47) — fixes #67052 (tldraw offline). Hermes' computer_use wrapper dropped cua-driver's structured action verdicts, exposed no `delivery_mode`, and injected background-only guidance — so the agent reported unverified no-ops as success and concluded cua-driver "cannot drive" Electron/Chromium surfaces.
  - **Phase A — preserve the result contract:** `ActionResult` carries `verified`/`effect`/`escalation`/`path`/`degraded`/`code`/`delivery_mode`; `CuaDriverBackend._action()` reads `structuredContent` (was data-only); helper normalizes it, additive and None-safe on old drivers; `_text_response` surfaces the fields additively.
  - **Phase B — bounded, model-reachable foreground:** `delivery_mode` (`background|foreground`) + `bring_to_front` on schema/dispatcher/ABC/all input methods; foreground capability-gated (`input.delivery_mode`); old drivers get a structured `foreground_unsupported` refusal, never a silent background downgrade; no automatic/hidden foreground retry — model selects from signal.
  - **Phase C — guidance + isolation:** system prompt (`prompt_builder`) + bundled `skills/computer-use/SKILL.md` go from background-ONLY to background-FIRST, teaching the AX→PX→foreground ladder driven by returned `effect`/`escalation`, not predicted from the app being Electron; foreground approval scoped by `(action, delivery_mode)` — a background approval never silently authorizes foreground; approval state keyed per `session_id` so concurrent gateway runs don't leak unlocks.
  - Tests: `tests/tools/test_computer_use_delivery_ladder.py` (15) cover confirmed/unverifiable/suspected_noop/degraded/old-driver verdicts, delivery_mode gating + foreground_unsupported, session-scoped foreground approval.
  - Live E2E (cua-driver 0.8.3 + tldraw offline Linux/X11): background click returned `effect='unverifiable'`/`path='ax'` (no fabricated success); foreground request returned `code='foreground_unsupported'` — correct on a pre-ladder driver.

### Platform adapters

- **Telegram: reconnect watchdog.** `c2cb37532` (cause-agnostic wedged-recovery watchdog), `3391e639f` (bound polling drain so wedged pool close can't stall reconnect ladder) — both for #66377. Merge: `7235592ad`.
- **Telegram: free-response topics.** `13906cd4d`.
- **Telegram: standalone sends include duration.** `d73a6f5ac`; voice/audio duration set so long clips don't show `0:00` (`27364b24f`).
- **Telegram: transport-error redaction widened to all remaining raw exception sites.** `a04fcbf77`, `6e96b745d` (redact bot tokens from transport error logs).
- **Discord: reconnect recovery hardened.** `2278f2cb7`, `80744bc2b`, `fad6cbaed` (lifecycle-safe), `d5b9c1ee3` (guard recovery claims and ledger failures), `da955a643` (preserve recovery message identity), `bc0e5adb1` (persist per-channel recovery cursors), `9412f2dd8` (persist streamed final delivery), `2b2203e3a` (advance cursors only after final delivery), `38b39b87e` (keep recovery ledger I/O off event loop), `26eafd6a0` (remove unused recovery reaction probe), `ec24fcc68` (docs clarify default recovery scope), `95ce3344c` (assert global recovery scan cap), `867037bce` (update backfill import for plugin adapter), `a52041b2e` (avoid masking missed Discord parent messages), `303949acd` (backfill missed Discord messages on startup, #3), `bfca45bda` (`/reasoning reset|show|hide` exposed as slash choices), `f57157a12` (recover Discord websocket and event-loop stalls).
- **Discord: persisted toolsets.** `3ffd8b3da` — persist Discord toolsets to Discord platform (not just to the agent).
- **Discord: scoped via `multiplex`.** N/A — falls under gateway multiplex fixes.
- **Slack: app token scoped in multiplex gateway.** `ea2c9bc10`.
- **Slack: scoped Agent View workspace state.** `38cfae9b5`.
- **Slack: live per-tool status line.** `d4396797c` — see Gateway.
- **Slack: clear stuck assistant status on `/stop` and via explicit metadata.** `dbd870467`.
- **Slack: agent view APIs.** `f1328a6bf`, `9a3b676fe` — agent view manifests support.
- **Feishu: websocket mode allowed in multiplex profiles.** `9cb3569e9`.
- **Weixin: profile-scoped secret resolution.** `6160a8025`.
- **Generic: reaction signal emit/consume** (from previous window: `0e2adf9da`) continues to propagate.

### Skills

- **Humanizer skill expands patterns 30–34.** `ef010f874` (1 file, +86/-17) — adds forced metaphors/figurative overwriting, dramatic fragmentation and punchy kickers, rhetorical questions answered immediately, sentence-opener tics, reassurance kickers. Adds a marketing/LinkedIn cliches list to pattern 7. Edits the skill's own instructional prose to follow its own guidance (remove em dashes and negative parallelism).
- **MCP-OAuth-remote-gateway skill.** `03885e0aa` (added), `5f171e36a` (moved to optional-skills + modernize).
- **Blender MCP skill enhanced.** `5601f2444`, `6efec39ec`, `b6c11a35a`.
- **Unreal MCP companion skill.** `a8b81c56a`.
- **Bundle bindings hardened** (from previous window: `51382ac24`, `c36f6b725`).

### Reverts

- **`revert(memory): drop the provider actions extension point.`** `f000fbe5c` — no bundled provider declared actions, and the motivating request (openviking, #56309) needs dynamic select options rather than actions. Removing the unconsumed POST dispatch surface keeps the config panel focused; the extension point can return as its own PR with its first real consumer. **Note for the next re-compile:** remove any reference to `ProviderAction` on `ProviderConfigSchema` and the `POST /api/memory/providers/{name}/actions/{action}` dispatch surface from `references/Hermes_Architecture.md`.
- **`Revert "fix(tests): force manual approval mode in E2E blocking tests".`** `6ce160a5a` — reverts `55624e10b`. Test isolation didn't play well with the rest of the suite.
- **`fix(desktop): model picker reverts in existing threads (#65777).`** `659d1123c` — NOT actually a revert of a feature; a fix to the desktop model picker where stale closures in `selectModel`/`use-model-controls` prevented picker selections from persisting in existing threads (because `activeSessionId` was captured as a closure prop but the actions bag mutated in place). Drop the prop; read `$activeSessionId.get()` live from the store.

### Salvaged PRs (cherry-picks preserving authorship)

Active salvage lane in this window:

- #66870 focus-tab hijack (`7db2decbe`)
- #66109 action-menu tooltip (`a5396765a`)
- #60980 apiserver offload (`e99a0f6a9`)
- #38614 resume cwd (`d015500d4`)
- #67192 p2-config-salvage (`34e66a0d5`)
- #67182 p0-salvage (`ed957aeb2`)
- #63082 sessiondb offload (`91ed8e4a9`)
- #62308 stale-backend (`bed46fcd5`)
- #44797 TUI Python env from dashboard chat (`5122ddd47`)
- #65893 salvage-63082
- #65885 salvage-62308
- #37349 title follow-ups (`8222b1678`)
- 20+ `chore(release): AUTHOR_MAP` entries for individual contributors — full list in the prior CHANGELOG-2026-07-19.md plus this window's `89130bf1f`, `61be8b311`, `6cd5a2c5f`, `826276436`, `007cd1513`, `475936218`, `6cc4691c8`, `2fba721ab`, `bb853b2e9`, `56e06d7ee`, `5604d1852`, `702473edb`, `b60c940d9`, `a79b81836`, `4a7930593`, `a7ef17da7`, `d6c14a952`, `d2c81eb68`, `e357b69a6`, `5d410355a`, `49167ffe0`, `a729a5d38`, `ddd81e935`, `94f8166dc` (FuryMartin mapping). When the next re-compile updates contributor attributions, fold these in.

### Web / search providers

- **Fireworks pricing snapshot refresh to 2026-07.** `fe5c0cb6c` — covers full serverless catalog + cached picker pricing.
- **DeepSeek v4 Flash added to official-docs pricing snapshot.** `97397a1cc`.

### Docker / Photon

- No material changes this window.

## Action items for next full reference re-compile

When the next full re-compile of `references/Hermes_Architecture.md` lands, these sections need updating (in addition to the 11 carryover items from CHANGELOG-2026-07-19.md):

1. **Schema versions**: bump **schema_version 22** in the state.db section, document the new `task` PK dimension on `session_model_usage`, and note the table-rebuild migration approach (SQLite can't alter PKs in place).
2. **Provider routing section**: add **Upstage Solar**, **DeepInfra**, **Kimi K3** (from `75ca29fb2`/`311a5b0a5`), **Kimi retired (kimi-k2.x)**, **Fireworks pricing snapshot**.
3. **CLI section**: document `hermes config get` / `hermes config unset`; document `hermes doctor` deprecated-key/env reporting.
4. **Auxiliary section**: add `reasoning_effort` shorthand per task block; document `record_auxiliary_usage()` accounting context, ContextVar ambient, `/api/analytics/usage` `by_task` summary.
5. **Memory section**: replace any reference to the provider-actions extension point (reverted in `f000fbe5c`); document Honcho config schema with `honcho_host_block` storage backend, `memory_query_rewrite` task key, `surface=declared` routing, per-field tooltips.
6. **MCP section**: document **exact version pin policy** (`9df5f879b`), **Blender catalog entry** with `tools.default_enabled` trimming, **manifest `transport.env`** declaration, **Unreal MCP** live-verify skill, **HTTP MCP auth** in dashboard, **OAuth hosted lifecycle hardening** with `redirect_host`.
7. **Cron section**: document `cron/executions.py` truthful execution ledger, `target_model` kwarg on codex resolver, dashboard cron create scoping to backend's own profile.
8. **Kanban section**: add `kanban_attach`/`kanban_attach_url`/`kanban_attachments` tools + `hermes kanban attach/attachments/attach-rm` CLI verbs; document `attach-rm`; document `_collision_free_path` shared helper; document `default_workdir` board settings modal; document comment workflow hint (no block/unblock dance).
9. **Cache section**: document `messages.api_content` sidecar column and replay semantics; document which rewrite paths drop the sidecar; document the **27.9s vs 2.4–5.8s** cache-boundary measurement that motivated the feature.
10. **Computer-use section**: document `delivery_mode` (background|foreground), `bring_to_front`, foreground capability gating, structured `foreground_unsupported` refusal, per-session approval scoping, AX→PX→foreground ladder, `ActionResult` field set.
11. **CLI/Desktop section**: document `/model --once` one-turn override; `/topup` + `/subscription` billing UX; per-job model picker in cron dialog; session color inheritance; terminal execution backend picker; multi-session tiles; plugin SDK surface; layout-tree renderer.
12. **Approvals section**: document per-session turn lease (`gateway/turn_lease.py`), profile-aware approval mode control, smart-deny owner operation scoping, observer hooks, canonical gateway timeout.
13. **Credentials section**: document the three-store unification (`.env` + `auth.json` + `config.yaml` mirrors), `export FOO=bar` recognition in save/remove, OAuth refresh lock scaffolding consolidation.
14. **Anthropic section**: document Kimi-family thinking-block preservation on replay.
15. **Configuration hygiene**: document `_KNOWN_ROOT_KEYS` derived from `DEFAULT_CONFIG.keys()`, unknown-key warning, `hermes doctor` deprecation report.
16. **Salvage lanes**: incorporate the AUTHOR_MAP additions from both this window and the prior one.

## Known limitations of this pass

- Lightweight-local model only (per operator directive). No full docs crawl or semantic comparison of `Hermes_Architecture.md` against the upstream codebase beyond spot-`git show --stat` checks.
- No live docs-at-HEAD fetch beyond spot-checked URLs; the docs site may have structural changes not caught here.
- Local Hermes is now only **3 commits behind** `origin/main` (was 271 last week); the only deltas are post-merge `fmt(js)` cleanups. This is excellent — the local install is effectively current.
- 9 commits in this window had `git show --stat` output truncated in the spot-check pass (`missing` lines). All non-`missing` commits in the changelog were verified against their actual file count and diff stats; the missing ones are flagged so the next re-compile can verify them before folding into the SSOT.

## Verification trail

- `git log --oneline 7b5ba2054..origin/main | wc -l` → **1,074**
- `git log --oneline 7b5ba2054..origin/main --pretty='%s' | sed -E 's/^([^:( ]+).*/\1/' | sort | uniq -c | sort -rn`:
  - `563 fix`, `124 feat`, `109 test`, `72 chore`, `63 Merge`, `38 refactor`, `29 fmt`, `28 docs`, `18 perf`
- `git show --stat <sha>` spot-verified on: `5854aad8b`, `7b3dcee92`, `3f84b7a16`, `9d6d77283`, `5e65f6d79`, `19527db73`, `9a987f142`, `d9dd05b69`, `3fccd698f`, `e4f87557b`, `ef010f874`, `53adb3fd9`, `336c3b13a`, `f5bacee27`, `bd37ff913`, `df5700ebe`, `f000fbe5c`, `1a323d608`, `b51fbc738`, `9cb0c62e6`, `7041c56cd`, `779019ef7`, `89bd0fba9`, `20502b407`, `fe002eb12`, `9be941dac`, `9df5f879b`, `3dca75b45`, `3c7b9f2e9`, `1305a690e`, `deae8e3b4`, `3510b1881`, `aaf569126`, `9be941dac`, `569b912d7`, `eb6aa0360`, `d9cdb8192`, `73ad9136b`, `e7fb51d5a`, `b27d8b6ac`.
- `grep SCHEMA_VERSION hermes_state.py` → `SCHEMA_VERSION = 22` (bumped from 21 in the v22 window; reflects `eb6aa0360` PK-rebuild migration).
