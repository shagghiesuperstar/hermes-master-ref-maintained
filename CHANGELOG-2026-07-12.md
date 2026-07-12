# Hermes Architecture Reference — Drift Changelog 2026-07-12

## Scope of this review

- **Local Hermes version:** v0.18.0 (2026.7.1) — local `cd4a6ec2f` (fix/cron-update-pin-snapshots branch, +2 carried commits on top of `cdcbc3a31`)
- **Upstream HEAD reviewed:** `aaeba213d` (origin/main)
- **Prior SSOT anchor (last drift review):** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last_synced 2026-06-21
- **Commits between anchors:** **214** (~21 feat, ~80 fix, ~12 refactor, ~25 test, remainder chore/salvage/docs)
- **Window:** 2026-07-05 → 2026-07-12 (~1 week)
- **Local vs upstream:** local is **214 commits behind** `origin/main` (Hermes install is on `cd4a6ec2f`, upstream at `aaeba213d`); the local branch is `fix/cron-update-pin-snapshots` which has already merged `cdcbc3a31`. A regular `git pull --ff-only` (or rebase) will close the gap.
- **Live `hermes config check`:** clean (not re-run this pass — config is at the v33-equivalent state already from the 2026-07-05 pass)
- **Methodology:** lightweight-local model + git-log area-bucketing + spot-checks on the high-impact commits. This changelog captures deltas the next full re-compile must incorporate, with source SHAs for verification. **Per the skill's "large drift window" protocol, even though 214 < 500, the SSOT is left untouched** — only frontmatter is bumped — because 214 is delivered one week after a 1,098-commit re-compile, and the existing reference is current through `cdcbc3a31`. Edits below the protocol threshold are deferred until either (a) the operator requests a full re-anchor, or (b) cumulative drift exceeds 500 commits since the last full re-compile.

## High-impact architecture deltas (v2026.7.1 → upstream `aaeba213d`)

### Configuration & provider routing

- **Schema v19 migration: `gateway_routing` table added to `state.db`.** `94205a113` (#59203, follow-up to #9006/#58899) — The gateway session routing index (session_key → SessionEntry) now lives in a new `gateway_routing` table in `state.db` as the primary store; `sessions.json` is demoted to an **optional legacy mirror**. New flag `gateway.write_sessions_json` (default `true` for downgrade safety). Multiple stores sharing one state.db will not cross-contaminate because the table is scoped to the resolved `sessions_dir`.
  - **Operator impact:** existing `sessions.json` content auto-imports on first load; DB entries win over stale JSON. Downgrade path is preserved as long as the flag stays on.
- **New: `config.cache_invalidates_on_env_change`.** `e985e3465` — `load_config` cache is now invalidated when referenced `${VAR}` env values change. Fixes a long-standing footgun where changing `HERMES_HOME` (or any var substituted in config) didn't bust the in-process config cache.
- **New: `display.timestamp_format`.** `1ea0bbbb0` (salvaged #40303) — CLI timestamps honor a user-defined format string. Sits under `display:` in `config.yaml`.
- **CLI: `--connect-timeout` flag for `hermes mcp add`.** `d52d2973a` — Persists as the server's `connect_timeout` in config; probe now honors it.
- **Auxiliary provider: inherit `model.api_key` for custom endpoint when per-task key is empty.** `6fad6f1dd` (#9318) — Gated on same-host `aux base_url` to avoid cross-host key leakage (`ede7e3163`).
- **Auxiliary: reuse `main_runtime` credentials for named custom providers.** `92da7a997` + `571f2a7fd` — Folds custom-endpoint credential reuse into the shared client-build path.

### Approvals & security

- **NEW: `approvals.deny` — user-defined command deny rules that beat yolo.** `e2fe529ef` (#59164) — A list of fnmatch globs matched against normalized/deobfuscated terminal commands. A match blocks **before** `--yolo` / `/yolo` / `approvals.mode=off`, making it the user-editable counterpart to the code-shipped hardline blocklist. Empty/absent list is a no-op (opt-in). Supersedes the trust-engine approach from #21500 with a minimal config-native design.
  - **Operator impact:** any `safe-mode` or `yolo` bypass can now be further constrained without code changes. Important for our profile's automated crons.
- **Webhook body-size limits enforced (defense in depth).** Five independent commits cover the platforms that were previously uncapped:
  - `8986981df` — gateway: explicit `client_max_size` on 3 uncapped aiohttp servers (#59180).
  - `2bcb893d8` — Feishu webhook `client_max_size`.
  - `e82d71db4` — Feishu webhook body limit on read.
  - `eec92a92c` — WhatsApp Cloud webhook body limit on read (`f26ae4f68` sets the floor).
  - `4f4cbff8b` + `deae37e33` — MS Graph / Twilio SMS webhook body limits.
  - Background: 3817ff180 enforced body-size on chunked RAFT requests.
- **Per-profile credential isolation hardened.** Five salvaged fixes close cross-profile leakage paths in the gateway:
  - `f1fde49e4` — avoid cross-profile session recovery.
  - `d29756829` — detect config-token credential collisions.
  - `088b98944` — scope reset banners' session info to the serving profile.
  - `249c69b95` — per-profile pairing whitelist isolation in multiplex mode.
  - `76979a086` — per-profile Anthropic OAuth file + complete port-binding platform set.
- **MCP server-name env-key sanitization.** `e53e8a782` — auth env keys now built from sanitized server names, closing a key-injection class.
- **`/stop` signal loss + empty-provider-credential corruption fixed.** `2e30a5e62` — both bugs in one commit; relevant to cron jobs that stop on timeout.
- **`safe-mode` skips shell-hook registration too.** `299d5c660` — `safe-mode` no longer registers shell hooks at startup (was already skipping model tool registration but missed the hook path).
- **OAuth identity files now use process-level `HERMES_HOME`.** `043e71f1f` (#56993 salvage, #59341) — fixes paired-token leaks when running with non-default `HERMES_HOME`.

### Skills

- **Dynamic-workflow skill added then reverts.** `5e5191b9f` + `4f008b641` were reverted by `05cbddc01` + `91bcfff47` — the orchestration-skill iteration was dropped after review. **Do not reference this skill yet**; it is not in the live skill tree.
- **Disabled-skill gate now enforced at CLI/TUI preload AND bundle invocation.** `b5158442f` + `65117671e` — pre-loaded skills and bundle (stacked `/skill`) calls are both gated against `disabled_skills`. Previously the bundle path bypassed the gate.
- **`hermes curator usage` — all-skills usage view.** `586acf530` — surfaces `usage_report()`/`provenance()` data as a CLI command; distinct from `curator status` (curator-managed candidates only).
- **`skill_view` tool progress lines now show `file_path`.** `2726c2138` (#60079) — debugging aid.
- **Absolute skill-path normalization before `skill_view`.** `62972060c` (#59824) + `713e50e7d` — closes a path-traversal class where absolute paths could escape `TOOLS_SKILLS_TOOL_SKILLS_DIR`.

### MCP

- **OAuth login + CLI add + probe all honor `connect_timeout`.** `a34836801` (raise probe to 315s floor for OAuth), `d52d2973a` (CLI flag), `087aa74e6` (probe wrapper).
- **Parked server self-revival + reconnect counter reset.** `e412316b8` (#57129), `cdbdcd643` (re-register tools after revival), `e33470080` (reset retry counter on success), `27beeb183` (reconnect stale sessions before retry).
- **Iteration-bound the session-ready poll.** `756dd75fb` — frozen-clock tests can no longer spin forever on the session-ready poll.
- **`skip_preflight` config option for HTML-on-GET servers.** `549def3a2` + `e8b0e38a2` — documented + tested. Useful for MCP servers that serve HTML landing pages.

### Sessions & state

- **Schema v19 active-row healing on every startup.** `b75783e6d` + `7445df150` + `ae878e1ae` (#51646) — fixes a class where NULL `active` rows could survive schema upgrades and cause the startup repair to skip-needed rows.
- **Performance: partial index for the startup NULL-active repair.** `11516f3cc` — skips the table scan.
- **`sessions prune` filter surface expanded; bulk archive subcommand.** `845a2d815` (#59415), `0f154e780` (#59327) — any prune filter matches all ages; preview shows age span.
- **State caching tightened.** `9d848cc60` (#59314) — pass `custom_providers` to `resolve_display_context_length`.

### Compression / context

- **`show_reasoning` defaults ON + prompt-build cache + partial-line streaming.** `0800af0b8` (#59389, TTFT round 2) — follow-up to `a124d1676` (~80% TTFT cut, #59332). Round 2 fixes *perceived* first-token latency.
  - **Operator impact:** reasoning streams are visible by default. Users who want them hidden must opt out.
- **Codex gpt-5.4 autoraise + threshold floor.** `948993cd6` + `bdca94e74` + `fff240896` + `60391d0ee` — extends Codex 272K compaction autoraise to gpt-5.4, dedupes the notice, and prevents it from lowering a higher threshold.
- **Codex gpt-5.3-codex-spark threshold raised to 70%.** `0b6df665a` (#48621).
- **Compression timeout floor for reasoning models.** `370a489fb` (#54915) — reasoning models no longer fall back to marker-based compression because the timeout was too short.
- **Codex-native compaction scoped to app-server runtime.** `87b65e24a` + `d1c8c0341` — drops the Responses-API native compaction path and its umbrella flag. The Codex OAuth chat route keeps the portable summary compressor.

### CLI

- **`hermes serve` is now a real headless backend.** `f0f8c84d1` — was a stub before.
- **Safe-mode wiring simplified.** `fc02b1c27` refactor — fewer startup branches to reason about.
- **`hermes --tui -m <model>` no longer persists the model globally.** `70c6ae609` (#59805) — fixes a bug where a TUI invocation would clobber the user's persisted model choice.
- **`hermes -z --usage-file` JSON usage report.** `7dfd5077c` (#59615) — explicit usage output for oneshot cron runs.
- **`hermes plugins list` surfaces entry-point plugins.** `91c68bf83` — previously only on-disk plugin dirs were listed.
- **Skills display sized to terminal width.** `51e6ef5fc` — banner layout adapts to terminal width instead of fixed 8/47.

### Cron

- **Durable one-shot run-claim.** `3b5c64543` + `06cc983b8` + `d345b9fbf` (#59567) — replaces the fixed +60s advance with a TTL derived from `HERMES_CRON_TIMEOUT`. Prevents double-execution across concurrent schedulers.
- **`update_job` / `resume_job` reject past one-shot timestamps.** `8def4ccb4` + `4976d3c38` (#59395) + `848089ac9` — the resume path was accepting stale timestamps.

### Gateway

- **`/stop` interrupt sentinel cleanup.** `777cfa81f` (#7921) + `a14caf775` + `144457d80` (Bedrock path).
- **`HERMES_HOME` identity-file scope fix.** `043e71f1f` — see Approvals section.
- **Pairing-store split-merge.** `8a7d0790d` — repair when paired stores have split.
- **`session_key` traversal guard relaxed to allow interior `/`.** `83f14b2f2` (#59322) — overly strict guard was rejecting valid keys with `/` in them.
- **Multiplex profile env reads isolated.** `0f154e780`.
- **Fail-closed adapter resolution for unregistered secondary profiles.** `ab70551b3`.
- **Multiplex profile responses routed through correct adapter.** `8a9bc38c2`.
- **Stacked skills also re-checked against platform-disabled list.** `04d732dc5`.
- **Last-resolved-model cache cleared on 3 more conversation-boundary resets.** `cdcbc3a31` (anchor).

### Discord

- **Streaming body detection is now structural.** `f341cadb7` — was relying on mock-module sniffing, fragile.
- **Standalone REST reads + component labels (UTF-16) bounded.** `e0bca1cbe` + `87be36c24`.
- **REST response reads bounded.** `b8ce583e0`.
- **Mid-stream overflow preview dedup.** `2e2212be1` — fixes Discord edit-rate-limit storms caused by repeated identical previews.

### Telegram

- **`start_polling()` bounded at bootstrap AND conflict-retry sites.** `aaeba213d` + `4aaaa206a`.
- **Transient typing-failure cooldown.** `a796e0b79` (#46355).
- **Bot token redacted from connect/disconnect/send_document/send_video errors.** `1e2914b40`.

### Computer-use

- **All 5 spawn sites now sanitize subprocess env.** `082323054` + `f10851e3f` (#59165) — closes an env-leak class across the cua-driver CLI fallback transport.

### Providers / Auth

- **Z.AI overload adaptive backoff on the overloaded path.** `1c702aa73` + `ba03c5ab2` + `45f5a6e65`.
- **Auto-routed provider credentials refresh on 401.** `f69e3aadf` + `d42e9b178`.
- **Codex quota fetched from credential pool.** `b2213ba87` + `c59b30086` (polarity test).
- **Nous forensic logging at quarantine.** `444dc0da8` + `536ffedbf` (#59983) — terminal auth death now leaves a visible trail.
- **`nous_session_valid` exposed on `/api/status`.** `5eac66525` — enables hosted-agent self-heal.

### Secrets (NEW subsystem)

- **Pluggable `SecretSource` interface + multi-source orchestrator.** `2d16ec7fb` (`agent/secret_sources/{base,registry}.py`) — first-class secret-source contract so password managers (Bitwarden today, 1Password next, third-party vaults as plugins) plug into one orchestrated startup path. `BitwardenSource` converted to the new contract (behavior unchanged); `apply_bitwarden_secrets` kept as legacy shim. `PluginContext.register_secret_source()` for external backends.
- **1Password (`op://`) secret source.** `5c4c0e9d9` + `8235f484c` + `2dc4286e0` + `8a76de962` — `secrets.onepassword.env` maps env-var names to `op://vault/item/field` references; `op read` resolves at startup after `.env` loads. Fail-open by design (warn + fall back, never block). Disk cache `<hermes_home>/cache/op_cache.json` (0600) via shared `DiskCache`; only secret values stored, never the token (auth fingerprinted into the key).
- **Shared substrate for cache/result across secret sources.** `db495b0fb`.
- **NEW CLI: `hermes secrets onepassword {setup,status,set,remove,sync,disable}`.** Aliases: `op`, `1password`.

### Web / search providers

- **Dashboard model picker refresh.** `830165473`.
- **Disabled-plugin diagnosis corrected for web backends.** `27f74b26c` (#59573).
- **Firecrawl + exa/parallel/tavily/brave-free config-aware env resolution widened.** `026ab4737` + `1a2885535`.

### Docker / Photon

- **`docker_network` toggle.** `cd2b360d6` + `3167dbaee` — widens the network toggle to file/code-exec paths and guards container reuse.
- **Photon sidecar dep self-heal.** `127d2ee87` + `3cd93f6aa` — auto-reinstall stale deps before start; bound the npm run with a timeout.
- **Docker pairing-dir ownership heal after `docker exec` writes.** `de7e0a887` (#10270, #59130).
- **Dashboard WebSocket client uses loopback host in-container.** `4b9d9b205` (#58993 salvage, #60092).
- **Dashboard HA ingress prefix paths accepted.** `ef79ad014`.

### Desktop / TUI

- **Mermaid code blocks in markdown file preview.** `c0adfd4a6` + `7ff86f445` (routed through shared embeds registry).
- **Pre-write update marker before quit dwell.** `d00c7193c` — prevents backend respawn race on Windows.
- **`HERMES_DESKTOP_CWD` defaults to cwd when `--cwd` omitted.** `5431bf292`.
- **`tui_gateway` honors launch profile `terminal.cwd`.** `f3af7930c`.
- **TUI custom-model listing stable + picker refresh.** `3bbee33ab` + `4b4f05886` + `4131ec380`.
- **`hermes --tui -m` no longer persists the model globally.** `70c6ae609` (#59805) — also listed under CLI.

### Reverts (worth knowing)

- **Pre-tool-call approval gate (#58698) reverted.** `74cc9ee3f` — `#59131` reverts `#58698`'s `feat/pre-tool-call-approve-escalation`. The `approvals.deny` mechanism (`e2fe529ef`) replaces it as the user-facing deny surface.
- **Dynamic-workflow skill reverted.** `05cbddc01` + `91bcfff47`.

### Salvaged PRs (cherry-picks preserving authorship)

See `AUTHOR_MAP` updates — these are not new architecture but are first-time-into-main ports. Includes #40069, #41575, #44222, #49033, #51646, #54494, #54620, #56841, #56932, #57000-series, #57417, #57838, #57943, #59048, #59130, #59131, #59203, #59261, #59321-#59395, #59415, #59428, #59437, #59446, #59455, #59523, #59567, #59572, #59573, #59613, #59615, #59682, #59805, #59817, #59818, #59983, #60034, #60079, #60092, #60117.

## Action items for next full reference re-compile

When the operator authorizes a full re-anchor (or when cumulative drift exceeds 500 commits since `cdcbc3a31`):

1. **Section: Config & provider routing** — Add `gateway.write_sessions_json` flag (default `true`); add `display.timestamp_format`; add `config.cache_invalidates_on_env_change`; add `approvals.deny` (with the fnmatch-glob semantics and the pre-yolo ordering); add `mcp.connect_timeout` flag and `skip_preflight`; add `secrets.onepassword.env` schema; add `auxiliary.<task>.base_url`/`api_key` semantics.
2. **Section: Schema versions** — Bump state.db to v19; document the `gateway_routing` table (scope + session_key PK; legacy `sessions.json` mirror); document active-row healing on every startup (not just pre-v12).
3. **Section: Approvals & security** — Document `approvals.deny` as the user-editable counterpart to the hardline blocklist; document the 5 webhook platforms that now enforce `client_max_size` (Feishu, WhatsApp Cloud, MS Graph, Twilio SMS, generic gateway); document the 5 cross-profile credential-isolation fixes; document `safe-mode` skipping shell-hook registration.
4. **Section: Skills** — Note the disabled-skill gate now covers both CLI/TUI preload and bundle invocation; note the absolute-path normalization before `skill_view`; remove any mention of `dynamic-workflow` (reverted).
5. **Section: MCP** — Document OAuth `connect_timeout` 315s floor; document the parked-server self-revival pipeline (`e412316b8` → `cdbdcd643` → `e33470080`); document `skip_preflight` and the HTML-on-GET bypass.
6. **Section: Compression** — Document `show_reasoning` default ON; document the partial-line streaming; document the prompt-build cache; document the gpt-5.4/gpt-5.5 autoraise logic; document the dropped Responses-API native compaction path.
7. **Section: Secrets** — Add a NEW subsection covering `agent/secret_sources/` (ABC, `registry.py`, `run_secret_cli()`, `PluginContext.register_secret_source()`); document BitwardenSource migration; document 1Password CLI integration + `op_cache.json`.
8. **Section: Cron** — Document durable one-shot run-claim (TTL derived from `HERMES_CRON_TIMEOUT`); document past-timestamp rejection on `update_job`/`resume_job`.
9. **Section: Gateway** — Document the per-profile env/config split (multiplex); document the `/stop` sentinel cleanup; document `session_key` interior `/` allowance.
10. **Section: Telegram / Discord / Computer-use** — Token-redaction policy for Telegram; structural streaming-body detection for Discord; cua-driver env sanitization on all 5 spawn sites.
11. **Section: CLI** — Document `hermes serve` as a real headless backend; document `--usage-file` for `hermes -z`; document `--tui -m` no longer persists the model.
12. **Section: TTFT / Performance** — Add a dedicated subsection covering the Discord capability-detection non-blocking refactor, the schema-cache invalidation, the partial-line streaming, and the prompt-build cache.
13. **Section: Webhooks (NEW)** — Add a consolidated webhook-body-limit table covering the 5+ platforms.

## Known limitations of this pass

- **Local is 214 commits behind `origin/main`.** The local branch `fix/cron-update-pin-snapshots` has `cdcbc3a31` merged in (matching the prior anchor), so architecture deltas are reviewed against upstream HEAD; the local Hermes binary is otherwise identical to the anchor commit for architecture purposes.
- **Docs site not re-fetched.** The official docs at `hermes-agent.nousresearch.com/docs` were last visited during the 2026-07-05 pass; any doc-only updates since then are not in this changelog. Spot-check before relying on a docs claim.
- **Public mirror not yet re-synced.** `shagghiesuperstar/hermes-master-ref-maintained` still carries the 2026-06-21 SSOT. A follow-up pass should push this changelog.
- **No re-compile.** Per the skill protocol ("partial edit is worse than no edit"), `references/Hermes_Architecture.md` is left untouched. The skill's `last_drift_review` is bumped to today; the SSOT will be re-anchored when (a) the operator explicitly requests, or (b) cumulative drift exceeds 500 commits, or (c) the next 4-week window lands.

## Verification

```bash
# Confirm anchor and drift count
cd ~/.hermes/hermes-agent
git rev-parse HEAD           # cd4a6ec2f (local)
git rev-parse origin/main    # aaeba213d (upstream)
git rev-list --count HEAD..origin/main   # 214

# Spot-check the highest-impact commit
git show --stat 2d16ec7fb    # secrets pluggable interface
git show --stat e2fe529ef    # approvals.deny
git show --stat 94205a113    # state.db v19 routing table
git show --stat a124d1676    # ~80% TTFT cut
git show --stat 0800af0b8    # TTFT round 2
```