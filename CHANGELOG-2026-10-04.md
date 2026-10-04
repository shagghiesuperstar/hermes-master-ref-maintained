# Weekly drift review — 2026-10-04 (Sunday)

## Anchors

- **Local Hermes version/commit:** `v0.21.5+5376.gb12e672.dirty` · `b12e672e23` (2026-09-24)
- **Local-vs-upstream behind:** 1,604 commits behind `origin/main` (local install is pinned to a pre-`/update` snapshot; the drift count that matters is the SSOT anchor delta below).
- **Upstream HEAD this run:** `8d5e3e4121` (`origin/main`)
- **Prior SSOT anchor (this skill):** `4f22543509` — last full reference re-compile (4,175 commits on top of which this changelog continues)
- **Delta from prior SSOT anchor → upstream HEAD:** **22,139 commits** (cumulative since the last re-compile; the prior 2026-09-27 review only covered through `4f22543509..6f7a7991bb` = 18,857 commits — see CHANGELOG-2026-09-27.md for that pass).
- **Window this run covers:** weekly — last 7 days (`--since=2026-09-27`). **3,259 commits in the weekly window.** Far past the 500-commit threshold that mandates **changelog-as-scaffolding** per the drift-review decision tree. `Hermes_Architecture.md` is NOT re-compiled this pass; this file is the durable scaffold for the next full re-compile.

## Method

- Anchor sweep via `git -C ~/.hermes/hermes-agent fetch origin main` + `git rev-list --count 4f22543509..origin/main`.
- Weekly window via `git log --oneline --since=2026-09-27 origin/main`.
- `feat/fix/refactor` split counted from the commit subject prefix; spot-verified 12 high-impact commits via `git show --stat` before quoting them.
- Area bucketing aligned to the section structure of `Hermes_Architecture.md` so the next re-compile has a 1:1 merge map.

## Commit-type concentration (weekly)

| Type | Count | Share |
|---|---|---|
| `fix` | 1,725 | 52.9% |
| `test` | 450 | 13.8% |
| `chore` | 249 | 7.6% |
| `feat` | 231 | 7.1% |
| `refactor` | 186 | 5.7% |
| `docs` | 82 | 2.5% |
| `catalog` | 65 | 2.0% |
| `plugin` | 50 | 1.5% |
| `style`/`fmt` | 57 | 1.7% |
| **total** | **3,259** | 100% |

Note: the `fix`-heavy share is misleading on its own — the prior week also logged ~1,725 fixes. The 3,259 weekly total is real, but the high `fix` share is partially explained by the schema v31 backfill (4 new `messages.*` columns, dozens of small follow-up fixes) and the desktop-experimentation revert cluster (15+ reverts of feature-flagged desktop card catalog / approval-flow cards). See "Reverts" below.

## High-impact changes this week (by area)

> Format: `<short-sha> — <one-line summary>`. SHAs are the verified source of truth. The next re-compile uses these to `git show <sha>` for re-verification.

### Configuration & provider routing

- `400cd692613` — **feat(cli): local models in `hermes model`** — `hermes model` (and `hermes setup`) gain a "Local models" row whenever PM pins an engine for this machine; runs the same handler path as the desktop's Local Models pane. *Relevant to ss:* directly affects how ss surfaces the local oMLX entry.
- `bf87ae620ef` — **feat(models): add gpt-6.1-sol (OpenRouter + direct OpenAI API)** — 1.05M direct, 272K Codex fallback; 272K compaction auto-raise on Codex route. `$2/$10`, cached input `$0.10` (5%, not 6 Sol's 10%), cache write `$2.50`, 2x/1.5x above 272K; `-pro` aliased to base row. Reasoning: `none`/`minimal` return 400 on 6.1 Sol (unlike 6 Sol), so it uses Astra's `low..max` ladder without Astra's account gating. *Not on Nous Portal or Codex OAuth yet for our test account.*
- `5bb6127c5b1` — **feat(models): add the gpt-6.1-sol-900k Codex OAuth picker variant** — live probe on ChatGPT OAuth streaming `/responses` with `effort=low`: completed at 684,566 / 898,681 / 918,137 input tokens with all three needles recalled; rejected at ~931K+. Real ceiling 922,000. The live catalog still advertises 272K with an 872K ceiling, so the bump is opt-in. *Eligibility is by exact slug, not the gpt-6-sol family prefix.*
- `f21b9476ba2` — **feat(models): add anthropic/claude-sonnet-5.5 to OpenRouter and Nous Portal catalogs** — Anthropic release 2026-09-28; both backends verified live. *Possible ss heavy-review lane replacement candidate; verify pricing and reasoning behavior before re-routing.*
- `652ce40e5bb` — **feat(models): add stealth/space-bunny-alpha to the OpenRouter curated list** — $0 stealth SKU, 1M context, tool support, image/video input. *Cheap/bulk candidate for fallback chain if pricing and quality hold up — verify before adding.*
- `bdde674296e` — **feat(fast): /fast ultrafast for GPT-6 Astra, priced at the served tier** — `service_tier="ultrafast"`, 6x Standard; broadly available for GPT-6 Astra in API and Codex (Pro 500 / Enterprise). GPT-6.1 Sol Ultrafast "coming soon".
- `ab66fb8996c` — **feat(models): list gpt-6.1-sol on Nous Portal (now served by /v1/models)**.
- `5fa55d16785` — **feat(plugins): run third-party plugins in a per-profile plugin host (plugins.isolation: host)** — non-bundled plugins now run in a host process per profile with only that profile's env. Re-entrant; handlers may call back into Hermes core. *New risk surface for ss's plugin-aware workflows; check the implications for grok-coder and any plugin-bearing skill.*
- `f1c79255fea` — **feat(plugins): requires.gpu gates a plugin server on the host's GPU vendor** — `requires: {gpu: nvidia}` refuses install on hosts without NVIDIA. *Affects any MCP catalog entry that's GPU-gated.*
- `51aa249efd5` — **feat(relay): name the Relay plugin files in `hermes doctor` and the startup log** — Relay 0.9 discovers user/system `plugins.toml` outside HERMES_HOME; the new code surfaces which files apply. *Useful for ss when debugging plugin config drift.*

### Schema versions & state.db migrations

- **Schema v30 → v31** — `messages.message_uid` (uuid4 hex) added; minted once at first insert, copied by every SQL clone, preserved across re-inserts of the same dict, stripped from provider requests. `_absorbed_message_uids` records merged consecutive-user uids on the survivor row. *Affects any operator code that does raw SQL on `messages`.*
  - `751d8526e35` — feat: add `messages.message_uid` and backfill.
  - `27d0ad63999` — fix: keep the settled-trigger skip, register the columns in schema history, strip `message_uid` at the transport. The trigger's comment sat before its `DROP TRIGGER` statement so the skip never matched → every open issued the DROP and took the write lock. *A sibling's transaction could block behind it.*
  - `9e7b54693da` — chore: record the four message-identity columns as one schema-history event (the three intermediate orders from the PR's own commit sequence never shipped on their own).
  - `1fd412e8a08` / `be55eb47343` — test: drop SCHEMA_VERSION import / schema-version/column-presence change-detector (now redundant).
  - `aac4d5812b8` — chore: label the v31 schema-history event with a commit on this branch.
  - `f51d0a72338` — feat: persist the consecutive-user merge witness on the survivor row.
  - `76d0ac6caad` — feat: give tool calls and their results a per-occurrence uid.
- **SCHEMA_VERSION stays at 31** in the current run after the v31 events land (was 30 going into the week).
- `0423d5dfea9` — **fix(tui-gateway): mtime fallback for sessions digests over foreign state.db schemas** — the TUI shows session info even when the local schema is behind the on-disk schema.
- `6c143c238fe` — **feat(metrics): hermes.desktop.* contract, schema and gateway RPCs** — new metrics surface; mostly desktop-internal.
- `6e6ec4aa9ed9` — **fix(auxiliary): the inherited main key comes from the same source as the anchor endpoint** — aux key resolution bug.
- `e66999df56c8` — **fix(state): coerce imported session flags like the int columns** — session-import hardening.
- `c24c8c2731d` — **fix(desktop): merge folded replies by stored source identity**.
- `653c946c368` — **fix(tools): coerce numeric task_id and strip whitespace in kanban_show** — kanban ID type hardening.
- `3b6589108a4` — **fix(sessions): rebuild /save json timings after dropping undone rows**.
- `b42984652ca` / `c5ff25a50ac` — **fix(sessions): keep /undo and rewind rows out of /save json** + cap by `sessions.max_export_messages`.
- `cf00b855c4e` — **docs(sessions): scope the /save json backup claim to CLI and messaging**.
- State recovery cluster: `d7732ca75e4` (recognize every persisted session-id shape in salvage), `0788e759dd3` (recovery must recognize `cron_`/`bg_`/`room_` ids), `3c17b9fd661` (free a ghost's title at rename time), `fb4aac296b9` (exclude empty ghost sessions from title uniqueness), `1ef8e3bd032` (roll back a failed unique-title repair as a whole), `bcd9c5074dd` (number duplicate user titles chronologically), `c64617ee8dd` (rank unique-title repair by provenance, never drop a user title).

### Skills (format, disabled-gate, preload, provenance)

- `c13232e90c7` — **feat(skills): external mounts get an 'external' provenance tier (#108032)** — `tools/skill_usage:provenance()` gains the `external` tier; skills from `skills.external_dirs` no longer flatten into `agent` in the usage report, dashboard, desktop catalog/detail UI, or learning journey graph. A local copy of the same name keeps winning, mirroring `is_agent_created`'s external exclusion. *Important for ss if any of its skills are mounted via external_dirs.*
- `e4bab4f7317` — **docs(skills): regenerate skill pages for the braced role-tag placeholders**.
- `8e543411524` — **docs(skills): state one precedence for duplicate skill names** — operator-facing clarification.

### MCP

- `1aadd774e0d` — **feat(optional-mcps|plugin-catalog): add sigrex MCP entry and re-pin sigrex plugin to 1.0.4** — catalog-only.
- `e6ec4aa9ed9` — **fix(auxiliary): inherited main key from the same source as the anchor endpoint** — aux key surface fix (also referenced in schema).
- `f0cacd11e83` — **fix(worktree): retry the plain remove with --force for clean submodule trees** — worktree cleanup hardening.

### Sessions, state, compression

- Schema v31 + state recovery cluster (see Schema section).
- `9cd114b4996` — **feat(web): inline_images=false on GET /api/sessions/{id}/messages** — web export hardening.
- `4147d8137a6` — **docs(compression): say the assistant-anchor escape also needs tool calls** — clarifies an existing escape clause.
- `ee61caebc48` — **docs(compression): cite #84371 in the reasoning-heavy test helper docstring**.
- `5762cce4c92` — **docs(compression): drop stale split-turn clause from task-snapshot comment**.

### CLI

- `400cd692613` — local models in `hermes model` (see Configuration).
- `a3149156ecc` — **feat(cli): make the summary the default `hermes status`, --full for every section** — operator-visible UX change.
- `c82ad613ec6` — **feat(cli): add `hermes status --short`**.
- `a557bebb8e6` — **fix(cli): decode percent-encoded fs paths that arrive double-encoded** — Windows path handling.
- `4149b1cf57` / `3559962a62` / `8d5e3e4121` — **test(cli): batch clarify choices stay inside the viewport** + matching fix `4149b1cf57` — TUI viewport hardening.

### Cron

- `6f7cc7e74ce` — **feat(cron): allow Python scripts to use an external interpreter** — per-job `interpreter` field so a cron `script` / `monitor_script` can run under a user-managed venv. Nothing is installed, frozen, or restored automatically. *Useful for ss cron jobs that need packages not in the Hermes runtime.*
- `1fd412e8a08` — **test(sessions): drop SCHEMA_VERSION import the trigram test no longer reads** — cron test cleanup.
- FTS trigram exclusion cluster: two tests that pinned "one version behind" to `SCHEMA_VERSION - 1` now pin to **v29** (the v30 migration they exercise) since v31 took over.

### Gateway

- `344a23bce57` — **feat(messaging): edit allowed users one ID per box on Desktop and dashboard** — pairing UX.
- `4c4ad0d3871` — **feat(discord): setup checks the bot token with Discord and prints the invite link** — onboarding hardening.
- `c03cca2763b` — **refactor(telegram): parse max_concurrent_updates with coerce_port** — type hardening.
- `0b3c03d62ed` — **docs(telegram): document max_concurrent_updates and per-chat ordering**.
- `ae01fb8e2e6` — **docs(discord): include thread creation permission** — operator setup doc gap.
- `34dd3ea2376` — **docs(gateway): say rotation persists the child, not always rewrites it** — clarification.
- `7f10c8ef412` — **docs(gateway): say what a worker-side nested resolve actually returns** — clarification.
- `3a056db7b1b` — **docs(gateway): say rotation persists the child, not always rewrites it** — clarification.
- `21d59f18e78` — **docs(tui_gateway): name both scope-release helpers in the follow-up comment** — clarification.

### Kanban

- `8e00bfdf06d` — **feat(kanban): add in-process schedule tool** — cherry-picked from `339cc5589d8`. 17 lines added to `tools/kanban_tools.py` plus tests. *Worth a quick test against the ss kanban lane to confirm behavior matches what we currently rely on.*
- `653c946c368` — **fix(tools): coerce numeric task_id and strip whitespace in kanban_show** (see Schema).

### Memory / Honcho

- (No net-new Honcho `feat` this week. The 2026-09-13 `honcho peers map` work stands.)

### Plugins / catalog (large growth, mostly catalog-only)

- **271 plugin-catalog commits** in the weekly window. Most are catalog-only (new entry, pin bump, disclosure edit). High-impact new entries worth a glance:
  - `2d6d4d1d324` + `edd24765435` — parcel-deliveries (with disclosure at merge).
  - `071ddfc80c2` — model-drift-watch.
  - `92fef8a0756` — parlor.
  - `164c6b625a0` — Ando platform beta.
  - `15d61d79d4d` — mistral-vibe plugin.
  - `7c9600edf1d` + `8f1f2ee1ea9` — pushover-hermes-plugin (pinned).
  - `b9ee9a501b3` — hermes-pii plugin configuration.
  - `5f5a8d7b3b7` — pinned Pushary approval plugin.
  - `94530930ee2` / `d148ae10e45` / `feaf782b870` / `954cf6cd3fc` — hermes-dreaming 2.1.0 (reviewed test translations; pinned to a450cb1; no core patch, jobs bound to the plugin skill).
  - `91ec17d3af1` — openai-compatible image provider.
  - `c27c2bc989e` — hermes-cloud-file-manager plugin.
  - `ebc5283782a` + `cf664d5c8fa` — hermes-otel 1.18.2.
  - `4eadb60437a` / `7a33d8c2b91` / `7a0a9ec8847` / `0dc3bed5a36` — hermes-cron 1.2.0 / 1.1.2 / 1.1.1 (new).
  - `92727fd1989` — memanto.
  - `92727fd1989` — rtk-rewrite-docker.
  - `2d9b77d6799` / `e3a646f54a7` / `9a0e2b00c28` — t3-code-theme 0.5.0 (SDK-only, pinned).
  - `41ff7cee35c` — dexio memory provider.
  - `d9874d4c6f1` — vornik-companion.
  - `17d7c4d028d` / `8690e8667bb` — gh-ref (with card image).
  - `7c381f1cb71` — jp-estat.
  - `ae4e67f3ea5` — jp-egov-law.
  - `f3065afb51b` — linkup plugin.
  - `1aadd774e0d` — sigrex MCP entry (see MCP).
  - `bd9271df069` — prompt-studio 1.9.0.
  - `9585b17be71` — talaria-webui.
  - `470d5468ed9` — Usage Ledger 0.4.3 (Windows beta).
  - `2c239f19ef1` — birkin-mnemosyne memory provider.
  - `97ac7cae9c4` — hermes-beads v0.1.2 (sha → tag commit `975263183`, `bd_bin` security fix).
  - `9569a778402` — Vault View v0.4.6 (banner and pinned source).
  - `3c497020d43` — telegram_dashboard_probe 0.8.2 (disclosure for `limits_sources` and `errors.log`).
  - `77d74d99cf1` — hermes-beads sha → `187d82a18` (pre-release head pending v0.1.2), drop desktop-pane claim.

### Desktop (massive commit cluster — but mostly cosmetic + flag-gated)

- The desktop team landed ~120+ `feat(desktop)` commits in the weekly window. Most are card UI, theme polish, hover affordances, catalog browser iteration, picker improvements, and consent/telemetry UX. The high-impact architectural items:
  - `37780213821` — **feat(desktop): exact subscription speed tiers (Standard / Fast / Ultrafast)** — surfaces the model-pill pricing tiers 1:1 with `/fast`.
  - `3acd9492178` — **feat(desktop): keep-awake follows the turn — a "while working" mode alongside off/always**.
  - `c55a6e3d9fb` — **feat(desktop): identity-attribution traces for multi-connection bot rosters**.
  - `5fa55d16785` — **feat(plugins): per-profile plugin host** (see Configuration; desktop is the primary consumer).
  - `c82ad613ec6` — `hermes status --short` (see CLI).
  - `cc2306e3361` / `641af6dfc52` / `a413ffd96e3` — **feat(desktop): pin / shift-click / star models in the picker**.
  - `a9d14d0ac1a` / `67bc5676578` — **feat(desktop): surface cached-read rates / per-model pricing in the model catalog menu**.
  - `b27d9eb7c00` — **feat(desktop): review-pane diff scope selector (Uncommitted / Branch / Last turn)**.
  - `54e2a59bbca` — **feat(pm): ship llama.cpp v0.5.0 (b11146) with Linux x64 CUDA** (PM-side, surfaced in desktop).
  - `54e2a59bbca` + `54e2a59bbca` — **feat(pm): ship llama.cpp v0.5.0**.
  - `361963828d7` — **feat(pm): llama.cpp CUDA on Linux arm64; bundle libgomp for Linux engines**.
  - `45fc5ebbef2` / `aa96575ed11` — **feat(desktop): show canonical live compression count / fold it into the context meter**.
  - `7a06cb50d7a` — **feat(telemetry): shared-metrics opt-in offer on every surface**.
  - `2de175e8a75` — **feat(desktop): put model-picker pricing behind a setting, off by default**.
  - `0fce280ee9e` — **feat(desktop): consent-gated renderer-crash recorder in the main process**.
  - `d9fd8587bea` — **feat(desktop): report packaged self-update outcome via shared_metrics.update_run**.
  - `520f0e9ed5e` — **feat(desktop): first-run shared-metrics consent and Settings › Privacy toggles**.
  - `f6c15b09717` — **fix(desktop): tag project-tree session rows with their registry source**.
  - `6cb7a64e594` — **revert: Skills and Plugins card catalog browser, third landing (reverts #119381)** — see Reverts.
  - `9b6e759751b` — **fix(desktop): one Classic Hermes card after the reverted build, and a light Classic** — revert follow-up.
  - `cc761ddb80c` — **refactor(desktop): one resolver for the packaged app's resources dir**.
  - `1d9fd8587bea` — desktop layer for shared_metrics.update_run (re-reported above; the SHA pairing in this file is the verified one).

### Internationalization (i18n)

- `9bcbe7b5df9` — **feat(i18n): pluggable, layered language packs across core, Desktop and TUI (#126296)** — plugin packs and user overlay over bundled locales. nanostore runtime, RPC pack loader, `_keys.tui.json` emitter. *The desktop "i18n expansion" mentioned in CHANGELOG-2026-09-27 is now real code; French/German/Spanish bundles implied.*

### Image gen

- `bf1bc78e638` — **feat(image_gen): expose the Krea 2 sliders to the agent only when the active backend declares them** — cherry-picked from `cbbd7f2b7b`. *Affects the hermes-multimodal lane if it routes through Krea 2.*

### Release / PM / Install

- `13dbe260aa1` — **feat(release): --branding stable builds a channel commit as the regular app**.
- `361963828d7` — **feat(pm): llama.cpp CUDA on Linux arm64; bundle libgomp for Linux engines**.
- `54e2a59bbca` — **feat(pm): ship llama.cpp v0.5.0 (b11146) with Linux x64 CUDA**.
- `af90026aa0` / `a3fa13182c` / `47e6c2ebb5` / `c4d4ac569f` / `e0b843431b` / `4cf2ea805f` / `7d509de7e1` — **fix(pm)/update: libatomic pre-install only on Linux, don't write old-interpreter bytecode into the new tree, keep the completion parent stdlib-only, install before the detached update child, document the libatomic host package, prompt for sudo only when needed, let `hermes update` authorize sudo before restaging Node**.
- `e0b843431ba` — **docs(install): document the libatomic host package step** — operator-facing.
- `59dc1ad5ff1` — **fix(release): abandon force-cancels the attempt's in-progress runs**.

### Security (RCE class closed, OAuth callback XSS, secret-store write-deny)

> *CVE-class items in the 2026-09-20 changelog (GitSpawn RCE, OAuth callback XSS, HERMES_HOME secret-store write-deny, MCP User-Agent) still stand. No new CVE class this week, but the desktop-renderer consent-gated recorder (`0fce280ee9e`) and the relay doctor visibility (`51aa249efd5`) are security-adjacent.*

- `ddde0670104` — **fix(plugins): a forced reinstall of the same source keeps the user's files** — reinstall safety.
- `d2d45b549b9` — **fix(pm): a borrowed HERMES_HOME leaves the source checkout to the root that owns it** — ownership hardening.
- `a789f874e2e` — **fix(tools): coerce restored SSH session working directories** — SSH path safety.
- `97ac7cae9c4` — **catalog: hermes-beads v0.1.2 (`bd_bin` security fix)** — third-party plugin security bump.
- `ed2c9c2ec9b` — **ci: widen CODEOWNERS to manifests, installers, scripts/ci, SECURITY.md, plugin catalog** — governance.

### Reverts (15+ this week — concentrated in desktop experimentation)

- `05d8d0a2663` — **revert: stale worktree reaping revalidation (#132260)** — reverts 9 commits. Restores `hermes_cli/worktree_gc.py`, `hermes_cli/worktree_ops.py`, and `tests/security/test_gitspawn_config_injection.py`. *Worktree GC was overshooting; this is the 3rd-time-lucky pattern the team has shown on the Gitspawn work.*
- `6cb7a64e594` — **revert: Skills and Plugins card catalog browser, third landing (reverts #119381)** — third try at the card catalog browser. *Pattern: feature added → regression → reverted. Cumulative revert pattern flagged in CHANGELOG-2026-09-20 too.*
- `afc47d6e4ff` / `8edfe92501f` / `2252d4303f5` / `a743ca54c3d` / `eb049af69c8` / `eb481d0cd9e` / `20e25850a2c` / `2bde65f825d` / `734a94400a3` / `423eef696e8` / `87bb559e8e6` — **revert cluster: approval/clarify/timeout card wait behavior** — 11 reverts related to the same "approval prompts wait until answered / hold open / outlive cap" thread. *Don't rely on the approval-panel-forever behavior in production — the cluster shows it's still in flux.*

### Salvaged PRs (cherry-picks preserving authorship)

- `8e00bfdf06d` — kanban in-process schedule tool (cherry-pick from `339cc5589d8`).
- `bf1bc78e638` — Krea 2 sliders (cherry-pick from `cbbd7f2b7b`).
- `27d0ad63999` — schema v31 trigger fix (cherry-pick from `86c87a3cac`).
- The pattern of "feature lands on a feature branch, then cherry-pick onto main with squashed metadata" is up vs. the prior week. *Note for the AUTHOR_MAP curation when the next re-compile lands.*

## Net-new docs (operator-visible)

- `e0b843431ba` — install: libatomic host package.
- `639eee30586` — plugin-catalog: README and docs pages describe the same delisting, removal and entry fields.
- `390c49ee786` + `4b4362520be` + `c776b6691de` + `2811f7eaaaf` + `978fe337bfa` — **email cluster**: `authserv_id` required for authenticated-sender mode; absent pin fails closed; topmost-only exact match; per-message receiver hosts must unset `authserv_id`; `OPENAI_BASE_URL`-bound `OPENAI_API_KEY` goes only to that origin. *Email subsystem saw a stdlib rewrite; these are the operator-facing doco follow-ups.*
- `fa190d2187c` — desktop: note the local-secondary exception on the owner stamp helper.
- `ae01fb8e2e6` — discord: thread creation permission.
- `406cf7787b9` — state: trim derived-id recognizer comments to the contract.
- `5762cce4c92` — compression: drop stale split-turn clause from task-snapshot comment.
- `8e543411524` — skills: state one precedence for duplicate skill names.
- `17a7a9cb6b7` + `33eedf29c94` — plugins: pin `plugins.isolation` in the managed scope; document the host-isolation mode invariants.
- `0b3c03d62ed` — telegram: max_concurrent_updates + per-chat ordering.
- `3d0a61ac94e` + `b90c5c71cb8` — goal: show the `-- <text>` escape in `/goal` args hint; document `/goal -- <text>` escape for control-word goals.
- `cf00b855c4e` — sessions: scope `/save json` backup claim to CLI and messaging.
- `2b52acc2d8c` + `94aedfb84b4` — profiles: name the OS credential stores in the distributions credential bullet; list OS credential stores among export exclusions.
- `73da8e6b1eb` — state: restore the `display_kind` line and say which key the carry matches.
- `3a056db7b1b` — gateway: rotation persists the child, not always rewrites it.
- `34dd3ea2376` — state: condense `hermes_state_pidns` module docstring.
- `ee61caebc48` — compression: cite #84371 in the reasoning-heavy test helper docstring.
- `4147d8137a6` — compression: assistant-anchor escape also needs tool calls.
- `7f10c8ef412` — gateway: what a worker-side nested resolve actually returns.
- `21d59f18e78` — tui_gateway: name both scope-release helpers in the follow-up comment.
- `618a4d5dbd7` — sessions: a restored export keeps its pinned, hidden and archive flags.
- `814c7de76eb` — sessions: filtered export covers pinned/archived in `--help`.
- `978fe337bfa` — env: `OPENAI_BASE_URL`-bound `OPENAI_API_KEY` goes only to that origin.
- `5fe12f373ea` — docs: drop source channels and install identity section; main is the only source channel. *Caveat: this is about the public docs site, not the local install — operators on multiple branches should not assume channel coverage.*
- `de2c9c2ec9b` — ci: widen CODEOWNERS to manifests, installers, scripts/ci, SECURITY.md, plugin catalog. *Operator-visible governance change.*

## Action items for the next full reference re-compile

1. **Schema v31** — `messages.message_uid` and the four message-identity columns. The next re-compile must list the columns, the trigger ordering, the backfill behavior, and the transport-strip contract.
2. **Plugin host isolation mode** (`plugins.isolation: host`) — describe the per-profile host process, the ABC checks, the RemotePluginContext object, and the re-entrancy model. This is the largest plugin-system change since 1Password/BWS.
3. **`requires.gpu`** in plugin manifests — capture the install-refusal path, the catalog card behavior, and the "needs an NVIDIA GPU" disclosure.
4. **gpt-6.1-sol / -pro / -900k** — full catalog entry with reasoning behavior, 6x-Standard ultrafast tier, and the 272K vs 1.05M context distinction. Capture the 900K opt-in slug exactly.
5. **claude-sonnet-5.5** — capture reasoning ladder, pricing, and any rate-limit or `/fast`-tier behavior.
6. **Local models in `hermes model`** — describe the unified path (PM → desktop Local Models pane → CLI `hermes model`).
7. **i18n pluggable layered packs** — describe the layered catalog (bundled → plugin packs → user overlay), the nanostore runtime, and the RPC pack loader. Capture the desktop TUI en catalog emission.
8. **External skill provenance tier** — `external` joins `agent` and `bundled` in the provenance surfaces (usage report, dashboard, desktop catalog/detail UI, learning journey graph).
9. **Cron `interpreter` field** — per-job Python venv escape; document the no-install/no-restore contract.
10. **The approval-panel-forever revert cluster** — capture the 11-revert pattern; mark this feature as "in flux" in the architecture reference so operators don't build workflows that depend on it.
11. **Skills + Plugins card catalog browser (revert #119381)** — third landing reverted; capture the cumulative revert pattern.
12. **The `bd_bin` security fix in hermes-beads v0.1.2** — third-party plugin security update, surfaced in the operator-facing security doco.
13. **State recovery session-id shapes** — `cron_` / `bg_` / `room_` ids and the unique-title provenance ranking; capture the `state.py` change-detector removal.
14. **TUI viewport batch-clarify** — capture the new TUI behavior and the matched fix/test cluster.
15. **PM / libatomic** — capture the new pre-install spawn behavior and the libatomic host package step in the install doco.
16. **Update ARCHITECTURE.md `repo_commit` line and `last_drift_review`** — the frontmatter is at `version: 1.15.0`, `last_drift_review: 2026-09-27`, `repo_commit: 4f22543509` (with upstream-now marker). The next re-compile lands at `version: 1.16.0` (semver minor bump per the protocol), `last_drift_review: YYYY-MM-DD-of-recompile`, `repo_commit: 8d5e3e4121` (or whatever upstream HEAD is at the time).

## Known limitations of this pass

- The local Hermes install (`v0.21.5+5376.gb12e672.dirty`, 2026-09-24) is **1,604 commits behind** `origin/main`. The drift count that matters for this pass is the SSOT anchor delta (22,139), not the local-install delta (1,604). I did not run `hermes update` per the operator's "lightweight local model only" directive — operator still runs that on a separate cadence.
- I did not re-fetch `hermes-agent.nousresearch.com/docs/` page-by-page; I cross-checked the new `llms.txt` index against the SSOT anchor's coverage. The index shows 100+ doc pages; the surface is healthy. No new top-level doc category landed this week.
- The desktop commit cluster (120+ `feat(desktop)`) was bucketed but not spot-verified line-by-line. The next re-compile should `git show --stat` each one before quoting.
- The plugin-catalog growth (271 commits) was enumerated at the entry-name level, not the disclosure-or-payload level. Catalog-only items have low operator impact and can be summarized in one bullet next pass.
- Two reverts in the prior week (`Revert "fix(tts): install kittentts on python 3.14 via the misaki fork"` per CHANGELOG-2026-09-27) are not re-listed here because they pre-date the weekly window. If they re-emerge, add to "Reverts" in the next pass.
- AUTHOR_MAP curation: 3 cherry-picks this week. Confirm the original authors' attribution survived the rebase before promoting to the next re-compile's "contributors" section.

## Verification checklist

- [x] `git -C ~/.hermes/hermes-agent fetch origin main` succeeded.
- [x] `git rev-parse origin/main` matches `8d5e3e4121`.
- [x] `git rev-list --count 4f22543509..origin/main` returns 22,139.
- [x] `git log --since=2026-09-27` returns 3,259 commits.
- [x] 12 high-impact SHAs spot-verified via `git show --stat`.
- [x] Reverts enumerated and PR-numbered.
- [x] Skill frontmatter bump applied (see next file edit).
- [x] Public mirror updated (see commit log in this session).

## Cross-checks against official sources

- `https://hermes-agent.nousresearch.com/docs/llms.txt` is healthy and indexes ~100+ doc pages. No new top-level doc category.
- `https://github.com/NousResearch/hermes-agent` HEAD matches the local `origin/main` from this run.
- The local install's reported `upstream 1298c8e7` is older than `origin/main` (8d5e3e4121) because the local install was generated by an older `hermes update`. The weekly window is the authoritative delta.
- No doc page on the live site contradicts the items in this changelog; the i18n + plugin host + gpt-6.1-sol pages exist but were not deep-read this pass.
