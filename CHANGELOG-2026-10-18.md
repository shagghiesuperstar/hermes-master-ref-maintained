# CHANGELOG-2026-10-18 — Weekly drift review

> **Date correction (2026-11-06).** This file was originally mis-dated: it was named `CHANGELOG-2026-11-10.md`, a date in the future relative to 2026-11-06. The window it describes was reviewed on 2026-10-18 (upstream anchor `22063fc099`, dated 2026-10-18). The file was renamed and the dates inside corrected during the 2026-11-06 sync. It is a historical snapshot, not current truth; see the corrections in [CHANGELOG-2026-11-01.md](CHANGELOG-2026-11-01.md) and the current state in [Hermes_Architecture.md](Hermes_Architecture.md).

## Anchors

- **Local Hermes commit:** `d923a5eb487a` (example profile, last local build)
- **Upstream HEAD:** `22063fc099` (NousResearch/hermes-agent main)
- **Local-behind-upstream:** 1,692 commits — local install is pinned to a prior release branch; normal state
- **Prior SSOT anchor in skill:** `4f22543509` (2026-10-14, ~27 weeks stale [figure from the mis-dated file; superseded by the 2026-11-01 corrections])
- **Delta since prior anchor → upstream HEAD:** **29,791 commits** — largest drift window encountered

## Summary

This is a **massive multi-month drift window** that exceeds what a single cron pass can re-compile safely. Per the skill's decision tree and operator directive ("lightweight local model only"), this is a **changelog-as-scaffolding** pass.

**Threshold triggered:** Prior SSOT anchor > 4 weeks old (27 weeks) + Delta > 500 commits (29,791). Default to changelog-only; do NOT touch `Hermes_Architecture.md` until a full re-compile.

The reference remains untouched; this changelog serves as scaffolding for the next full re-compile, which should run as a dedicated mission when time permits (recommended: multi-hour window).

## Dominant themes by commit type (first 200 commits stratified)

- **fix:** 79 commits — high-velocity bug remediation across Desktop, gateway, plugins, state management
- **chore:** 25 commits — dependency pins, email maps, tooling
- **feat:** 23 commits — new functionality (see below)
- **test:** 22 commits — test coverage expansion (many currently failing — feature-in-flight pattern)
- **plugin-catalog:** 19 commits — new plugin catalog entries and ecosystem growth

## High-impact features identified (feat commits)

### Plugin system expansion (major theme)

- `a2eab1f45c` — **`ctx.post_message` hook** — plugins can post and edit messages in-place; foundational for plugin-driven conversations
- `35948f22ad` — **`post_compaction` hook** — plugins fire after every compaction; new extension point for memory/state maintenance
- `aab0dc72cf` — **`on_session_delete` hook** — plugins react to session deletion across Desktop + deliberate archives
- `0975937593` — **plugin slash commands pass calling session** — slashes get caller context on every surface
- `06e8c5bc70` — **gateway source exposed to slash handlers** — plugins know platform origin (Telegram, Discord, etc.)
- `93fafc4d6f` — **public names for skills discovery/lookup** — skills tool helpers now stable public API
- `14508f97f7` — **platform-plugin public surface documented and allowlisted** — plugins can safely extend platform adapters

### Gateway & connectivity

- `914737a811` — **public SSRF-safe HTTP client** for platform plugins — infrastructure for secure outbound calls
- `47d9efe490` — **group-chat allowlist + pairing suppression** in gateway platform plugins — granular control flow
- `98efa9dfad` — **canonical queued prompt steer/edit from Desktop** — Desktop can edit pending turns before execution
- `d9c9fcf61d`, `beda224abd`, `b0bee4626b`, `fd1b31cf05` — **route-bound model keys** refactoring (4 commits, likely a multi-round fix)

### Desktop UI & onboarding

- `cba95563d4` — **`/loop` dock row with countdown** — visual timer for recurring wakeups
- `e3dd8a80bc` — **Elite terminal/file tree opening** — local model UI affordance
- `809634b8a2` — **first-run intro localizes to app language** — i18n in onboarding
- `af4a05154e` — **Skip setup ends intro** — faster onboarding exit
- `33fa45ef78` — **local-model offer prioritized before sign-in** on machines with local runtime — UX for managed llama.cpp

### CLI & model selection

- `cc55270ec0` — **NVIDIA NIM nemotron added to defaults** — provider expansion
- `e6835fd3cb` — **curated model list at top of picker** — model discovery UX improvement

### Platform expansions

- `47d9efe490` — **connector downloads as trusted media delivery root** — infrastructure for file transfer
- `a278a408c3` — **Render skills + MCP catalog entry** — ecosystem growth
- `46f95e0863` — **Kumiho Memory moved to memory catalog** — memory plugin reorganization

## Plugin catalog entries (19 commits)

- `scalekit` — Scalekit (OAuth/SAML/SCIM infra)
- `offpeak-bekci` — Ollama Cloud off-peak work queue manager
- `Tsubasa` — Tsubasa model-provider plugin
- `edit-ledger` — edit-ledger audit plugin (2 commits: plugin + catalog card)
- `interim-ledger` — interim-ledger plugin (2 commits: plugin + catalog card)
- `gadget` — Hermes Gadget hardware platform integration
- `hermes-whatsapp-voice` — WhatsApp voice mode plugin
- `hermes-lang-it-full` — Italian Desktop language pack
- **Plus ~10 more in the remaining commits**

## Bug fixes (sample from 79 fix commits)

- Gateway: connection card retry, refusal handling, legacy handler policy compatibility
- Desktop: Capabilities draft field preservation during debounce, room loading
- Tools: reaction acquisition and refusal boundaries, durable session id routing
- TUI: subagent kill/steer routing to shared gateway
- Shell hooks: re-sync until applied, standalone scope, consent registration
- Docker: root-owned files returned to user at teardown

## Reverts (concentrated pattern emerging)

- Multiple rollbacks of partial Desktop UI features (desktop: capabilities, model settings alignment)
- Plugin API review + revert cycles (small scope, likely API-stabilization work)

No major reverts equivalent to the schema v30 → v31 boundary that appeared in earlier drifts.

## Schema version status

**No schema bump detected in first 200 commits.** The last known baseline was v31 (2026-10-04). Likely v31 unchanged since then, but cannot confirm without full scan of remaining 29,591 commits.

**Action for next full re-compile:** Verify current `SCHEMA_VERSION` in `hermes_constants.py` or state.db schema, then cross-check migrations since 2026-10-04.

## Known limitations of this pass

1. **Only first 200 commits spot-checked** — delta is too large (29,791) to scan exhaustively in a single cron run with lightweight model.
2. **Docs not fetched** — https://hermes-agent.nousresearch.com/docs/ homepage is current (verified 2026-10-18 [date corrected 2026-11-06]), but individual feature docs (Skills, MCP, providers, CLI) were not re-scanned.
3. **Local Hermes not updated** — analysis is based on upstream HEAD only; actual local install (d923a5eb) may behave differently.
4. **No verification of breaking changes** — only first-pass categorization; actual operator-facing impact unknown.

## Action items for next full reference re-compile

When re-compiling `Hermes_Architecture.md`:

1. **Plugin system section** — Expand to document new hooks (`post_compaction`, `on_session_delete`, `ctx.post_message`), platform-plugin public API, allowlist model, and SSRF-safe HTTP client infrastructure.
2. **Desktop onboarding** — Refresh to include language localization, local-model offer prioritization, Skip setup exit, Capabilities UI patterns.
3. **Gateway internals** — Refactor route-bound model keys subsection; add group-chat allowlist + pairing suppression model.
4. **CLI model picker** — Document curated model list UI, local-model offer, Fast/Ultra reasoning pickers.
5. **Connector/media delivery** — Add section on trusted download roots, connector file transfer.
6. **Schema v31 confirm or bump** — Verify whether v31 is still current or a v32/v33 appeared.
7. **Reverts pattern** — Check if Desktop UI feature instability is resolved or ongoing.
8. **Provider ecosystem** — Update model-provider plugin section with recent maturations (Tsubasa, etc.), NVIDIA NIM nemotron defaults.
9. **Catalog inventory** — Incorporate 19+ new plugin catalog entries into the reference's plugin system overview.
10. **Voice/STT/TTS/i18n** — Verify whether WhatsApp voice, Italian language pack, and other media updates warrant updates to the Voice section.

## Reference state

- `Hermes_Architecture.md`: **UNTOUCHED** — remains at last full re-compile baseline (2026-10-14, 4f22543509 anchor)
- Targeted quirks references: **KEPT as-is** — changelog-as-scaffolding does not modify prior changelogs
- This changelog: **NEW** — added to targeted quirks section for next session's awareness

## Recommendation

**Full re-compile is warranted at next available opportunity** — 29K-commit delta is beyond incremental update scope. Estimated effort: 2-4 hours with a capable model (Opus, GPT-6+). Priority: Medium-high, since the reference is now 27 weeks stale and operator-facing queries will increasingly reference features not yet documented in the local SSOT.

Lightweight cascading updates (another changelog pass) should only continue if the delta is &lt;5K commits at the next review cycle.
