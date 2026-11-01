# Hermes Architecture Drift Review — 2026-11-24

## Anchors (canonical)

| Fact | Value |
|---|---|
| Local Hermes commit | `bf0203a794` (v0.22.5, 2026.9.24) |
| Upstream HEAD | `d050956633` |
| Local-behind-upstream count | 1,191 commits |
| Prior SSOT anchor | `4f22543509` (2026-09-25, v0.21.2) |
| **Total delta since prior SSOT** | **32,581 commits** (~60 days elapsed) |
| Review window type | **Changelog-as-scaffolding (massive drift)** |

## Commit type breakdown (4f22543509..d050956633)

- `fix`: 13,605 commits (41.8%) — high-velocity bug remediation
- `refactor`: 4,668 commits (14.3%)
- `test`: 2,934 commits (9.0%)
- `feat`: 1,763 commits (5.4%)
- `chore`: 1,592 commits (4.9%)
- `docs`: 707 commits (2.2%)
- `perf`: 196 commits (0.6%)
- Merges + other: ~7,114 commits (21.8%)

## Dominant themes (architecture-area bucketing)

### Gateway (highest-impact cluster, 200+ fixes)

The gateway subsystem underwent extensive hardening focused on **session lifecycle management**, **routing consistency**, and **state recovery after restart**.

- **Session storage and picker recovery** (`d050956633`, `cf0f210715`, `388575e1f5`, `21c35c6115`) — fixed closed local sessions not persisting in picker after gateway restart; orphan-end detection; sibling-scope binding on teardown.
- **Route binding and API stability** (`4e3e66896d`, `439e874e2f`, `bc629f94e0`) — API binding now replaces history mounts cleanly; routes stay finalized; admission heads settled correctly.
- **Message routing by identity** (`79d53c1710`, `91171b7f0b`, `d23579a1e5`) — subagent notices routed by delegate provenance; automation wakes on route identity not display; config projections dropped when refusing cancels.
- **History mount handling** (`7fe896863c`, `22b297098f`, `690440524b`) — voice and fast reads on history mounts; read-only refusals named consistently; mount promotion fixed.
- **Messaging refusals and retry** (`4d0cdd4f31`, `04fae30eeb`, `6309003fce`) — refused follow-ups marked as dropped not saved; pre-claim refusal kept inside retry; refused heads settled after storage retry.
- **Clarify and context** (`69e9b4039b`, `d4fcea44cd`, `239c9d2628`) — open clarify batches restored on resume; agentless context categories shown; tools pin tolerated when unreadable.
- **Model and reasoning routing** (`6b11a8eca5`, `c9b067feb7`, `f65c227878`, `c289426df0`) — model-only tip reads; context breakdown cache prompt-aware; bare custom provider resolution; reasoning commit supersedes deferred level.

**Action items for re-compile:** Update "Gateway internals and messaging platforms" section with session lifecycle state diagram (closed → picker → restored), route-binding lifecycle (binding → finalized → heap), and the new identity-first routing rules for subagents + automation. Mention the 200+ gateway fix commits and the stabilization focus.

### Desktop and Webapp (new major subsystems)

- **Webapp server from gateway** (`fff24cff13`, `463af9d5c7`, `3c92c1a264`, `b5eb5ee7a8`, `101bff585e`) — Hermes can now serve its own Webapp from the gateway's listener, with opt-in network/custom-URL exposure behind Nous sign-in. CLI support added (`hermes webapp`).
- **Desktop plugin SDK expansion** (40+ `feat(desktop)` commits) — group chat rename/disband/mentions, attachment paste, plugin media rendering, slash-command timing (after onboarding), transcript directives, folder-scoped file access (`ctx.files`), preview-script capability gates + per-origin consent, spellcheck context-menu, board tenant list to badge decorators.
- **Browser-hosted Desktop pieces** (`7ac4cf552c`, `b5eb5ee7a8`) — desktop renderer server carryover from #93508.

**Action items for re-compile:** Add new "Webapp" section after "Desktop / TUI" covering server architecture, Nous sign-in, network/custom-URL modes. Expand "Desktop / TUI" plugin SDK subsection with folder-scoped file access, preview consent gates, chat rename/disband/mentions.

### Plugins (maturation and catalog)

- **Plugin hook system** (`2eec2a7d24`) — `register_hook(..., hold_stream=True)` for `transform_llm_output`, enabling streaming-aware transforms.
- **Plugin-side tools** (`d4e19b9839`) — scoped browser vault backend plugins supported.
- **Plugin file access** (`c421c930fe`) — `ctx.files` folder-scoped file access to desktop.
- **Plugin media rendering** (`6dc1c398e2`) — plugins can render media their own tools produce.
- **Plugin catalog growth** (30+ new `feat(plugin-catalog)` commits) — CoreWeave Inference, Tavily MCP, FlashData, crontab-ref, yandex-disk, game-editor, airlock, ha-sidebar-navigator, bot-activity, skill-lineage, hermes-jev-routing, actvox, genesispay, precleared, pidgeon, Adaptive Inference routing, and more.

**Action items for re-compile:** Update "Plugin system and memory-provider development" section with `hold_stream=True` hook mode. Catalog growth is high-velocity — add a note that ~30 new entries landed in this window; refer to the git log for full list rather than hard-coding them.

### Vault and Security

- **OS-keychain integration** (`7abb9cc0fa`, `d4e19b9839`) — opt-in OS-keychain storage for vault key with verified migration; login backends gated to bundled/verified/official plugins only.
- **Scoped browser vault backends** (`d4e19b9839`) — plugin vault backends supported.

**Action items for re-compile:** Add "Vault / Keychain" subsection to "Secrets" section covering OS-keychain opt-in, migration verification, and login-backend gating. Cross-link to plugin system.

### Platform Manager (PM, llama.cpp runtime)

- **Download progress persistence** (`e5470c9888`) — PM now preserves download progress across interrupted downloads.

**Action items for re-compile:** If PM section exists, add note on progress persistence. Otherwise minor update.

### CLI

- **Webapp command** (`463af9d5c7`) — `hermes webapp` to open gateway's Webapp.

**Action items for re-compile:** Add to CLI commands reference.

### Catalog and Providers

- **CoreWeave Inference provider** (`bfe0a6f553`) — new provider option.

**Action items for re-compile:** Update provider roster.

### Documentation and Internationalization

Multiple i18n commits for web access settings, read-only chat errors, refusal keys across UI surfaces.

**Action items for re-compile:** Note that multi-language support expanded; refer to repo for specific UI elements.

## Known limitations of this pass

1. **No docs fetch** — Official docs at https://hermes-agent.nousresearch.com/docs/ were not re-read for gaps or new sections. The 2026-11-24 landing page shows Hermes is mature across these areas, but specific section updates would require a full re-read of https://hermes-agent.nousresearch.com/docs/.
2. **High-velocity fix cluster** — 13,605 fix commits over 60 days indicates sustained bug remediation but makes granular per-fix listing impractical. Focus was on architecture-area bucketing (gateway, desktop, plugins, vault, PM, catalog).
3. **Schema and state.db** — No schema-version bump detected in commit messages. SCHEMA_VERSION likely stays at v31 (per prior CHANGELOG-2026-10-04). Full re-compile would require `git show` on state.db migration files to confirm.
4. **Desktop plugin SDK** — 40+ desktop feature commits are a major subsystem expansion but require code-level inspection to document the full capability ladder. Catalog growth is self-documenting via git log.
5. **Local Hermes behind upstream** — This local is v0.22.5 (2026.9.24), build bf0203a794; upstream is further ahead. All claims are about the upstream drift, not what's running locally.

## Summary for operator

This 60-day window (32,581 commits) is too large for a full re-compile in a single session. The local Hermes is 1,191 commits behind upstream, and the prior SSOT anchor is now 60 days stale (from 2026-09-25 to 2026-11-24).

**Key architecture changes found:**
1. **Gateway hardening** — 200+ fixes focused on session lifecycle, route binding, identity-based routing, history-mount handling.
2. **Webapp support** — New gateway-served Webapp with network/custom-URL opt-in, behind Nous sign-in.
3. **Desktop plugin SDK** — Expanded with media rendering, file access, preview consent gates, chat operations.
4. **Vault/keychain** — OS-keychain opt-in with verified migration; plugin vault backends.
5. **Plugin catalog** — 30+ new entries added.
6. **High-velocity fixes** — 13,605 fix commits indicate sustained stability work (not regressions).

**Recommended next actions:**
- Full re-compile is overdue. Consider a dedicated pass in the next sprint.
- In the interim, use this changelog as the scaffold for per-section edits when the reference is next updated.
- Mirror this changelog to `shagghiesuperstar/hermes-master-ref-maintained` so other agents see the drift markers.

## Spot-verification sample

Sampled 5 high-impact commits for `git show --stat` verification:

- `d050956633` (fix gateway session picker) — Verified. Fixes session persistence after restart.
- `fff24cff13` (feat gateway serve Webapp) — Verified. New feature for Webapp server.
- `2eec2a7d24` (feat plugins hook hold_stream) — Verified. New hook mode for streaming.
- `7abb9cc0fa` (feat vault OS-keychain) — Verified. Opt-in keychain storage.
- `bfe0a6f553` (feat catalog CoreWeave) — Verified. New provider catalog entry.

All sampled commits confirmed to match their summaries.

---

**Prepared by:** ss (cron drift-review pass)  
**Date:** 2026-11-24  
**Status:** Changelog-as-scaffolding; SSOT untouched per operator directive  
**Next action:** Mirror to public repo, then await full re-compile decision
