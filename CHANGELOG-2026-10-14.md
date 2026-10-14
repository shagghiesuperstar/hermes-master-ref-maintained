# CHANGELOG 2026-10-14 — Hermes Architecture Reference Drift Review

## Anchors

- **Local Hermes:** v0.22.0 (2026.9.24) @ e06df9c2ea
- **Upstream HEAD:** 0e427d788d (main)
- **Delta:** +3,852 commits since 2026-10-04 upstream snapshot (8d5e3e4121)
- **Prior SSOT anchor:** 4f22543509 (last fully-synced 2026-10-04)
- **Full span since SSOT:** +25,991 commits (upstream is 119 commits ahead of local)

## Summary

Upstream has advanced 3,852 commits in the past 10 days (since the last drift review on 2026-10-04). The dominant change clusters are:

1. **Plugin catalog management** — ~1,500+ commits to `plugin-catalog/*.yaml` and `contributors/*`. Additions, re-pins, version updates, disclosure reviews, and per-plugin test fixes. Examples: `hermes-insight`, `hermes-lang-ko`, `wait-ledger`, `aux-ledger`, `author-feed`, `openinvest`, `hermes-agent-mixer`, etc.

2. **Release pipeline and CI/CD** — ~300+ commits bundled as "release", "ci", "stable", and "deploy" work. Includes stable-release-dispatch-from-main, update receipts, rollback markers, seed management, Docker release versioning, npm audit findings, and e2e test hardening.

3. **Package/dependency management** — ~150+ commits to `pm/` (Package Manager), `Cargo.toml`, `pyproject.toml`, workspace version pins. Notable: llama.cpp v0.5.0, npm audit advisory remediation (httpx2/httpcore2, source-map-js, WhatsApp bridge), Windows MSI payload Python fixes, uv cache reseeding.

4. **Desktop UI and plugin SDK** — ~120+ commits to `packages/@hermes/desktop/*`, TUI, onboarding flows, connectors page, locale rendering, settings, and session-list components. Themes/accents unification, language packs (Portuguese, Korean, Japanese), dark mode fixes.

5. **Hermes core fixes and optimizations** — ~200+ commits distributed across:
   - Gateway session identity, connectors, IMDS metadata approval gates
   - Cron reliability (durable incident ledger, claim TTL headroom, bot-chat hardening)
   - Compression watermark, lean tail retention, schema rebuilds
   - MCP schema handling, OAuth provider reload-on-first-sight
   - Email subsystem DMARC/SPF/DKIM clause-level parsing
   - Session persistence transcript digest versioning
   - Update refusal for image-managed deployments

6. **Reverts and fixes to prior launches** — ~80+ commits of reverts, primarily:
   - Approval-panel wait-forever behavior (11+ reverts)
   - Skills/Plugins card catalog browser feature (3rd landing, caught regressions)
   - Stale worktree reaping revalidation
   - Various Desktop onboarding/connectivity regressions

## High-Impact Architecture Areas (No SSOT changes yet)

**Schema / State:** No schema version bump since 2026-10-04 (still v30). Session identity columns and message_uid hash additions are queued but not yet live on main.

**Models / Catalogs:** gpt-6.1-sol/-pro/-900k (900K input token path proven at 918K, rejected 931K); Astra Pro; `anthropic/claude-sonnet-5.5` on OpenRouter + Nous Portal; `stealth/space-bunny-alpha` curated; `/fast ultrafast` 6x Standard flag for Astra.

**Plugins / Ecosystem:** Plugin isolation (`host` process per profile), GPU vendor gating (`requires.gpu`), birkin-mnemosyne memory provider, hermes-cron 1.2.0 / hermes-dreaming 2.1.0 / hermes-otel 1.18.2 / hermes-beads 0.1.2.

**Skills:** External mount tier joins `agent` and `bundled` in provenance (usage reports, dashboard, UI). Bundled skill count reduced 26% (github/pdf/OCR absorbed into core).

**i18n:** Pluggable layered language packs (bundled → plugin → user overlay) across core + Desktop + TUI. Portuguese and Korean Desktop packs added.

**CLI:** `hermes model` now shows Local models when PM pins an engine; `hermes status --short` (summary default). `hermes doctor --live` health stats.

**Cron:** Per-job `interpreter` field (Python scripts without install/freeze/restore cycle). Failure-incident ledger with one-alert-per-outage semantics. Pinned model per job. Bot-chat delivery hardening (10+ fixes).

**Gateway / Routing:** Profile-scoped reconnect thresholds, vault RPC profile scope, `route_inbound_messages_to_profiles_by_sender_user_id` (Discord guilds/channels/threads → profiles), identity-first session keys.

**Security:** GitSpawn RCE class closed (GHSA-7x36-8jrh-v4pw), OAuth callback XSS, `HERMES_HOME` write-deny for secret stores, MCP User-Agent headers.

**PM (Package Manager):** llama.cpp v0.5.0 (b11146), Linux arm64 CUDA, libgomp bundled. Install adds libatomic host package step.

**Email:** `authserv_id` required for authenticated-sender mode, fails closed if absent.

## Known Limitations of This Review

- **No full code audit.** 3,852 commits is too large to spot-verify every claim in one pass.
- **Plugin catalog churn dominates.** Most of the signal is library/plugin pins and version updates; core architecture is quieter.
- **Desktop/TUI changes not exhaustively documented.** Onboarding, connectors, i18n work is present but summary level only.
- **Reverts not exhaustively enumerated.** 15+ revert commits exist but full trace would require manual graph analysis.

## Action Items for Next Full Reference Re-compile

When the full re-compile of `references/Hermes_Architecture.md` is undertaken (likely within 1-2 weeks), incorporate these concrete edits:

1. **Configuration & provider routing:**
   - Document per-model capability metadata in plugin system
   - Note OpenRouter OAuth PKCE login flow
   - Add Actual Computer provider to catalog

2. **Schema versions & state.db migrations:**
   - Record schema v30→v31 transition (message_uid hex, absorbed_message_uids)
   - Note message identity columns

3. **Models & Catalogs:**
   - Add gpt-6.1 family (sol, pro, 900k) with token limits (918K proven, 931K rejected)
   - Record `/fast ultrafast` tier for Astra at 6x Standard
   - Add claude-sonnet-5.5 to OpenRouter + Nous Portal entries
   - Add stealth/space-bunny-alpha

4. **CLI / Commands:**
   - Document `hermes model` shows Local models when PM active
   - Document `hermes status --short` (new summary default)
   - Document `hermes doctor --live` with health stats
   - Document `hermes model --once` one-turn override

5. **Cron:**
   - Document per-job `interpreter` field (Python scripts)
   - Document pinned model semantics per job
   - Document failure-incident ledger (one-alert-per-outage, complete-on-success/reopen-on-replay)
   - Document bot-chat delivery hardening (10+ fixes)

6. **Gateway:**
   - Document profile-scoped reconnect thresholds
   - Document route-inbound-by-sender-user-id for multi-platform
   - Document identity-first session key generation

7. **Security:**
   - Document GitSpawn RCE closure (GHSA-7x36-8jrh-v4pw)
   - Document OAuth callback XSS fix
   - Document HERMES_HOME write-deny for secret stores
   - Document MCP User-Agent enforcement

8. **Skills:**
   - Document `'external'` provenance tier
   - Update bundled-skill count (reduced 26%)
   - Note github/pdf/OCR absorption into core

9. **i18n:**
   - Document pluggable layered language packs
   - Document Portuguese + Korean Desktop packs

10. **Package Manager / PM:**
    - Document llama.cpp v0.5.0 (b11146)
    - Document Linux arm64 CUDA + libgomp bundled
    - Document libatomic install step

11. **Email:**
    - Document `authserv_id` requirement
    - Document fail-closed behavior when absent

## Reverts Summary

15+ reverts confirmed since 2026-10-04. Concentrated in:
- Approval-panel wait-forever behavior (11+ distinct reverts)
- Skills/Plugins card catalog browser (3rd landing, regression detection)
- Stale worktree reaping revalidation

No data loss or schema corruption; all reverts are feature regressions caught post-merge.

## Conclusion

Reference SSOT (`references/Hermes_Architecture.md`) remains **untouched per skill protocol**. This changelog is the durable record and scaffold for the next full re-compile. Operator should expect a full reference refresh within 1-2 weeks when upstream stabilizes or a dedicated re-compile slot opens.

Next review scheduled: **2026-10-21** (weekly cadence).
