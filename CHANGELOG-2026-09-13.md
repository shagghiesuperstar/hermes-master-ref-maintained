# Drift Review — 2026-09-13 (changelog-as-scaffolding)

## Anchors
- Local Hermes: v0.21.2 (2026.9.11), commit `31d0a2428e`
- Upstream HEAD (origin/main): `5dea46d13deec9549bdc2ea703ae9201d733c28d`
- Delta: **649 commits** (`31d0a242..origin/main`); 631 conventional commits, ~60 feat
- Prior SSOT anchor: `4f22543509` (CHANGELOG-2026-09-06, v0.21.0, +5,800-commit window)
- Local install is 649 commits behind upstream — `hermes update` pending (operator call, not this pass)

## Window summary
v0.21.0 → v0.21.2. Dominated by gateway multiplexing maturation, Honcho peer-model rework, desktop/TUI/shared-TS consolidation, and cron delivery reliability. No SCHEMA_VERSION bump observed in the feat stream. One breaking change (multiplex allowlist removal).

## Deltas by architecture area (source SHAs attached)

### Gateway / multiplex (highest impact)
- `9848e22ed6` **BREAKING**: `gateway.multiplex_profile_allowlist` dropped — multiplexed default gateway now serves default + every live named profile under `profiles/` (directory read, tombstones skipped, no mkdir). All readers drop the allowlist param.
- `e2fc493427` + `07a4ae016a` + `df95f378a6`: `hermes gateway migrate --multiplex / --standalone`, auto-migration when unblocked, dashboard System-page migration card.
- `bcdb49ac7b`: adapters declare `serves_profile_prefix` for `/p/<profile>/` ingress (api_server + webhook).
- `c0d8d6b5d5` + `6fc76ce751`: secondary profiles' inbound-port platforms served on the default's shared listener via `shared_ingress.bind_listener`.
- `d1dbb0ac9e`: multiplexer hot-serves profiles created while running; unroutes deleted ones.
- `65fd3a2b9c`: status shows each served profile's shared-listener callback URLs.
- `f561155a70` / `5dea46d13d`: gateway token-refresh coalescing per provider; dashboard auth refresh single-flight off the event loop.

### Approvals / security
- `24692ee790`: cloud metadata-endpoint (IMDS: 169.254.169.254, metadata.google.internal, Alibaba) fetches flagged for approval — closes silent credential-exfil path.

### Auth / providers
- `0007a4c2f9`: OpenRouter OAuth PKCE login — `hermes auth add openrouter --type oauth`; minted key stored as a normal api_key pool entry (source `manual:openrouter_pkce`).

### Cron
- `ec58e08a35`: automatic bounded re-runs (5/15/30 min pattern) when a recurring fire never reached the model (transient network/DNS, e.g. wake-behind-VPN).
- `df51797e2e`: planned downtime can skip missed recurring runs.
- `0037a4b17a`: cron `resnap` action adopts the current global inference default.

### Vision / media
- `0a545f7fa5` (+ f5a7dc14c7, c83c413214, adaa6643b9): HEIF/HEIC/AVIF decode (iPhone photos) — magic-byte sniff of ISO-BMFF `ftyp` box; malformed ftyp size fails closed; pillow-heif bounded dep.

### Voice
- `f923faa0b8`: GPT-Live voice chat mode (`voice.voice_chat_mode: gpt-live`) — full-duplex gpt-live-1; heard requests become normal Hermes turns on the open chat.
- `20816c13cd`: voice chat engine selectable from the composer.

### Honcho
- `21180ae7e8` / `0a7c17159c` / `dc7c8673d1`: setup wizard reworked around Honcho's peer model; `sessionAiPeerPrefix` isolates sessions per AI peer; `hermes honcho peers map` interactive mapping.
- `56231e51ad`: honcho read baseline advances after each write so a revert reaches disk.

### CLI / TUI / Desktop / shared TS
- `2a5373da2c` / `750b2fc5e2`: `display.vim_mode` config + `/vim` command; vi editing mode in prompt_toolkit.
- `7b037f0efa`: opt-in git_branch status-bar field.
- `b05a47b9d2`: desktop reasoning-effort composer pill; `e1c05ffa32` persist video playback speed; `995e79afe4` tutorials retire after first month; `4f1966edac` guided first launch + connector sign-in cards.
- Large `@hermes/shared` consolidation wave: one slash parser (`057c2c85fc`), slash block-list derived from Python registry (`458595a20b`), sRGB color-math/fuzzyRank/compactNumber/i18n/stripAnsi shared modules (`35022e02ed`, `3f02259518`, `172b2a722b`, `65ca7eac5f`, `a3d259019b`).
- `c5973cd540`: TUI-gateway agent built with authenticated dashboard user as user_id.

### Plugins / hooks
- `f361971eed` + `d3202bbc8d`: `agent_loop_stopped` plugin hook fires on interrupt too.
- `fafb27ee5c`: `on_room_member_activity` hook projects Group Chat member runtime events.
- `65fd3a2b9c` sibling: touchdesigner ships as catalog plugin (`f364c19775`); snyk MCP + skill added (`53c57871d6`).

### Skills
- New ports: dream-loop (`47c029927f`), system-atlas (`41b2c615d`), mono-color (`b35c0284a1`), pr-lens (`d7a56474fc`), dynamic-workflow re-added (`351041463d`).
- `be2f7e9c36`: curator prunes unused skills at 30 days (was 90), stale at 14.
- `850c48cd84`: skills-hub tap K-Dense + OpenScience under a "science" bucket.

### Video / image
- `63584da036` MiniMax H3 Max Turbo family; `387ac50d85` OpenRouter Hailuo 3 Max; `c6f87deb2c` OpenRouter video backend covers live catalog; `82199439c7` Meta Muse Image ($0.01/img) in FAL catalog.

### Platform adapters
- `a5522f69c0`: Slack status/title routed through Agent Sessions API (slack-sdk 3.44.0).
- `5dcd4844bd` / `8dd0e80f29` / `cef499fbb5`: Discord liveness probe gains dispatch-side dimension; unreachable frame-silence dimension dropped; int knobs reject inf/fractions.

### Session/migrate/tooling
- Migration-tool hardening wave: `422bc9bde9`, `fadcff9227`, `d8cfb149b1`, `f9e47aa6fe`, `043286ae68` (rollback manifest written before destructive ops; standalone rollback recoverable).
- Auxiliary: `aa40c1d21b` Messages adapter wrapped on profile's declared api_mode; `5dd8fa1daa` anthropic_messages profiles keep /reasoning reachable.
- Agent loop: confirmation-expiry / replay-canonicalization / send-path-prefix hardening (`f296652a66`, `820d3ca65d`, `e5ca5207de`, `401fef6e6f`, `5c4c31cf4d`).
- Tests: tree mirrors source layout, issue numbers dropped from filenames (`d10bb2ab6f`, `76e88cae2a`, `42e7471c64`).

## Reverts
None in this window (one honcho fix `56231e51ad` mentions "revert reaches disk" — a fix, not a revert commit).

## Action items for next full reference re-compile
1. Gateway section: document multiplex default-serve-all-profiles, `/p/<profile>/` ingress, `gateway migrate`, shared_ingress listener; remove any mention of `multiplex_profile_allowlist`.
2. Approvals section: add IMDS metadata-endpoint dangerous pattern.
3. Auth section: OpenRouter PKCE OAuth flow.
4. Cron section: bounded re-run on no-model-reach, planned-downtime skip, `resnap`.
5. Vision section: HEIF/HEIC/AVIF sniff + fail-closed ftyp.
6. Voice section: `voice_chat_mode: gpt-live` full-duplex mode.
7. Honcho section: peer-model wizard, `sessionAiPeerPrefix`, `honcho peers map`.
8. CLI: `display.vim_mode`, `/vim`, git_branch status field.
9. Skills: curator 30d/14d cadence; new hub bucket "science".

## Known limitations of this pass
- Lightweight local model + single cron pass: docs site not re-crawled; deltas sourced from commit stream + `git show --stat` spot-checks (5 SHAs verified in depth, rest by message).
- Local install not updated (still 649 behind) — behavior differences cannot be probed live this pass.
- SCHEMA_VERSION not probed; assume still 30 unless next pass verifies.

## SSOT status
references/Hermes_Architecture.md untouched (scaffolding pattern). SSOT now +649 commits / ~1 week past the v0.21.0 anchor.
