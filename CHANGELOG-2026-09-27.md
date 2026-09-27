# Hermes Architecture Drift Changelog — 2026-09-27 (weekly)

> **Read this before assuming any section of `references/Hermes_Architecture.md` is current.**
> The main reference is unchanged from the 2026-07-05 full re-compile. This changelog
> is the scaffold the next full re-compile will consume; the reference stays untouched
> until then per skill protocol.

## Anchors

- **Local Hermes install:** `59004a62356f3a4697ab0fe8ad5086d2b405e2a6` — v0.21.5+2168.g59004a6.dirty (2026.9.24)
- **Upstream HEAD:** `6f7a7991bb069db07ae74a479823ce8310f8c7e0`
- **Local behind upstream:** 1,530 commits (normal — local pinned to working branch)
- **Prior SSOT anchor:** `4f22543509` (used by `CHANGELOG-2026-09-20.md`)
- **Delta since prior SSOT anchor:** 18,857 commits (weekly window: ~7 days)
- **Schema version:** still v30 (no SCHEMA_VERSION bump visible in this window — verified by grep on `SCHEMA_VERSION` and `schema_migration` markers)
- **Hermes_Architecture.md SSOT:** unchanged; treated as +18,857 commits stale pending full re-compile

## Window focus (since 2026-09-21)

The previous week's changelog (2026-09-20) covered a 4,175-commit window ending at upstream `0497b94570`. This window starts there and lands at upstream `6f7a7991`. The dominant themes for the **last 7 days** specifically are:

| Theme | Count | Why it matters for the architecture reference |
|---|---|---|
| Desktop SDK plugin expansion | ~43 `feat(desktop)` | New plugin-host primitives — capabilities bridge, composer, sidebar, settings, session-list, model-pill, swatches — change the Desktop extension surface from "mostly read-only" to a typed plugin SDK |
| Release pipeline rework | ~30 `feat(release)` | Attempt refs, abandon markers, receipts, canary derivation, `--skip-bundles`/`--skip-tests` — the release process has its own artifact vocabulary now |
| Process Manager (PM) rework | 6 `feat(pm)` | In-process generation adoption, `restart_needed()`, `hermes pm lock`, isolated dev venvs, `hermes pm install --extra` — the PM subsystem went from installer to live dependency manager |
| Plugin catalog growth | ~28 `feat(plugin-catalog)` | New community plugins (`eatwise`, `good-student`, `reading-buddy`, `wechat-social-assistant`, `bot-forge` 0.5.0, `DeskRPG` 0.16.0, `boardstate`, `plan-mode`, `hermes-monitor`, `rss-reader` 1.0.9, `mybot-farm` 0.2.0, `needs-you`, `glasser`, `jev-skill-router`, `jev-judge`, `lib-docs`, `command-tray`, `hermes-impossibl`, `Titanos`, `nvidia-app`, `nvidia-broadcast`, `Telnyx`, `secure-env-ingress`, `pricewin`, `coder` 0.1.0, `prompt-enhance`, `Hermes-zh`, `DeskRPG` gateway plugin) — catalog scaling, not core change |
| Agent streaming/reasoning fixes | many `fix(agent)` | Inline reasoning recovery, repetition abort, retry-exhausted 429 handling, prefix-cache semantics — runtime behavior tightening |
| MCP OAuth improvements | `fix(mcp)`, `feat(plugins)` | Disk-watch baseline, OAuth reload on first sight, public plugin event bridge (`broadcast_plugin_event`) |
| Webhooks cross-platform | 1 high-impact `feat(webhook)` | Webhook deliveries mirror text into the target chat's session (cron-style) — closes a long-standing delivery loop gap |

## Architecture-area deltas (since prior SSOT anchor `4f22543509`)

The next full re-compile of `Hermes_Architecture.md` should consume this list as a 1:1 merge map. Each row is a delta bucket; spot-verified commits are cited.

### Configuration & provider routing
- No new provider plugin milestones observed in this window. Model-provider plugin system (`749443e292`, `97fd8c8f2a`, `bce20d0b1f`, `ef0bac6cdd`, `400b6c41cd`, `dce233e809`, `de66512ab7` per CHANGELOG-2026-09-20) remains the most recent major routing change.
- **feat(web): serve managed search through Perplexity** — `749220ef00` — managed route uses Perplexity for search, keeps Firecrawl for extract + per-call fallback. Status/portal/docs say "managed" not the vendor.
- **chore(terminal)** + refactors around heartbeat schema description and `terminal_env_registry` — minor surface cleanup.
- **fix(config)**: stop the migration ladder rewriting unversioned configs — `a88bef98b2`.

### Schema versions & state.db migrations
- No SCHEMA_VERSION bump observed in the last 7 days. Still **v30** per prior changelog.
- `hermes_state_schema`, `hermes_state_portability`, `hermes_state_titles`, `hermes_state_telegram`, `hermes_state_registry`, `hermes_state_search`, `hermes_state_usage` — incremental internal refactors (table layout unchanged at the public contract level).
- `hermes_state_titles` cleanup; `hermes_cli/update` and `agent/session_persistence` co-renames suggest an ongoing mixin split.

### Approvals & security
- **fix(security): don't warn while tirith's first download runs** — small UX fix.
- No approvals-mode default flips this week (smart still default per CHANGELOG-2026-07-19).

### Skills (format, disabled-gate, preload)
- **`test: preserve skill schema cache stability across profiles`** — `7add9e62a4` — schema cache is now stable across profiles (important for multi-agent profile hygiene on M5 fleet).
- **`fix: stabilize Python e2e entrypoints and skill schema`** — `af23d04787`.
- New community skills via plugin catalog: `plan-mode`, `jev-skill-router`, `jev-judge`, `lib-docs`, `command-tray`, `glasser`, `hermes-monitor`, `needs-you`, `prompt-enhance`.

### MCP (auth, parked-revival, preflight)
- **`fix(mcp): reload OAuth provider on first sight when no tokens in memory (#39551)`** — closes the long-standing "first OAuth login fails because tokens aren't loaded yet" loop.
- **`fix(mcp): seed OAuth disk-watch baseline before reload`** — proper ordering.
- **`fix(mcp): keep required-only constraint fragments in tool schemas`** — schema fidelity fix.
- MCP exact-version pins still apply (per CHANGELOG-2026-07-26).

### Sessions & state
- **`fix(sessions): refuse to delete a session row a live turn still owns (#123583)`** — `40523600b0` — write-guard, plus test coverage for delegation-cascade and idle compression-ended rows. Source of truth: the `delete` command now refuses mid-turn deletes and reports `skipped_active` in bulk responses.
- **`fix(sessions): share the write-guard filter and trim repeated walks`** — `230f89b47b`.
- **`fix(persistence):** a cluster of related commits (`4f6ab19304`, `4854225903`, `6e8aa00626`, `ca7e4c1636`, `1a43a4ef48`, `02b8c4a655`, `e4e7837623`, `7581677da8`, `1a95a75b1a`, `f337631f43`) — transcript rows versioned by digest, metadata-only adoption, live multimodal content preservation, caller row state restored on rollback. **This is a substantive persistence layer rewrite** — the next re-compile should treat transcript-row identity and digest-based versioning as new core behavior.

### Compression / context (TTFT, autoraise, native paths)
- Multiple `refactor(agent/conversation_compression)`, `refactor(agent/compression)`, `refactor(agent/context_engine)`, `refactor(agent/context_references)`, `refactor(agent/context_compressor)`, `refactor(agent/context_breakdown)` entries — incremental module split.
- **`fix(agent): recover inline reasoning stripped by the think scrubber`** — `725…` cluster — inline `<think>` text fed to the live reasoning pane; recovered if the scrubber removed it (#89647).
- **`fix(agent): scope recovered inline reasoning per model response`**, **`fix(agent): feed inline <think> text to the live reasoning pane (#89647)`**.
- **`fix(agent): only 4xx/5xx body codes count as a stream error's HTTP status (#121270)`**, **`fix(agent): classify status-less streaming render errors as format_error`**, **`fix(agent): an in-stream SSE error object's numeric code is its HTTP status`** — three commits tightening stream-error classification.
- **`fix(agent): collapse orphan continuation trail on retry exhaustion (#119001)`**, **`fix(agent): retain delivered partial output on retry-exhausted 429/network errors`** — partial-output retention semantics clarified.

### CLI (new commands, removed commands, behavior changes)
- **`fix(cli): keep hermes_cli import off os.environ; run the import-purity test on Windows`** — small but affects Windows install path.
- **`fix(cli): limit the turn-level streamed guard to interrupted results (#65666)`**, **`fix(cli): don't re-render an interrupted reply streamed before a tool-call transition (#65666)`**, **`fix(cli): drop dead _deferred_content plumbing; cover reasoning-then-answer streaming (#47116)`**, **`fix(cli): close reasoning box on first content token to enable answer streaming`** — stream-rendering cleanup cluster.
- `hermes_cli/setup`, `hermes_cli/commands`, `hermes_cli/main`, `hermes_cli/web`, `hermes_cli/voice`, `hermes_cli/web_routers`, `hermes_cli/web_git`, `hermes_cli/AGENTS.md`, `hermes_cli/update` — internal module churn.

### Cron (run-claim, scheduling, past-timestamp rejection)
- **`fix(cron): bound Codex max-iteration summaries`** — bounds runaway Codex cron summaries.
- **`refactor(cron): move job-definition merge schema into cron/job_definition.py`** — `c2063cf61d` — module split.
- **`docs(cron): list the retry state's expr fingerprint`**, **`docs(cron): state the bounded, sparse-only quota recovery retry`** — retry-state contract documentation.
- No new cron commands observed.

### Gateway (routing, pairing, session-key guards, multiplex)
- **`feat(webhook): mirror cross-platform deliveries into the target chat session`** — `11fb429f49` — webhook deliveries are mirrored into the target chat's transcript as a labelled user turn, same convention as cron briefs. Route opt-out via `mirror_to_session: false`. Closes a delivery loop where the agent had no memory of having just sent a webhook message.
- **`fix(gateway): log failed transformed-final edits; make fail_result required`**, **`fix(gateway): don't ask the Windows login question when stdout is captured`**, **`fix(gateway): prove gateway ownership, not just liveness; stop trapping a mid-migration host`**.
- Slack draft-stream regression cluster: `72893ca6a1`, `d5bddd7f00`, `d5a2d1f069`, `d6d3ed005c`, `02426efbae`, `fdec926ef5` — slack native draft stream reopen, in-place replacement of rewritten native-stream final, per-thread stream key. Tied to PR #119041.
- Multiplex still default (per CHANGELOG-2026-09-13).

### Platform adapters — per-platform
- **Discord**: incremental `fix(discord)` (32 commits) — typical adapter churn.
- **Telegram**: `fix(telegram)` (46) — incremental.
- **Slack**: `fix(slack)` (20) — see Gateway section above for the draft-stream cluster.
- **WhatsApp / WhatsApp-Cloud**: `fix(whatsapp)`, `fix(whatsapp_cloud)` — incremental.
- **WeCom**: `fix(wecom)`, `docs(wecom): native streaming follows the global streaming.enabled switch (#53697)`.
- **Feishu**: `fix(feishu)` (22).
- **Matrix**: `fix(matrix)` (17).
- **BlueBubbles (iMessage)**: `fix(bluebubbles)`.
- **Signal**: `refactor(signal)` — incremental.
- **SimpleX**: `fix(simplex)` (9) — name allowlist hardening, profile-scoped reads.
- **Email**: **9 commits** — `fix(email): parse the From mailbox with the stdlib, not a first angle-bracket match` (and 8 follow-ups on linear mailbox-safe From parsing, DMARC/SPF/DKIM verdict scoping, malformed-but-real From value retention). **Substantive email subsystem rewrite.** Source-verified: `d0c588972e`, `54b0f5db25`, `2d44f5512b`, `b821f8637b`, `517a97cd31`, `3f4533b9de`, `6f7a7991bb`. Next re-compile should treat email parsing as stdlib-based with explicit clause-level DMARC/SPF/DKIM verdict handling.
- **Ntfy / SMS / Mattermost / Google Chat / IRC / Raft / Homeassistant / Yuanbao / Dingtalk / QQBot / Tencent / Google Meet / Line / Weixin**: incremental adapter churn.
- **Discord / Telegram / Slack pairing**: no new defaults; existing pairing model unchanged.

### Computer-use (cua-driver, env sanitization)
- `refactor(computer_use)` (78 commits) — incremental; no semantic changes observed this week.

### Providers / Auth (OAuth, Z.AI, Codex, Nous, Anthropic, OpenRouter, etc.)
- **`fix(auth)`: 93 commits** — credential-pool and OAuth-session churn.
- **`fix(provider)`, `fix(providers)`** — `fix(providers): report why a lazy SDK install did not land` is the only publicly visible provider-shape change.
- **`fix(xai)`, `fix(minimax)`, `fix(deepseek)`, `fix(kimi)`, `fix(meta-ai)`, `fix(gemini)`, `fix(openai)`, `fix(openrouter)`, `fix(anthropic)`, `fix(bedrock)`**, **`fix(nous)`**, **`fix(copilot)`**, **`fix(copilot-acp)`**, **`fix(codex)`** (56 commits) — provider-specific fix-ups; no catalog additions this week.
- **`fix(moonshot)`, `fix(opencode)`, `fix(opencode-free)`, `fix(opencode-go)`** — OpenCode provider churn.
- **`fix(vercel)`** — Vercel AI Gateway.
- **Test wiring**: **`test: copilot-acp wire E2E`** (20c41d1e69), **`test(e2e): Gemini native generateContent wire-conformance suite`** (0929292aca) — provider wire-conformance E2E tests added for Copilot ACP and Gemini native. **These are the first conformance suites of their kind** — next re-compile should note this as a new layer of provider quality enforcement.
- No new OAuth providers added this week.

### Web / search providers (Firecrawl, exa, parallel, tavily, brave-free)
- **`feat(web): serve managed search through Perplexity`** — `749220ef00`. See Configuration section.
- `fix(web)`: incremental — TTL caches, skipped-active toast i18n, etc.
- `fix(tavily)`, `fix(brave-free)` — provider-key cleanups.

### Docker / Photon
- **`feat(docker): install opt-in dependencies into PM generations on the volume`** — `72df5aa60e`.
- **`docs(docker): document the full Chromium in every image`**.
- **`feat(bot_desktop): ship Bot Screen on hosted images (-desktop tags)`** — `686c34d3f6` — Docker image axis `[slim, desktop]`. `:latest-desktop`, `:main-desktop`, `:v*-desktop` carry Xvnc/Xfce/headed Playwright Chromium. **This is a meaningful hosted-Bot-Screen deliverability change** — hosted instances no longer need to install at run time.
- Photon: `refactor(photon)`, `fix(photon)` — incremental.

### Desktop / TUI
- **MAJOR SURFACE CHANGE.** 43 `feat(desktop)` commits. Highlights:
  - Sandboxed embed primitive for plugins
  - `sdk/bridge.ts` capabilities bridge (pluginDecisions read-only)
  - Typed skills/toolsets/profiles bridge for plugins
  - Model-pill label providers for plugins
  - Sidebar nav visibility and prefs (`sidebarNav.prefs`)
  - Session-list API + row-decoration slots for plugins
  - Composer draft API (`host.composer.*`) + `host.composer.focus(sessionId)`
  - Plugin-declared settings rendered in Plugins tab
  - Choose plugin install profile
  - After plugin install: "Connect now" for MCP servers (no gateway restart needed) (#119349)
  - Voice function-key shortcuts, composer dictation keybind
  - ⌘K groups pinned sessions first
  - French / German / Spanish i18n catalogs (full)
  - Appearance text direction (Auto / RTL / LTR)
  - "Always open links in external browser" setting
  - Custom model entry in Settings model selects
  - Interface mode (Simple hides instrumentation)
  - Connectors page replaces MCP tab (#119074)
  - Files-pane HTML render | source toggle
  - Tray-preference + minimize-to-tray
  - Per-chat composer status stack hide
- TUI: `refactor(tui_gateway)` (76), `fix(tui)` (93), `refactor(tui)` (72) — fanout-overflow signal, socket-close only on overflow (`59004a6235` is the local HEAD; matches the latest upstream fix).

### Secrets (SecretSource plugin system, Bitwarden, 1Password)
- No new SecretSource plugins this week.
- `fix(secret_scope)`, `refactor(secrets)` — incremental.
- `fix(security-guidance)` — security guidance incremental.

### Plugins (catalog, SDK, events)
- **SDK growth**: capabilities bridge, composer, sidebar, session-list, model-pill, swatches, settings — see Desktop section.
- **`feat(plugins): public event bridge for plugin backends`** — `8034d493dc` — `hermes_cli.plugin_events.broadcast_plugin_event(plugin_id, event, payload)`. Plugin authors no longer import private `tui_gateway.server._broadcast_global_event`. Closes wishlist item 8 of #116305; unblocks `rss-reader` (#115972).
- **`feat(plugins): plugin event names take a dotted hierarchy; ids are catalog names`** — `5fa1402819` — `plugin.<plugin_id>.<event>` namespace, validated.
- **Catalog growth**: 28+ plugin catalog entries this window (see Window focus table).
- **`docs(plugins): broadcast_plugin_event contract, delivery scope and rss-reader migration`**.
- **`docs(desktop-sdk): model-pill label providers — signatures, arbitration, migration`**, **`docs(desktop-sdk): host.composer signatures, arbitration, consumer migrations; document COMPOSER_AREAS.underside`**, **`docs(desktop): host.settings — teardown rule, declined keys, consumer migration`** — SDK consumer-migration docs.

### Models / catalog
- No new model additions this week (no `feat(models)` high-impact commits).
- `feat(model-pickers)`, `feat(model-catalog)` — incremental picker UX.

### Memory (Hindsight / Omega / Graphiti / Honcho / Mem0 / supermemory)
- `fix(memory)`: 35 commits — incremental.
- `fix(hindsight)`: 5 commits.
- `fix(memory/mem0)`, `fix(memory/byterover)`, `fix(memory/supermemory)`, `fix(memory/holographic)` — provider-specific.
- `feat(honcho)`: 8 commits.
- `refactor(honcho)`: 19 commits.
- `fix(honcho)`: 60 commits.
- `feat(memory)`, `feat(docs+memory)`, `feat(super)memory` — no externally-visible contract changes.

### Reverts
- **`Revert "fix(tts): install kittentts on python 3.14 via the misaki fork"`** — 1 revert this week. **`docs(i18n): note KittenTTS is unavailable on managed Python 3.14`** — confirms the revert.
- No other reverts observed.

### Salvaged PRs
- **`chore: map shali10 for salvage of #123725`** — `a7f146fd15` (sessions/email area).
- **`chore: map Nagisa-3000 for salvage of #123849`** — `f1826946ca` (gateway area).
- **`chore: map shali10 for salvage of #123725`** (also seen — second mapping).

### Salvage-test trims (cosmetic but notable)
- Several `test(<area>): trim ... to two invariants` commits indicate ongoing test-matrix compaction across sessions, email, files, persistence. Cite-on-pattern for the next re-compile.

## Action items for next full reference re-compile

1. **Persistence / transcript rows** — adopt the digest-versioning + metadata-only adoption model. Cite `4f6ab19304`, `4854225903`, `6e8aa00626`, `ca7e4c1636`, `1a43a4ef48`, `02b8c4a655`, `e4e7837623`, `7581677da8`, `1a95a75b1a`, `f337631f43`.
2. **Session delete write-guard** — `40523600b0` + cluster. Document `delete` mid-turn refusal, `skipped_active` in bulk responses.
3. **Email parsing** — stdlib From parser, clause-level DMARC/SPF/DKIM. Cite the 7-commit email cluster (`d0c588972e` … `6f7a7991bb`).
4. **Webhook delivery mirror** — `11fb429f49`. `mirror_to_session: false` opt-out.
5. **MCP OAuth ordering** — `fix(mcp): reload OAuth provider on first sight when no tokens in memory` and seed-disk-watch-before-reload.
6. **Desktop plugin SDK surface** — 43 `feat(desktop)` + 5 `feat(plugins)`. Add a new "Desktop Plugin SDK" section covering capabilities bridge, composer, sidebar, session-list, model-pill, settings, swatches, sandboxed embed.
7. **Release pipeline** — attempt refs, abandon markers, receipts, canary derivation. The release section needs its own artifact-vocabulary subsection.
8. **PM subsystem** — `restart_needed()`, `adopt_selected()`, in-process generation adoption, `hermes pm lock`, isolated dev venvs, `--extra` install.
9. **Managed search vendor** — Perplexity route, "managed" not the vendor in copy. `749220ef00`.
10. **Provider wire-conformance E2E** — Copilot ACP (`20c41d1e69`), Gemini native (`0929292aca`). New quality-enforcement layer.
11. **Inline reasoning recovery** — think-scrubber recovery + pane integration. Cite the `fix(agent)` cluster (#89647, #121270).
12. **Stream-error classification** — only 4xx/5xx body codes, SSE error object numeric code as HTTP status, status-less render errors classified as `format_error`. Cite `fix(agent)` cluster (#121270).
13. **Partial-output retention** — orphan continuation trail collapse + retry-exhausted 429/network errors. Cite #119001.
14. **Docker image axis** — `[slim, desktop]`. `:latest-desktop` etc. carry Xvnc/Xfce/Playwright Chromium.
15. **Plugin event bridge public API** — `hermes_cli.plugin_events.broadcast_plugin_event`, dotted hierarchy `plugin.<plugin_id>.<event>`.

## Known limitations of this pass

- Window is large (18,857 commits since the prior SSOT anchor). Many deltas are not deeply spot-verified — only the high-impact commits (above) were confirmed via `git show --stat`.
- The previous-changelog window (2026-09-20, 4,175 commits) had its own `git fetch origin main` issues at the time it was authored; some of that delta may not be deeply covered anywhere in the chain.
- Local install is 1,530 commits behind upstream — the upstream HEAD has features the local install does not yet have. Run `hermes update` if a feature is needed locally.
- Cron/memory/MCP/auth sections were not re-verified against the live running daemon; rely on empirical probes (`hermes cron list`, `hermes mcp list`, etc.) for those.

## Verification commands (canonical, this drift review)

```bash
# Anchor commands
hermes --version
git -C ~/.hermes/hermes-agent rev-parse HEAD
git -C ~/.hermes/hermes-agent rev-parse origin/main
git -C ~/.hermes/hermes-agent rev-list --count HEAD..origin/main
git -C ~/.hermes/hermes-agent rev-list --count 4f22543509..origin/main

# Area spot-checks
git -C ~/.hermes/hermes-agent show --stat 749220ef00   # managed search
git -C ~/.hermes/hermes-agent show --stat 11fb429f49   # webhook mirror
git -C ~/.hermes/hermes-agent show --stat 40523600b0   # session delete write-guard
git -C ~/.hermes/hermes-agent show --stat 8034d493dc   # plugin event bridge
git -C ~/.hermes/hermes-agent show --stat 686c34d3f6   # bot_desktop docker image axis

# Diff buckets since prior anchor
git -C ~/.hermes/hermes-agent log --since=2026-09-21 --pretty=format:'%h %s' | grep -E '^feat\(desktop\)' | wc -l
git -C ~/.hermes/hermes-agent log --since=2026-09-21 --pretty=format:'%h %s' | grep -E '^feat\(release\)' | wc -l
git -C ~/.hermes/hermes-agent log --since=2026-09-21 --pretty=format:'%h %s' | grep -E '^feat\(pm\)' | wc -l
git -C ~/.hermes/hermes-agent log --since=2026-09-21 --pretty=format:'%s' | grep -E '^revert' | wc -l
```