# Hermes Architecture Reference — Drift Changelog 2026-07-19

## Scope of this review

- **Local Hermes version:** v0.18.2 (2026.7.7.2) — local `570141a39` (+1 carried commit on `fix/cron-update-pin-snapshots`)
- **Upstream HEAD reviewed:** `7b5ba2054` (origin/main)
- **Prior SSOT anchor (last drift review):** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Prior upstream HEAD (last drift review):** `aaeba213d` (CHANGELOG-2026-07-12)
- **Commits between prior upstream and current upstream:** **502** (~38 feat, ~270 fix, ~12 refactor, ~35 test, remainder chore/salvage/docs/merge)
- **Window:** 2026-07-12 → 2026-07-19 (~1 week)
- **Local vs upstream:** local is **271 commits behind** `origin/main`
- **Methodology:** lightweight-local model (per operator directive) + git-log area-bucketing + spot-checks on high-impact commits. Changelog-as-scaffolding per the drift review decision tree (502 > 500 commits, but operator said "lightweight local model only" — default to changelog-as-scaffolding). The main reference (Hermes_Architecture.md) is left untouched.

## High-impact architecture deltas (aaeba213d → 7b5ba2054)

### Configuration & provider routing

- **Fireworks AI added as first-class BYOK provider.** `c97d9a4c0` (#60660) — Full provider plugin with attribution headers, registered in `CANONICAL_PROVIDERS`, alias wiring (`fireworks-ai`, `fw`), PAYG-safe default auxiliary + fallback models. Appears in CLI/web/TUI/desktop pickers.
  - Follow-up fixes: `31152ae10` (align with project policy), `d83cd6f7c` (secure Azure catalog probes).
- **grok-4.5 GA added to model catalog.** `62ada5175` (#60887) — Added to `_XAI_CURATED_EXTRAS`, `_XAI_STATIC_FALLBACK`, context length 500K, reasoning-effort allowlist.
- **Tencent Hy3 Preview → GA swap.** `b64b80213` (#60943) — `tencent/hy3-preview{,:free}` removed, `tencent/hy3` / `tencent/hy3:free` added across `OPENROUTER_MODELS`, `_PROVIDER_MODELS[nous]`, and reasoning-prefix list. `openrouter/owl-alpha` dropped.
- **Per-model token tracking for mid-session model switches.** `cb7f6bbb2` — Agent now tracks token usage per model so compaction and cost display are correct after `/model` changes mid-conversation.
- **Fireworks `extra_headers` wiring.** In `c97d9a4c0` — includes `HTTP-Referer` / `X-Title` attribution headers.

### Approvals & security

- **Smart approvals made DEFAULT.** `62a76bd3d` (#62661) — `approvals.mode: smart` is now the default in `DEFAULT_CONFIG` instead of `approvals.mode: on`. Affects all new installs; existing configs keep their current setting. **Operator impact:** our profile already sets approvals.mode explicitly, but this changes the fallback behavior if the config key is ever absent.
- **Extensive security hardening for credential/catalog provider paths:** `d83cd6f7c`, `1f46145e0`, `92c214603`, `4530a4ca4`, `cf34a1e8c`, `27a1042b1`, `6e75ba7fa` — Azure catalog probes, urllib redirect policies, sanitizer ordering, installed hooks. Bulk credential-leak patching across multiple provider catalogs.

### CLI

- **`sessions list --workspace` filter + Workspace column.** `0c4aed249` — Filter sessions by workspace key; new Workspace column in tabular output.
- **`sessions export --format trace` (HF upload support).** `0e04d1420` (#60507) — New trace format with optional HuggingFace upload for RL/training pipelines.
- **`sessions export` now supports `--format html`, `--format prompt-only`, and full `--prune-filter` set + `--redact`.** `f76899fac`, `acfefa4fd` (#60554).
- **`--no-restore-cwd` flag on resume.** `b5f0e451c` — Restore of working directory on session resume can be suppressed.
- **Pip/Homebrew installs now show unsupported warning.** `4d7f8ade3` (#57225) — The CLI/TUI/desktop surfaces a warning on startup when installed via pip or Homebrew.

### Gateway

- **OIDC client-credentials relay provisioning (NAS/Nous-Portal-free).** `f64e4f4f5` (#60730) — Air-gapped deploy support: `gateway.idp.token_url` (or `GATEWAY_RELAY_IDP_*` env) enables generic OAuth2 `client_credentials` grant against any IdP (e.g. Microsoft Entra ID). The connector reads a `tid` claim off the token for tenant scoping.
- **`GATEWAY_MULTIPLEX_PROFILES` env override.** `75de0057b` (#60589) — Environment-variable override for the multiplex flag.
- **Webhook payload filters subsystem.** `0cf2e39c4` — New `gateway/platforms/webhook_filters.py` (302 lines). Allows filtering webhook payloads before they reach the agent. Configured via `config.yaml`. Docs: `aabfedcac`.
- **MEDIA: caption attached to media bubble.** `709da844b` — `hermes send "MEDIA:/x.png This Caption"` now arrives as a single captioned bubble instead of two messages.
- **Authenticated runtime readiness checks.** `f9728af5e` — Gateway endpoints now verify runtime is truly ready before accepting traffic.
- **Gateway session-store offloaded to thread pool.** `24ea21993` — Offloads `session_store` calls off the event loop via `asyncio.to_thread` to avoid blocking event loop.

### Reasoning effort

- **New effort levels: `max` and `ultra`.** `7550c594c` (#62650) — Two new `/reasoning` levels above `high`. Wired into `agent/anthropic_adapter.py`, `chat_completions.py`, `codex.py`, and desktop settings UI (constants, model-settings, model-edit-submenu). i18n: en, ja.

### OpenAI / gpt-5.6

- **Complete gpt-5.6 family E2E.** `4af484d3d` (sol/terra/luna + -pro variants = 6 slugs total). Key details:
  - Codex OAuth backend now recognizes gpt-5.6* for 272K compaction auto-raise (same predicate as 5.4/5.5, extended via `_is_codex_gpt54_or_gpt55` → now covers 5.6).
  - Full registration in `codex_catalog.py`, native picker, pricing.
  - -pro variants covered: `a3828a94d` (complement PR #61587).
  - Context window on Codex OAuth: 272K (hard cap). Direct-API/OpenRouter: full 1.05M.

### Desktop

- **Hermes Cloud connection mode.** `c101207b9` — Third gateway mode ('local' | 'remote' | 'cloud'). One portal sign-in discovers agents on your account, connects silently via a centralized `modeIsRemoteLike()` predicate.
- **Embeddable Gateway settings panel.** `f8152d232`, `b3bde1fbe` — Soft gateway switch UI component.
- **Structured Fallback Models editor.** `21781d54e`, `bf3667aee` — GUI editor for fallback provider chains.
- **Autosave MoA preset edits.** `29c9dd99a` — Mixture-of-Agents preset edits now persist automatically.
- **Vibe hearts / reaction system.** `3aaf7e387` (TikTok-style particle system), `0e2adf9da` (gateway+CLI emit/consume reaction signal), `422d9da9b` (core affection reaction detector + callback). Cosmetic but a new cross-cutting integration point.
- **TypeScript-ification complete.** `39d09453f` — Entire desktop codebase converted to TypeScript.

### PTY & API server

- **PTY keep-alive, reattach, session registry.** Multi-commit chain: `e5ac169c2` (RingBuffer for keep-alive output), `0ecfbc989` (PTY drain/attach/detach with EOF close), `41166bbe0` (PtySessionRegistry with reap + capacity), `e10e4bca8` (reattach via `?attach=` token), `c3d2be073` (periodic reaper in dashboard), `79f4f78fa` (persist attach token, reconnect on transient close).
- **API run transport lifetime management.** `837077dfa` (stop producers after transport expires), `8f18fa104` (separate run control from stream lifetime), `1da89a5f3` (keep live runs tracked past stream TTL).

### MCP

- **Primarily bug fixes, no new MCP features this window.** Key fixes:
  - Orphan stdio subprocess reaping on reconnect (5089c84db, f99e9f0d2, 086596ca2, 743c116fb)
  - Handshake bound for subprocess/FD leak prevention (1f6836cd8, 4638f3b43)
  - Watchdog wrapping after OSV preflight (86c5febdd)
  - Windows POSIX-only kill guard (838d50495)
  - Doc updates for `idle_timeout_seconds` / `max_lifetime_seconds` recycle keys (a6203839b)

### Cron

- **Inline dispatch to avoid gateway deadlock.** `5c5dd6b7e` (#62151), `d00c15c0c` — Cron LLM calls now run synchronously (inline) when the gateway is the transport, avoiding the deadlock where the gateway waits for itself to respond. Followups: `47c91e4c3` (timeout), `02063ece1` (scope to reported transport).
- **One-shot cron claim heartbeat fixed.** `dabae386e` — Bind claim heartbeats to dispatch owner so orphaned claims don't keep a one-shot locked.
- **Never stale-remove a one-shot whose run is still alive.** `9b72995a1`.

### Sessions

- **`sessions export --format trace`.** `0e04d1420` (#60507) — Exports session as OpenAI-compatible traces. Optional `--hf-upload` to push directly to HuggingFace.
- **Full prune-filter + --redact.** `acfefa4fd` — Sessions export now supports the complete `--prune-filter` set with redaction capability.
- **Format expansion:** `f76899fac` — `--format html` and `--format prompt-only` added.

### Platform adapters

- **Telegram: flood fallback recovery hardened.** `4aa499ff9`, `04898631c` — Telegram adapter now handles flood wait errors more robustly during stream delivery. PTB heartbeat error classification: `97fb9e1f6`.
- **Feishu: Channel signaling SDK + @mention delivery.** `651e632b6`, `949e4cb72` — Added `extra_ua_tags=["channel"]` to FeishuWSClient for group @mention delivery. Docs infographic: `0b0f60bf2`.
- **Discord: troubleshooting docs for silent fail-closed denials.** `db9e3e4ef`.
- **Generic: reaction signal emit/consume.** `0e2adf9da` — Platform adapters can now emit reactions (vibe hearts, etc.) through the gateway signal system.

### Skills

- **Skills: bundle bindings hardened.** `51382ac24`, `c36f6b725` — Skills now bind bundles to exact files and origins; installs track scan provenance. Mitigates a class of path-confusion bugs during skill installation.

### Reverts

- **#62141 reverted #61979 (pool FD release on thread client close).** `5ba2d167b` — The original fix for releasing credential pool FDs on owning-thread client close was reverted because it caused regression in TLS FD recycle/corruption tests. The revert restores previous behavior; the test suite was reduced from 105 lines to 31.

### Web tool configuration

- **Routing profile/preservation fixes for provider routing through Nous Portal.** `3aeaf3755`, `a0032f5f9` — Provider routing through Portal is now forwarded correctly; profile and delegation parity preserved.

## Action items for next full reference re-compile

When the next full re-compile of `references/Hermes_Architecture.md` lands, these sections need updating:

1. **Configuration docs (DEFAULT_CONFIG)**: Update `approvals.mode` default from `on` to `smart`.
2. **Provider routing section**: Add Fireworks AI as preferred provider entry. Add grok-4.5 to xAI subsection. Update Tencent Hy3 from Preview to GA.
3. **Gateway section**: Add OIDC client-credentials relay subsection. Add webhook payload filters. Add MEDIA: caption behavior. Add multiplex env override.
4. **Reasoning section**: Add "max" and "ultra" to the effort-level table.
5. **OpenAI/Codex section**: Add gpt-5.6 entry (6 slugs, 272K compaction auto-raise for Codex OAuth).
6. **Desktop section**: Add Hermes Cloud mode, reaction system, Fallback Models editor, Gateway settings panel.
7. **PTY/API section**: Add keep-alive/reattach/session-registry docs.
8. **Cron section**: Add inline dispatch deadlock avoidance, one-shot heartbeat fix.
9. **Sessions section**: Update export formats (trace, html, prompt-only, prune-filter, redact).
10. **Skills section**: Add bundle binding/scan-provenance behavior.
11. **Platform adapters**: Add Feishu @mention delivery notes.

## Known limitations of this pass

- Lightweight-local model only (per operator directive). No full docs crawl or semantic comparison of the Hermes_Architecture.md SSOT against the upstream codebase.
- No live docs-at-HEAD fetch beyond the root page; the docs site may have structural changes not caught here.
- Local Hermes is 271 commits behind upstream on branch `fix/cron-update-pin-snapshots` — some behavioral changes since v0.18.2 may not be reproducible locally.