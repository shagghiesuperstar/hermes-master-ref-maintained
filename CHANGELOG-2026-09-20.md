# Drift Review — 2026-09-20 (changelog-as-scaffolding)

## Anchors
- Local Hermes: v0.21.3 (2026.9.14), commit `498abb677ec39ea3ae9f8f5ed60e7def6bc47e70`
- Upstream HEAD (origin/main): `0497b945707788bf4297645d6c563463cbf98e0a`
- Delta from prior SSOT anchor: **4,175 commits** (`4f22543509..origin/main`) — 512 features, ~3,500 fixes/refactors, +2 release tags (v0.21.1, v0.21.2, v0.21.3 since prior 2026-08-31 v0.21.0 baseline)
- Prior SSOT anchor: `4f22543509` (CHANGELOG-2026-09-13, v0.21.0–v0.21.2, +649 commits from prior week)
- Local install IS the latest release tag (`498abb677e` is v0.21.3) but still trails upstream by the development done since the tag. `hermes update` is pending (operator call, not this pass).

## Window summary
v0.21.0 → v0.21.3. **Plugin-catalog expansion (62 commits)** is the single largest theme — every model-provider, plugin, and most reusable products now ship through `plugin-catalog:` pins and Discoverable MPP entries. **Model-provider plugin system** reached usable maturity: OAuth (Authorization Code + PKCE), per-plugin error classification, per-plugin capability metadata, account-usage hooks, plugin-owned `hermes auth` surface, declarative `ProviderProfile`. **Managed local llama.cpp runtime** (43e67d872f) shipped as a first-class backend. **GPT-6 Astra** + `claude-fable-5.1` joined the catalog. **Cron** got the biggest reliability story since v0.18: failure-incident ledger (one alert per outage, not per run), completion-on-success / re-open-on-replay, `pinned` job model, doctor health check, degraded-marker redaction. **Desktop catalog UI** was attempted, partially reverted, re-attempted, then mostly reverted again — the Desktop now stays largely card-based without the native catalog browser (multiple `Revert "feat(desktop): browse skills and plugins in a shared native catalog"` commits). **Voicemail / STT/TTS** got a sweep of `XAI_API_KEY` precedence fixes, configurable streaming threshold (`tts.streaming.min_len`), ElevenLabs v3 + gpt-transcribe in pickers.

**SCHEMA_VERSION stays at 30** (since 2026-08-31 v0.21.0). The only schema-touching commit in this window is `perf(state): keep delegate-child transcripts out of the trigram FTS index` (2b55ded1ac) — it patches a behavior with no version bump.

## Deltas by architecture area (source SHAs attached)

### Plugins / plugin-catalog (highest impact — 75 catalog+ catalog-adjacent commits)

The `plugin-catalog:` surface absorbed the majority of architectural risk and reuse this window.

**Catalog backflow & entries (~62 commits):**
- `fafcddbca4` — Apify plugin entry added.
- `a3baf5135d` — `hermes-talk` pin to v0.21.0; new voice category.
- `55f09ed25f` — `hermes-speech` v0.3.2.
- `bac32f2890` — `hermes-sidepulse` beta backend.
- `f9073eff76` / `9c1ef6bfcf` / `23bb584b96` / `633b065abb` / `250d8a89d1` / `05e580ff62` — rss-reader lifecycle: add → pin v1.0.4 → pin v1.0.5 → pin v1.0.5 fix → public edition.
- `a7f63109e6` / `3ee2d10098` / `97f731020b` — provider-status plugin: catalog card → public edition pin.
- `97f731020b` — provider-status plugin entry.
- `4666ce6d83` — Adspirer MPP entry.
- `58bc617c8b` — sticky-notes community plugin.
- `2e58fb3b04` — provider-copy community plugin.
- `1764d01b7d` — `hermes-structured-aux-models` plugin entry.
- `00570550f3` / `538d0ee5bb` / `e4733ccad5` — iteration-budget-meter catalog entry → pin → card.
- `e5472fbc9c` — reasoning-switch catalog card.
- `f2a85c16f7` — Tempo MPP catalog entry.
- `4666ce6d83` — Corpus plugin entry.
- `5db3298a5e` — `hermes-project-stewardship` catalog add.
- `fc32ca7974` — `hermes-loadout` community plugin.
- `bee4fabf7a` — githermes catalog entry.
- `d02a840fb4` — hermes floor declared and placeholder key noted for zerosignal.
- `0b78a26903` — `hermes-security-audit` catalog entry.
- `13d5b65168` — `hermes-opper` (Opper model provider).
- `904be3a93a` — desktop plugin catalog refuses plugins that step outside the SDK surface.
- `9478d543da` — `my-github` desktop & dashboard plugin.
- `b978eaab43` — agent-log desktop plugin.
- `24fd6ca65a` — Session Diff desktop plugin.
- `369d3b49ca` — Toolsmith desktop plugin.
- `d09183d8d3` — hermes-talk catalog entry → v0.19.2 (separate from v0.21.0 pin in a3baf5135d).
- `904be3a93a` — desktop plugins that step outside the SDK surface are refused at install.
- `50551b24fa` — metamask-wallet community entry.
- `45d3182cd0` — filebox re-pinned to v0.3.0 (security fixes from review PR #6).
- `14802a586c` — pstack maintainer Zoeille → Cloeille (catalog metadata correction).
- `76fe8f7f5f` — plugin catalog ranks by stars only; review-claim badges dropped.
- `48257c6e31` — stewardship repin for security fix.

**Catalog-backed Desktop UX (multiply reverted):**
- `4718ba7352` / `8bd0da2b8c` — Revert of the shared native catalog browser (twice).
- `07c9088460` — Feat attempt: native skill+plugin catalog browse (reverted).
- `cbd76e4ea3` — Feat attempt: restore native skill/plugin catalogs (reverted).
- `3eb9712180` / `007e80ee0f` / `9796235822` / `9b8cc18364` / `9d67c7c29a` / `9139bb8a4e` / `6a912eb1cb` — Seven revert commits associated with the shared-catalog feature experiment. Net effect: **the Desktop catalog browser is OFF**. Cards-only catalog rendering remains.

### Model-provider plugin system (substantial maturity step)

This is the most architecturally significant movement. The plugin system reached a "first usable + first extensible" state.

- `bce20d0b1f` — `ProviderProfile.classify_api_error` lets model-provider plugins classify their own API errors (replaces Hermes core's per-provider heuristic table).
- `97fd8c8f2a` — Declarative OAuth Authorization-Code + PKCE login for provider plugins (no Python required to add an OAuth provider).
- `ef0bac6cdd` — Plugin-declared per-model capability metadata (capabilities, modalities, prompt-cache support).
- `749443e292` — Plugins OWN `hermes auth add/status/logout/refresh` for themselves.
- `400b6c41cd` — Plugin account usage hook.
- `dce233e809` — OAuth-shaped provider plugins register, log in, refresh through their profile's auth_handler.
- `de66512ab7` — Opt-in browser Authorization Code + PKCE login for `openai-codex` (sibling to the plugin system).
- `031d70f671` — `missing-auth_handler` guard fires only for plugin-mirrored providers.
- `04f80dba0d` — Re-syncs plugin providers into `PROVIDER_REGISTRY` after discovery.
- `0d2cb2cacc` / `438d2f061a` / `0bbf7b7997` / `3e7a8636e8` / `33b2461be4` / `6d4ca15d38` / `75b6281394` / `e218f3641c` — Codex OAuth hardening: streaming-compaction gate, refresh-401 soft failure, body cap at 1 MiB, residency claim extraction from JWT, slug-rewrite for paths/repos/mailboxes, docs URL preservation in sanitizer.
- `3934fe551c` — Sweep retired `gpt-5.3-codex` slug off every Codex-OAuth surface.
- `99fff22f` (Note: not in delta — verifying) — Codex OAuth/device-auth response body 1 MiB cap.
- `c6d12dafa0` — Cap Codex OAuth/device-auth response bodies at 1 MiB.
- `1710f8413d` — Exhausted OAuth credential pool reports cooldown + reset time, not "no credentials".
- `5c113bb07c` — Accept dict-form `reasoning_effort` for providers with bespoke thinking tiers.
- `8e180c69f6` — Label a `model.context_length` pin; warn once when it disagrees with the provider.
- `c47a0e6f6` (verify) — per-model `reasoning_effort` overrides.
- `ec1238fa65` — `credential-pool`: plugin refresh keeps rotated tokens, runs locked, quarantines dead grants.
- `99da30a35` — OAuth discovery carries User-Agent.
- `668e505ac7` / `7eed615690` — MCP OAuth: User-Agent on SDK-built discovery/registration; cancelled login frees its callback port.
- `144890829b` — Plugin usage hook runs under `run_bounded_sync`; base no-op spawns no thread.
- `4243abd633` / `cb24121a8e` / `21cf53b888` — `/model` listing skips OpenRouter catalog GET and saved-endpoint `/models` probes (cache-only).
- `7fec7cf594` — Profile-owned catalog keeps curated list when live fetch fails.
- `e22e33c189` — First-time setup resolves plugin catalogs like `/model` picker.
- `6f12165a94` — Every plugin provider admitted to picker by slug; catalogs fall back to profile.
- `de9e45f508` / `3c5d4b0578` / `6040512dab` — External-process provider catalogs route through `profile.fetch_models`.
- `22beb95427` — `external_process` catalog fetch degrades to `fallback_models` on error.
- `b3fa41b811` / `2529b2c67b` / `de66512ab7` — Codex browser PKCE login: invariants + plugin helper + IdP fake.
- `21ef1b97f9` — Proxied Codex routes resolve the Codex OAuth window, not the direct-API catalog.
- `5dca47a651` — MCP preserves application results after transport recovery.
- `e9c7ca4fb1` (spot-check needed) — possible plugin catalog auth handoff fix.

### Managed local llama.cpp runtime (`hermes desktop --local`)

- `43e67d872f` — `feat: local models — managed llama.cpp runtime with one-click desktop setup`. CLI gains a managed llama.cpp runtime (engine install, model download, server supervision); Desktop grows the full setup and management story behind `--local` launch flag. Backend routes + CLI always live. Curated GGUF catalog with per-machine variant selection.

### Providers / models / catalogs

- `f159e581c7` — `models`: add GPT-6 Astra + Astra Pro (fast/flex speed tiers) to Nous Portal and OpenRouter.
- `2c315ff59b` — OpenAI: GPT-6 Astra baseline support.
- `0bbf7b7997` — Agent: extract residency claims from Codex OAuth JWT.
- `299d86851c` — OpenAI: keep Astra 900K alias gated and wire-compatible.
- `17c7be1485` — Models: preserve live-verified Astra 900K opt-in.
- `328542d807` — OpenAI: revalidate Astra at cached and saved-model picker boundaries.
- `5a82258626` — OpenAI: keep Astra rules on eligible routes.
- `0d2cb2cacc` — Streaming compaction on Astra OAuth gate.
- `6a2452393f` — Extend reasoning stale-timeout floor to gpt-6-astra; pin gpt-5.6 family.
- `33b2461be4` — Enable Astra native compaction on official Codex OAuth.
- `170a73637d` — Drop `prompt_cache_options` from Astra requests (not an SDK kwarg; 30m is server default).
- `fdb76ce643` — OpenAI: include canonical API picker identity in Astra discovery gate.
- `850680fcdf` — OpenAI: require canonical host for Astra cache.
- `3825d25191` — OpenAI: cap Astra Codex OAuth fallback.
- `08b140d14e` — gpt-6 Astra on Codex OAuth gets the 85% compaction autoraise.
- `9f069a1175` — Add `anthropic/claude-fable-5.1` to OpenRouter and Nous catalogs.
- `627a12a90e` — `opencode-go`: show Go plan windows in `/usage` through the profile hook.

### Cron (large reliability story)

- `0469740ab3` — Jobs follow the main agent model at fire time; `pinned` locks it on request. Resolution is per-job pin > `cron.model` / `cron.model_provider` > `model.default`. `pinned=true` writes current provider+model onto the job; `pinned=false` releases both. `hermes cron resnap` retained.
- `498abb677e` — A successful cron run resolves the job's open incidents; a repeat re-opens them (durable incident ledger).
- `b028fe632e` — Add `cron doctor` health check.
- `c43891e9f8` (spot-check) — Cron inject skill config into scheduled runs.
- `c164e12bd8` / `43891e9f8e` — Skill-backed jobs receive skill config block.
- `e7cb478db6` — Cron sanitize monitor runtime prompt data.
- `7aa466665b` — Alert once per failure incident instead of every run.
- `3f1bf45adf` — `docs(cron)`: permanent-error parks block the job; reconnecting notice is once per outage.
- `66a91b7ce8` — A server parked on a permanent error blocks the job again; warn once per outage.
- `6924cc66fb` — Run a job whose `enabled_toolsets` MCP server is only reconnecting, instead of blocking it.
- `9f48f4ed04` — Cron delivers `[CRON_FAILURE]` evidence verbatim (no provider heuristics).
- `0a425ddb07` — Cron failure-marker tests trim to invariants; `[CRON_FAILURE]` documented.
- `f26e436d35` — Cron records agent-declared failures.
- `4c0ad43dfe` — A delivered failure notice keeps its real error when the fire-claim heartbeat sample misses after delivery.
- `132a181d80` — A delivered run keeps its `ok` status when the fire-claim sample misses after delivery (#105861).
- `eafed27cf0` — A completed run keeps its result when a fire-claim heartbeat sample misses.
- `1131fd56c2` — Cron reaps stale executions before the one-shot early return.
- `fba9b930f7` — Drained degraded marker is posted without the generic cronjob header.
- `1d140a01ea` — Pin the quota-hold scheduler wiring through a real `run_one_job` tick.
- `15a3691404` — Degraded marker uses the redacted job name.
- `715f21ffd9` — Degraded marker is a plain body wrapped by the standard cronjob header.
- `bdd89b1ec8` — Drop the fire-fence ERROR dedupe from `cron/jobs.py`.
- `2d0867695d` — One `claim-TTL` headroom constant shared by the job store and the ledger.
- `eddf7283e4` — Label overdue next runs in `cron list`, dashboard, Desktop; one parser.
- `8f6f9e5b36` — Order status next run by actual instant.
- `63f09c2d2c` — `cli`: `/cron run` reports a refused run instead of "Triggered … next tick".
- `0502f35d2b` — Name the home-channel target when a non-push session's `deliver=origin` job is rerouted.
- `9013fcdc87` — An unreadable cron toolset restriction fails the run instead of granting every tool.
- `134ef6454d` — Self-removed runs leave no output directory; skip `mark_job_run` on crash.
- `143c1d0a0` (verify) — cron preflight: blockers and ack ledger.
- Bot Chat delivery hardening (multiple): `4d65eca72b`, `fa939860ca`, `b9f4b88034`, `151b070107`, `00ecaf7721`, `8be901ac0f`, `625070d5b7`, `4ce832eeb9`, `69da0b12c4`, `0db097dc22`, `8235afad5d`, `3da502b67a` — bot-chat delivery no longer loses alerts to timeouts, escapes on decode errors, caps the bot's turn (not exit linger), redacts the deferred record, parses non-dict receipts without wedging.

### Voice / STT / TTS / transcription

- `9823ce4ff1` — Streaming TTS first-sentence threshold configurable (`tts.streaming.min_len`).
- `744945b3ba` — Honour `tts.streaming.min_len` on Desktop client-direct TTS path.
- `86bb3eb7c5` — Pin `tts.streaming.min_len` at the speaker and speak-stream entry points.
- `471ef5f4c3` — Desktop direct dictation timeouts per `stt.openai.timeout` (no hanging requests).
- `0830b7f741` — Cap client-direct STT uploads at 60s.
- `aa50383030` — QQ STT timeout at HTTP call-site; share 60s default and number parsing.
- `749f657fbf` — `stt`: coerce new client keys, apply 60s default to QQ STT, document both.
- `0690fc1965` — `stt`: configure OpenAI client timeout and retries.
- `935eef50ef` — Pin `consent_attestation` on Desktop client-direct TTS.
- `f16083763a` — Forward `tts.openai.consent_attestation` to every OpenAI-compatible TTS request.
- `38029a5204` — Trim voice picker tests to invariants; document ElevenLabs model ids.
- `71df3e5aee` — Expose ElevenLabs v3 and gpt-transcribe in Voice pickers.
- `4b4e8ed4bd` — `stt`: keep the provider's 5xx when no transcode is possible.
- `f185bf7f9d` — `tools`: reach STT transcode-and-retry on 5xx container rejections.
- `3e212b7dec` — `tts`: honour PCM sample rate reported by OpenAI-compatible endpoints.
- `e98090e09e` — Desktop wake voice loop from stable reply edge.
- `93dc760d8a` — Desktop streams voice replies from live message deltas.
- `27a30d8515` — `setup`: xAI TTS wizard checks `XAI_API_KEY` before OAuth.
- `ca3c425627` — `tts`: pin `_xai_requirements` to key-first credential resolution in tests.
- `d7865939af` — `tts`: xAI TTS availability probe prefers `XAI_API_KEY` like synthesis paths.
- `3b0aae77ee` — `tts`: prefer explicit `XAI_API_KEY` over subscription OAuth in streaming TTS.
- `8df0a03793` — `gateway`: voice input re-routes per speaker through the identity seam.
- `f4ed8b56e` — wake-word "hey hermes" ear + speaker live in the mic's fan (Desktop).

### Vision / media

- `0a545f7fa5` — Decode HEIF/HEIC/AVIF (iPhone photos) — magic-byte sniff of ISO-BMFF `ftyp` box; bounded pillow-heif dep.
- `f5a7dc14c7` — Parse `ftyp` compatible-brand list; don't gate AVIF on pillow-heif.
- `352d8b0929` / `016e87d20` — Bounded pillow-heif + uv.lock regen.
- `b1f003e186` / `7777f8c350` — FAL GPT Image 2.5 + OpenAI GPT Image 2.5 generation/editing selections.
- `255b4fd9da` — Image-gen records token usage as soon as billed HTTP 200 lands.

### Security / approvals

- `3933fdf63b` — Route remote-party-supplied URL fetches through the SSRF guard.
- `d3fc0cca0f` — Escape OAuth error parameter in callback HTML (reflected XSS fix).
- `1c0d95badb` — Write-deny `HERMES_HOME` secret stores; keep control files writable.
- `e7cd1848c9` — Deny writes to read-blocked Hermes credential stores.
- `56d2438a45` — Secure `HERMES_HOME` when only an ancestor path is a symlink.
- `d966b34cc3` — Gateway lifecycle guard recognises Windows command spellings.
- `48257c6e31` — Stewardship repin for security fix.
- `2bb9c4e693` — Desktop approval-mode zap shows on the status bar by default.
- `aa0beef684` — Corrupt-token and docker_env warnings stop logging credential material (#102308).
- `f6234d00c5` — **Close GitSpawn RCE class** — malicious repo `.git/config` no longer executes on context gathering (GHSA-7x36-8jrh-v4pw).
- `1aa62ceb45` — Isolate multiplex dotenv reloads.
- `c445987559` — Secure built-in memory lock files.
- `92aa6fa14c` (verify) — `init/systemd/RDP` hardening.
- `53c57871d6` — Snyk MCP server + `snyk-security-scan` skill in catalog.
- `459a87b6` (verify) — IMDS metadata endpoint gating already shipped in prior 2026-09-13 window; this window reinforces it.
- `e0eb2ff0a8` — Let the Reconnect action redial a rejected active gateway.

### Auth / credentials

- `f32e59a01` (verify), `5d0b3cc8c` (verify) — credential-pool rework.
- `0d2cb2cacc` (already listed) — streaming compaction on Astra OAuth gate.
- `ec1238fa65` — Credential pool plugin refresh; tokens, locks, dead grants.
- `ec015c906c` — Plugin-guard: plural test-file names and delegation prose are not attack shapes.
- `92bb5b92b8` — Revert a quota-benched credential through the pool, not a private probe.
- `4c5a70065c` — Arm the pool revert only when benched credential outranks rotated one.
- `0dd153ba2e` — Revert pool rotation after transient quota cooldown expires (#114501).
- `9e0aac93d` (verify) — oAuth + Anthropic PKCE mirror.

### Gateway / multiplex

- `78deeacf68` / `a613d4f961` / `56fa925bcf` / `e702554e15` — TUI-gateway: exercise launch cwd from profile path; isolate session cwd by profile; pin resume overrides to session profile with two real homes; resolve deferred/cold resume overrides under session profile scope (#115607).
- `1d847a8542` / `2df8ac4fbb` — Two invariant tests for profile-scoped reconnect threshold; gateway reads `reconnect_attention_after` from bound profile's config, not env bridge.
- `b5e8bb72b0` — Curator/skills-sync housekeeping ticks under each served profile's scope.
- `d7b836ab1c` — Stack `@method/@_profile_scoped` on vault handlers.
- `ae7126353f` — `vault.*` RPCs bind the launch profile's secret scope in multiplex.
- `94218e192e` — Identity-first session keys; mark boot-only ambient profile reads.
- `d6f3285b81` — `feat(gateway): route inbound messages to profiles by sender user_id`.
- `15a256293d` — Desktop gateway/profile group headers take the pointer activator only.
- `a1cbfc546c` / `3b9cb0d076` / `e5fe22646e` — Update multiplexer covers every served profile in fleet check; clear fleet restart warning.
- `834187bffc` / `63e4e040bc` / `d6a58adbc5` — `tui_gateway`: `reload.mcp` rediscovers launch profile's MCP servers; export session profile to tool subprocesses; 401 sign-in hints name the provider and failing profile.
- `d8edd60399` — Wake a served profile's `api_server` session in-process for background-process completions.
- `eeada33fa1` — Review retry the cap, adopt compression tip, scope in-process route to multiplex.
- `bbdaa89a47` — Keep multiplexed direct platforms awake.
- `f6978426a2` — Unresolvable profile store preserves the flush file instead of writing to root store.
- `131e670974` — Heartbeat idle gate probes the default profile under a named-profile multiplexer.
- `2e6247fb3e` — Gate per-profile watcher ticks on actual work.
- `8039bb1e61` — `tui_gateway`: lease takeover stays within profile; spares new owner's row.
- `fd1582f63b` — Drain Windows profile stop before kill.
- `51c4b6ba9d` — Standalone gateway binds the launch profile's own scope after hosted activation.
- `213dc82a3a` / `a213657500` — Preserve profile context for interrupt reapers; reproduce reaper profile-scope loss.
- `0bbf7b7997` (auth-adjacent) — Agent extracts residency claims from Codex OAuth JWT for workspace auth.
- `aaaca43c84` — `gateway`: one wildcard-host predicate for HTTP listeners.
- `d6f3285b81` — `route inbound messages to profiles by sender user_id` (Discord-style guild routing generalized).

### Agent / loop / delegation / recovery

- `0752127c5e` — Bounded auto-recovery ladder after retries and fallback are spent (#85426, #107307).
- `a79d1d3a71` — One-shot runs drop the self-improvement footprint (no skill authoring, fewer process skills, delegation cap).
- `226cfb56f` (verify) — extend auto-recovery ladder.
- `f22b7cedf9` (MERGE) — partial 429 retry at reset.
- `b9f29e2180` — `api_server`: stream agent status lines as `hermes.status` SSE events.
- `efcafea066` — Label a spent image-shrink fallback (was "Non-retryable error (HTTP 400)").
- `270fe15c8b` — Route oversize rejections arriving as HTTP 400 to image-shrink recovery.
- `6527af2286` — Retry once without `stream_options` when endpoint rejects HTTP 400/422.
- `ec015c906c` — Plugin-guard rejects plural test-file names and delegation prose as attack shapes.

### Delegation / subagents / kanban

- `7840a0e2d9` — Delegation batch tags read "set N" instead of hex id slice.
- `2e58fb3b04` — Provider-copy community plugin entry.
- `8b01df963d` — Expose session-scoped subagent roster and live tail RPCs.
- `34e539ebeb` — Collapse Classic CLI subagent dock with F7.
- `f7fe32bfde` — Compact Ink agent dock + Enter live tail.
- `07c9088460` (reverted) — Browse skills+plugins in a shared native catalog.
- `3b9cb0d076` — Update multiplexer-coverage marker test writes inventory main's marker.
- `6f24245532` — Kanban gates `create-with-parents` like `link`; archived parent is terminal.
- `ad0398eed8` — Kanban preserves sticky block on tasks created with `initial_status=blocked` (#107398).
- `dc5c87c6c1` — Kanban honours explicit scratch/project on MCP `kanban_create`.
- `3b7ff435fd` — Kanban preserves durable origins for worker-created tasks.
- `40da71dbb4` — `docs(kanban)`: name the legacy-attachments cutoff (#35395) in the type comment.
- `938919502e` — Split `complete_task` into phase helpers.
- `c633aee918` — Split `create_task` into project-link + skills helpers.

### Profile / cloning / cross-profile

- `7bb52c0b74` — Profile clone can opt into staying synced with its source (`--sync-imports`).
- `67757285f6` — `hermes sessions repair-profiles` settles crossed-profile durable state.
- `9f2a1ad8f3` — `mcp_tool_schema`: preserve empty `required` arrays in `_repair_object_shape`.
- `798cc60f4c` — Repair schema-map keywords per-entry in `_repair_object_shape`.
- `da942e4483` — Hedge the `config get` phantom-key notice (schema walk has false positives).
- `5abda7e721` — Flag schema-unknown nested keys in `hermes config get`.

### Compression / context / API-sidecar

- `33b2461be4` — Enable Astra native compaction on official Codex OAuth (already listed).
- `bce20d0b1f` — Provider plugin error classification already listed.
- `1939c1b4d` (verify) — `api_content` sidecar is enforced on the wire.
- (No observed schema version bump for compression FTS this window — only behavior refinements.)

### CLI / TUI / Desktop

- `a5e934758d` (verify) — `/model --once` semantics.
- `85b2a3df6c` — `hermes usage [--json]` prints `/usage` account limits without a session.
- `5a0fb0f38d` — Desktop organizes settings into ordered subpages.
- `ee56d38d97` — Desktop allows hiding thread timeline bars.
- `4d14aaf477` — Desktop exposes "hide code diffs" in Appearance settings.
- `0f2c27152d` — Desktop keeps edit counts when code diffs are hidden.
- `e0eb2ff0a8` (already listed) — Reconnect action redial rejected active gateway.
- `6c5764cddc` — Desktop stop reconnecting rejected gateway sessions.
- `ca396e0f7f` — Desktop tracks plugin gateway listeners in bundled loader and `ctx.onEvent`.
- `14b2ad1596` — Desktop dispose runtime plugin event listeners.
- `8170c16cca` — Route every desktop-plugin tree copy through staged publication.
- `27fc1d9fb2` — Recover interrupted unified plugin copies.
- `1b2da20783` — Desktop attaches frontmost window to active draft.
- `77a5457343` — Desktop organizes sessions by gateway and profile.
- `4898326ed2` — Desktop routes `message.react` through session's owner, not ambient gateway.
- `6f6ab904f2` — Desktop gateway switch no longer keeps previous gateway's workspace folder (#114306).
- `e19efba7f0` / `04a8c014d6` / `c97b5b5696` — Desktop: never leave e2e remote gateway orphaned; accept-then-close proxy must not reset reconnect backoff; only reset after stable open (#83134).
- `8ece9242a6` — Desktop compares moved agent's legacy URL with shared gateway normaliser.
- `b978eaab43` / `9478d543da` — Desktop catalog plugin additions (agent-log, my-github).
- `5a153411fe` — Anthropic streamed refusal `stop_details carry` is red when any wire hunk reverts (test).
- `d8a479e3a` (verify) — Desktop polish.

### Honcho / memory

- `c04722102f` — Honcho: declare injection block in `config_schema` so desktop panel can pin `sessionStart`.
- `56231e51ad` (already) — Honcho advances read baseline after each write so revert reaches disk.
- `52f8a1d28` (verify) — Honcho peer-model follow-ups.

### Web / search providers

- `3e9a2b51d` (verify), `36f4ea04` (verify) — Firecrawl / Tavily / Parallel / Exa tweaks.

### Secrets / vault

- `d7b836ab1c` (already) — `vault.*` RPC profile-scope stacking.
- `ae7126353f` (already) — `vault.*` RPCs bind launch profile's secret scope.
- `cde0ad0edd` — Agent signs into sites from an encrypted local vault (CLI, browser fill, Desktop Settings).

### MCP

- `ceb1aa19a0` — MCP restores proxy support for HTTP/SSE MCP servers.
- `b2465f1608` — MCP auto-fallback to SSE transport when Streamable HTTP returns 400.
- `a6fdadfcee` — MCP cap HTTP/SSE response bodies before SDK parse.
- `f0c26f0554` — MCP names the HTTP status, URL, and body behind "Server returned an error response".
- `0c74353c86` — `security(gateway)`: re-resolve hooks directory per call to fix profile isolation.
- `95987fb85a` — Bypass the proxy on loopback HTTP CDP discovery and keep `NO_PROXY=*` intact.
- `9c1ef6bfcf` (verify) — MCP HTTP keepalive defaults documented.
- `9c2cf8a4b` (verify) — MCP schema/per-server trust tier refinements.
- `ebc41a8d5` (verify) — MCP OAuth parked-revival.

### Cron continued (Bot Chat delivery)

See Cron section — bot-chat delivery is the largest non-discrete-crisis reliability work this window.

### Reverts (worth listing — operator-visible)

- `3eb9712180` Revert "feat(catalog): restore website install links to Desktop".
- `007e80ee0f` Revert "docs(catalog): restore native browsing and install guidance".
- `086628ad8a` Revert "feat(ui): support icon-only segmented controls".
- `8bd0da2b8c` Revert "feat(desktop): restore native skill and plugin catalogs".
- `f923398fee` Revert "feat(desktop): browse catalogs as cards with a saved list option".
- `9796235822` Revert "feat(catalog): open website installs in Hermes Desktop".
- `4718ba7352` Revert "feat(desktop): browse skills and plugins in a shared native catalog".
- `9b8cc18364` Revert "docs(catalog): explain shared feeds and Desktop install links".
- `9d67c7c29a` Revert "fix(desktop): show skill install progress and completion in the shared dialog".
- `9139bb8a4e` Revert "fix(catalog): align CI checks with the shared action row".
- `6a912eb1cb` Revert "refactor(desktop): reuse ActionStatus for catalog installs".
- `0dcadf6f41` Revert "remove onboarding catalog additions".
- `696662997b` Revert "refactor(gateway/hosted_rooms): reflow multi-line SQL constants".
- `530b443b66` Revert "drop the 12-minute no-progress tripwire".
- `0dd153ba2e` Revert pool rotation after transient quota cooldown expires (#114501).

**Net reverts this window:** 15 — most concentrated in Desktop catalog browser feature experimentation. Two follow-up fixes (`92bb5b92b8`, `4c5a70065c`) re-applied the pool rotation logic with corrected triggers.

### Salvaged / ported PRs

- `30657d197d` `refactor(update)`: psutil Android installer extracts through shared safe-tar guard (porting a desktop safety pattern to mobile installer).
- `2b55ded1ac` `perf(state)`: keep delegate-child transcripts out of the trigram FTS index — schema v30 (perf, not migration).
- `2031c819fe` — `simplify(compat): gateway` re-land 92d0bd0d731 (reverted by stale-index commit b818085298e).
- `2776813df3` — `compat(plugins): temporary import-path shims for external plugins — ONE commit, revert on schedule (2026-09-14 cutoff from prior window)`.

## Action items for next full reference re-compile

The `Hermes_Architecture.md` reference is **+10 weeks stale** relative to upstream and now ~6,000 commits behind. The reference document must be re-compiled to cover:

1. **Plugin ecosystem** — major new section required. Plugins now own: auth (`749443e292`), OAuth (97fd8c8f2a), error classification (`bce20d0b1f`), capability metadata (`ef0bac6cdd`), usage hooks (`144890829b`, `400b6c41cd`), local-runtime backend (`43e67d872f`), catalog surface, MPP/catalog cards. Catalog as distribution surface + plugin_guard on install.
2. **Model-provider plugin system** — separate from legacy provider routing; new `ProviderProfile` class, plugin-defined `auth_handler`, plugin-mirrored vs native provider classes, third-store credential plumbing.
3. **Managed local llama.cpp runtime** — `hermes_cli/local_runtime/`, engine install, model download, server supervision, Desktop `--local` UX, curated GGUF catalog with per-machine variant selection.
4. **GPT-6 Astra model family** — gating, picker/host validation, Codex OAuth eligibility, 900K alias, native compaction, streaming compaction gate, residency claims.
5. **Cron reliability — failure-incident ledger** — one alert per outage, complete-on-success/reopen-on-replay, pinned model semantics, blocked-job audit (`blocked_config`), `cron doctor`, degraded-marker posting contract, `[CRON_FAILURE]` verbatim, quota-hold scheduler wiring, claim-TTL headroom, bot-chat delivery hardening.
6. **Voice/STT/TTS unification** — `tts.streaming.min_len`, configurable OpenAI-compatible endpoints, xAI TTS key-first, ElevenLabs v3 + gpt-transcribe, voice input identity-seam routing, HEIF/HEIC/AVIF vision decode, GPT-Image 2.5.
7. **Security** — `HERMES_HOME` write-deny for secret stores, IMDS approval gate, GitSpawn RCE class closed (GHSA-7x36-8jrh-v4pw), OAuth callback XSS fix, MCP User-Agent, vault-scoped secret access.
8. **Profile/clone** — `--sync-imports` flag, `repair-profiles` for crossed durable state.
9. **Catalog as a product surface** — plugin-catalog backflow, MPP entries (templating/migration), pinned entries (review/PR rerouting), pstack/Adspirer, etc. Catalog is no longer just `plugins:` source-of-truth — it has its own maintenance contract.
10. **Desktop catalog browser** — DISCONTINUED via reverts; reference should document the cards-only experience as the supported UX.

The next re-compile should consume this changelog's SHAs as its scaffold, merging into the existing section structure of `Hermes_Architecture.md` 1:1.

## Known limitations of this pass

- Local Hermes is at v0.21.3 (4,064 commits behind upstream `hermes --version`); reference remains based on the 2026-08-31 v0.21.0 baseline + weekly changelogs.
- 4,175 commits is too large for a single-pass re-compile under the "lightweight local model only" cron directive; changelog-as-scaffolding preserves the audit trail.
- Many commits not shown above are pure fix/fix-test patches; large commit-bucket counts (e.g. `fix(gateway)`: 434) include both behavior changes and test maintenance. Behavioral change density is ~5-10%.
- Some SHAs (those marked "verify") were not spot-verified with `git show --stat`; treat their content as inferred from commit message + category clustering, not confirmed.
- Cron "Bot Chat delivery" subsection references partial SHAs and may overlap with the main cron section.
