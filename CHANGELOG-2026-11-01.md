# Hermes Architecture Drift Review — 2026-11-01

> Historical snapshot, not current truth. See [README.md](README.md) and [Hermes_Architecture.md](Hermes_Architecture.md) for current state.

## Anchors (canonical)

| Fact | Value |
|---|---|
| Local Hermes commit | `bf0203a794` (v0.22.5, 2026.9.24) |
| Upstream HEAD | `c7c7e0fa92` (2026-11-01 09:05 -0800) |
| Local-behind-upstream count | 3,226 commits |
| Prior SSOT anchor | `4f22543509` (2026-08-30, upstream v0.20.6) |
| **Total delta since prior SSOT anchor** | **34,616 commits** (~63 days) |
| Prior review's upstream anchor | `d050956633` (2026-10-25) |
| **This window** | **2,035 commits** (7 days, `d050956633..c7c7e0fa92`) |
| **SSOT body last updated** | **2026-06-21** — the body itself is **133 days stale** (48,592 commits behind upstream) |
| Review window type | **Changelog-as-scaffolding (lightweight local model directive)** |

> **Date-naming correction.** The prior two changelog files were misnamed relative to their content. The file originally named `CHANGELOG-2026-11-24.md` was written on **2026-10-25** (mtime) and its upstream anchor `d050956633` is dated **2026-10-25**; the file originally named `CHANGELOG-2026-11-10.md` was written **2026-10-18** with anchor `22063fc099` (2026-10-18). Both files were named for a date ~1 month *after* the window they describe. This file is named for its true date (2026-11-01). **[Update 2026-11-06: those two files have since been renamed to [CHANGELOG-2026-10-25.md](CHANGELOG-2026-10-25.md) and [CHANGELOG-2026-10-18.md](CHANGELOG-2026-10-18.md) and their internal dates corrected. The references to the old names below are kept as written at the time, with the current names added in brackets.]**

## Corrections to prior passes (verified this run)

| Prior claim | Source | Verified truth |
|---|---|---|
| "No schema bump (v31 unchanged)" | SKILL.md pointer for `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] | **FALSE.** `SCHEMA_VERSION` is **32** upstream (`hermes_state_common.py:289`). The 31→32 bump landed in `1eda335ae0` (2026-10-17, "catch a lagging state.db up once on browse instead of 503"). That commit is an ancestor of `d050956633` — it was inside the *prior* window and was missed. `FTS_STORAGE_VERSION` is **3**. |
| "Reference is 60 days stale" | `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] | Understated. The SSOT **body** (`Hermes_Architecture.md`) was last updated 2026-06-21 and has never been re-compiled since — it is **133 days / 48,592 commits** stale. The 60-day figure measured from the `4f22543509` anchor (2026-08-30), not from the body's actual last sync. |
| "Reference is 27 weeks stale" | `CHANGELOG-2026-11-10.md` [now `CHANGELOG-2026-10-18.md`] | Understated for the same reason. From the body's real last sync (2026-06-21) it was 119 days ≈ 17 weeks. |
| "Prior SSOT anchor `4f22543509` (2026-09-25, v0.21.2)" | `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] | Anchor commit date is **2026-08-30**; its `__version__` is **0.20.6** (`git describe` → `v2026.8.27`). Not 2026-09-25, not v0.21.2. |
| Type breakdown "fix 13,605 (41.8%) / refactor 4,668 / test 2,934 / feat 1,763 / chore 1,592 / docs 707 / perf 196" | `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] | Not reproducible. Actual over `4f22543509..d050956633`: fix **14,849**, refactor **4,757**, test **3,698**, chore **2,430**, feat **1,918**, docs **885**, catalog **400**, ci **249**, perf **197**. |
| "Total delta since prior SSOT: 32,581 commits (~60 days elapsed)" | `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] | Commit count **32,581 is correct** for `4f22543509..d050956633`. The "60 days" is measured from the anchor commit date; the elapsed time to the *body's* last sync is 56 days. |

**Root cause of the recurring miss:** the drift protocol anchors on the `repo_commit` frontmatter value (a *review* anchor), but the SSOT **body** has a separate, older last-sync date (2026-06-21). Measuring "staleness" from the review anchor understates it by ~2 months. The runbook's §1 anchor probes should read the body's own header date. See "Action items" item 1.

## Commit type breakdown (`d050956633..c7c7e0fa92`, 2,035 commits)

- `fix`: 1,163 (57.2%)
- `test`: 209 (10.3%)
- `chore`: 196 (9.6%)
- `feat`: 111 (5.4%)
- `refactor`: 94 (4.6%)
- `docs`: 51 (2.5%)
- `catalog`: 47 (2.3%)
- `perf`: 23 (1.1%)
- `ci`: 9
- merges: 21 | residual/other subjects: 109

**Fix-velocity ratio 10.5:1 over features** — consistent with the prior window. No schema-version bump landed in *this* window (v32 predates `d050956633`), but the prior window's miss above is corrected here.

## Dominant themes (architecture-area bucketing)

### Configuration & provider routing — remote config plane (largest new subsystem)

The single largest architectural addition in this window. Config is no longer necessarily a local file.

- **Remote config backend** (`69d037b349`) — `plugins/config_backends/remote/` (10 files) serves each profile's user layer from a config plane instead of `config.yaml`. Boot fetch inside `load_hermes_dotenv` with bounded retry and **fail-closed** exit; per-process **ETag poller** (D9); key-level diff writes with **CAS** + one conflict retry (D10); client-side lock and secret-literal checks (D7, D17); **in-memory migrations** (D12); managed `config.yaml` ignored (D18); plane credential from `auth.json` or IdP client-credentials only, **never** from a `secrets:` source (D26, D32).
- **ConfigBackend seam** — every user `config.yaml` read/write is routed through a `ConfigBackend` (`3f903abeed`, `afb8c9a1a7`), with backend-consulted tests. `hermes config edit` exits 1 when the backend has no config file.
- **Docs page now exists**: `https://hermes-agent.nousresearch.com/docs/user-guide/remote-config` — confirms `HERMES_CONFIG_BACKEND=remote`, `HERMES_CONFIG_INSTANCE_ID`, `HERMES_CONFIG_REMOTE_URL` (default `https://config-config.nousresearch.com`), and the locked-key / secret-literal rules.
- **LEFT_CORE migration** (`hermes_cli/left_core_migration.py`, 1,194 lines) — a feature that leaves core keeps its names/keys and its home is migrated onto the catalog plugin automatically. Backs the Copilot removal below.

**Action items for re-compile:** new section "Configuration backend seam" documenting `ConfigBackend`, the `HERMES_CONFIG_BACKEND` switch, the remote-plane env vars, key locking, CAS writes, fail-closed boot, and the LEFT_CORE migration contract. Fold into the existing "Configuration — config.yaml SSOT" section and the "Secrets" section (remote-mode secret rules).

### Providers / Auth — GitHub Copilot removed from core (breaking)

- **Copilot removed from core** (`1da2eb42dc`, `refactor!`) — `plugins/model-providers/copilot/` and `copilot-acp/` are deleted. Both ship as **one** plugin, `github-copilot`, in `NousResearch/hermes-labs-plugins`. Core keeps no Copilot code; `agent/copilot_acp_client.py` remains as the deprecated alias that `kimchi-acp` subclasses (`agent.acp_client.ACPClient` is the public base). Upstream docs confirm: `hermes plugins install github-copilot`; profiles already routing through `copilot`/`copilot-acp` get it auto-installed; `hermes auth add copilot` for device-code login; `gh`-CLI/Copilot-CLI logins are **no longer read**.
- **Left-core migration rows** (`889d753725`) + 18 `fix(left-core)` commits — notices kept in the home until an agent in some process delivers them; BOM-tolerant reads; a failed plugin load surfaces its import error, never "enable it".

**Action items for re-compile:** the SSOT "Supported Providers" table still lists `copilot` as a core provider (line ~2090). That row must change to a labs-plugin note with the install command and the auth change. Also add a "Labs plugins / left-core migration" subsection to the Plugins section.

### CLI — new commands and a real session-model control

- **`hermes cron cancel`** (`8c228c5b89`) — stops a job's **live run**; the schedule is untouched. Wired through `hermes_cli/subcommands/cron.py` (`cron_cancel` subparser) and the cronjob tool `action=cancel`. Removing a job while it runs now cancels that run instead of burning tokens until a lost fire claim discards the result (`ee394bd921`). A stopped/removed run books a terminal **`cancelled`** ledger row with its reason; `last_status` becomes `cancelled` (`3f43514791` fixes the CLI printing "failed").
- **`hermes sessions recover`** — offline, non-destructive rebuild of canonical session data into a separate clean database (`hermes_cli/session_recovery.py`, `hermes_cli/subcommands/sessions.py`). Refused for stores missing `sessions`/`messages`; never recreates an empty store.
- **`hermes webapp`** — opens the browser-hosted Desktop the gateway serves (`hermes_cli/webapp_launcher.py`).
- **`hermes hooks`** — inspect/approve/remove shell-script hooks declared in `config.yaml`.
- **`hermes config get` / `hermes config unset`** — present upstream (`hermes_cli/config.py:_CONFIG_SUBCOMMANDS`); the SSOT documents only the `/config get` slash form.
- **`/model reset`** (`5bb7bdc7c4`, `66c48cb158`) — clears a session's model pin **without losing the conversation**. Rejected when combined with any flag (`MODEL_SWITCH_ERR_RESET_WITH_FLAGS`); not aliased to "default" (MoA preset). Clearing a pin also drops an armed `--once` restore. Documented in upstream's slash-commands reference.

**Action items for re-compile:** add `cron cancel`, `sessions recover`, `webapp`, `hooks`, `config get`/`unset` to the CLI section; add `/model reset` to the slash-command table.

### Cron — per-job timezone, grace window, missed-run policy

- **Per-job timezone** (`bfa67878cd`) — a job may pin its own IANA timezone; its cron expression is then that zone's wall clock, and every slot move honours the pinned zone (`3dbe3ec4d8`).
- **Per-job `grace_minutes`** (`b81e71addd`) — optional per-job override of the catch-up window; `grace_minutes=0` skips real misses instead of every run (`3035c0c0ec`).
- **Missed-run policy** per job (`ee394bd921`), set from the Desktop editor (`0e0b2328e0`).
- **Actual run time in the prompt** (`3166c4a4b1`) — the system prompt carried only a date-granularity timestamp; scheduled jobs now receive their real run time.
- Cancel semantics hardened across the ledger: a second cancel keeps the first reason (`9e5ff5098b`), a stopped run stays stopped in every ledger reader (`249a90c037`), a Stop still lands when `HERMES_HOME` has no hard links (`cd377d884d`), and the cancel marker is read BOM-tolerantly (`422fce2f52`).

### Hooks — `pre_llm_call` can block a turn (security-relevant)

`ad9229c843` — a `pre_llm_call` callback (Python plugin or shell hook) may now return `{"action": "block", "message": "..."}` to **veto the turn before any inference request**. No model call is made, the message becomes the user's reply, the transcript keeps a normal user→assistant pair, and the result carries `turn_exit_reason=blocked_by_plugin_pre_llm_call`. The vetoed prompt stays on the host (no session-titling call). This is the supported seam for DLP/privacy/policy plugins that previously had to patch core (#57364).

**Action items for re-compile:** the SSOT's hooks content predates every hook hook point here. Add `pre_llm_call` with its inject-vs-block return shapes, the `turn_exit_reason` value, and the timeout contract (`plugins.hook_callback_timeout`, default 30s; bounded hooks fail closed for `pre_tool_call`, fail open otherwise).

### Approvals & security — invisible characters in approval prompts

`3f5612cde6` — approval prompts now show hidden characters and padding: zero-width and bidi-override characters, the U+E0000 tag block, raw ANSI/OSC escapes, a bare carriage return, and long runs of spaces that push the real tail off screen or past the 200-char chat preview. Without this, a human approves something other than what runs.

### Compression / context — prepared compaction

- `compression.prepare_ahead` (opt-in) — one prepared summary per compaction; newer messages join the tail; extension passes and the stale-tail check are gone (`c2383a6806`, `d2d1824fdf`).
- Every prepared-compaction pass and splice outcome is recorded (`4501b3f80b`); adoption/outcome counts are shared-metric counters (`4ea24baeb1`).
- One revert in this cluster: "a compaction's pruned tool stub no longer paints beside its archived original" (`4b4b679416`).

### Sessions & state — refresh-prompts

`e240d617e8` — a continuing session replays its stored system prompt verbatim to keep the provider prefix cache warm, so SOUL.md edits, new/edited skills, and project-context changes never reached a long conversation until `/new` (which drops history). `refresh-prompts` lets a continuing session adopt updated prompt sources without losing the conversation.

### Desktop / Webapp / plugin SDK

- Webapp: flash the tab + browser notifications for attention events (`18d8b4a65d`); load desktop plugins in a browser tab and serve the host's desktop plugins to the Webapp's loader (`abfe2876f3`); install a desktop plugin on the host that serves the tab; copy agent+desktop packages' desktop halves on the serving host; one UI preferences store per machine shared by Desktop and its Webapp.
- Desktop: every profile as a sidebar section (`47f0fc99d2`); error card v2 (severity, one action, DS buttons, details copy block); flash the taskbar/dock entry on a native notification while unfocused; a multi-question clarify is one form on Desktop and the Webapp; boot-failure notice wears the Hermes wordmark and theme colours.
- Docker image ships the Webapp renderer (`25a6a44c4a`); Nix ships the Webapp in the package; the session token rides the unit environment as a path (`7c8d7b9cf3`, `c90b27e8e3`); the deployment session token authorizes the gated dashboard (`2edfd0d369`).

### Gateway — intentional silence per room

`33cf20374d` — a room can be configured to receive no reply (new `gateway/response_filters.py` + `gateway/run_config_loaders.py` paths). Relevant to any gateway tuning work: silence is now a first-class per-room setting, not an absence of output.

### Providers — credential guidance seams

`d74fe78713` — `ProviderProfile.missing_credentials_hint()` appended by runtime resolution and both readiness checks; a plugin can now name its own sign-in flow. Public model-setup helpers and `secret_source_label` for provider plugins (`public` scope). `gpt-6.1-terra` is listed on OpenRouter and Nous Portal (1.05M context) (`a382af23f4`).

### TTS — agent-pickable ElevenLabs voices

`f219708926`, `c975831fbc` — voice search folded into the `text_to_speech` schema (`find_voices`); the agent can pick a `voice_id`, with paginated `tts_voices` search.

### Platform adapters

- Telegram: video note messages (`016bc1df65`).
- Slack: emoji reactions in thread context (`3a73078f3a`).
- WhatsApp: delivery and read status of sent messages logged to `gateway.log` (`31d5f1d44d`).
- Photon: setup success gated on iMessage opt-in via first message (`5d9b27cd23`).
- Hosted rooms: Discussion budgets configurable (`afe3a9111f`, `41010b6fef`).

### Streaming / sessions / skills / misc

- Opt-in cap on a streaming response's total lifetime (`ac8037c05b`).
- A `skills` lane for unbroker spam-call runs (phone-led runs + DNC one-shot) (`3e1a5b1368`), with a 10-commit `fix(unbroker)` cluster covering the DNC consent gate, defunct-broker exclusion, and the captcha-gated opt-out flow.
- Plugin catalog: 54 new `feat(plugin-catalog)` entries in this window alone (60 including `feat(catalog)`).

### Reverts (5)

| SHA | Subject |
|---|---|
| `43ee1b0872` | revert(plugins): Desktop, Webapp and dashboard installs take every source the CLI takes |
| `0c06e8e6a6` | Revert the render-time turn reply fold (maintainer ruling, PR #144899) |
| `4b4b679416` | Revert "fix(state): a compaction's pruned tool stub no longer paints beside its archived original" |
| `e5ea5fd9cc` | Revert "fix(gateway): the worker reservation always binds the owner's OS subject" |
| `db2054f48e` | Revert "fix(webapp): send a Content-Security-Policy with the Webapp page" |

## Action items for next full reference re-compile

Ordered by blast radius. The SSOT body is 133 days stale, so these accumulate.

1. **Fix the staleness measurement first** (runbook + SKILL.md). Add a body-provenance probe to the runbook: read the `Compiled:`/`Updated:` line from `Hermes_Architecture.md` and report body age separately from anchor age. Two prior passes under-reported staleness by ~2 months because they anchored on the review commit instead of the body's own last-sync date.
2. **New section: "Configuration backend seam"** — `ConfigBackend` abstraction, `HERMES_CONFIG_BACKEND=remote` + `HERMES_CONFIG_INSTANCE_ID` + `HERMES_CONFIG_REMOTE_URL`, boot fetch fail-closed, 5-minute ETag poll, CAS key-level writes with one conflict retry, locked keys, secret-literal rejection, in-memory migrations, managed scope ignored in remote mode, and the refused-commands list (`config edit`, profile clone/rename/distribution, backup/import/snapshot restore). Reference the upstream docs page.
3. **Provider table fix** — remove `copilot` from "Supported Providers" in core; add a "GitHub Copilot (labs plugin)" note (`hermes plugins install github-copilot`, auto-install for existing homes, `hermes auth add copilot`, and that `gh`-CLI/Copilot-CLI logins are no longer read).
4. **New subsection: "Labs plugins / left-core migration"** — `LEFT_CORE` table, the migrate-once-per-home rule, the failure backoff (skip an hour, doubling to a week), unattended dependency consent, and kept migration notices.
5. **CLI table additions** — `hermes cron cancel`, `hermes sessions recover`, `hermes webapp`, `hermes hooks`, `hermes config get`/`unset`.
6. **Slash-command table** — add `/model reset` with its reject-with-flags rule and the no-history-loss guarantee.
7. **Hooks** — add `pre_llm_call` with inject vs block return shapes, `turn_exit_reason=blocked_by_plugin_pre_llm_call`, and the timeout/fail-open-vs-fail-closed contract.
8. **Approvals** — add the hidden-character/padding rendering rule (the 200-char chat-preview interaction).
9. **Cron** — per-job IANA timezone, per-job `grace_minutes`, per-job missed-run policy, `cancelled` as a terminal ledger status, real run time in the prompt.
10. **Compression** — `compression.prepare_ahead`, one prepared summary per compaction, pass/splice outcome recording, shared-metric adoption counters.
11. **Sessions** — `refresh-prompts` and the prompt-cache-stability trade-off it addresses.
12. **Gateway** — per-room intentional silence; the new `response_filters` / `run_config_loaders` config surface.
13. **Schema section** — state `SCHEMA_VERSION = 32` and `FTS_STORAGE_VERSION = 3`; document the 31→32 bump (`1eda335ae0`) and the read-only drift-catch rule (a lagging store is *drift*, not corruption; a store missing `sessions`/`messages` is refused, never recreated).
14. **Provider routing** — `ProviderProfile.missing_credentials_hint()`, public model-setup helpers, `secret_source_label`.
15. **TTS** — ElevenLabs `find_voices` in the `text_to_speech` schema, agent-pickable `voice_id`, paginated `tts_voices`.
16. **Platform adapters** — Telegram video notes, Slack thread emoji reactions, WhatsApp delivery/read status logging, Photon iMessage opt-in gating, hosted-room Discussion budgets.
17. **Models** — `gpt-6.1-terra` (1.05M context) on OpenRouter and Nous Portal.

## Known limitations of this pass

1. **No SSOT re-compile** — per the operator directive ("run with lightweight local model only") and the skill's decision tree, this is a changelog-as-scaffolding pass. `Hermes_Architecture.md` was **not** modified. Its body is 133 days / 48,592 commits stale and the re-compile remains the top outstanding item.
2. **Prior-changelog corrections were made here, not by editing the prior files** — `CHANGELOG-2026-11-24.md` [now `CHANGELOG-2026-10-25.md`] and `CHANGELOG-2026-11-10.md` [now `CHANGELOG-2026-10-18.md`] still carry the false "v31 unchanged" and understated-staleness claims. The protocol forbids overwriting a prior changelog (the chain is the audit trail), so the corrections are recorded in this file's "Corrections to prior passes" table and in the SKILL.md pointer for the affected entries.
3. **Docs coverage** — the docs landing page, `llms.txt`, the configuration reference, the Remote Config page, the Event Hooks page, and the CLI/slash-command references were read. The full `llms-full.txt` (~1.8 MB) was not ingested; per-page reads were targeted instead.
4. **High-velocity fix cluster** — 1,163 fix commits in 7 days. Listing them individually is impractical; bucketing was used.
5. **Local Hermes is behind upstream** — local is v0.22.5 (`bf0203a794`, 2026-10-22), 3,226 commits behind `c7c7e0fa92`. All claims are about upstream, not the local install.
6. **Unbroker cluster** — 10 `fix(unbroker)` commits touch an opt-out/consent flow with live-browser findings. Read the commits before relying on that lane's behaviour.

## Spot-verification sample

25 commits were verified with `git show --stat` / `git show -s` (body inspected against the message). All matched:

- `1da2eb42dc` (Copilot removed from core) — Verified. Provider plugins deleted; ships as one labs plugin; revertible on its own.
- `69d037b349` (remote config backend) — Verified. `plugins/config_backends/remote/`, fail-closed boot, ETag poller, CAS writes, plane-credential rules.
- `ad9229c843` (`pre_llm_call` block) — Verified. Block shape, `turn_exit_reason`, withheld-prompt behaviour.
- `8c228c5b89` (`hermes cron cancel`) — Verified. `_CRON_SUBCOMMANDS["cancel"]` → `cron_cancel`; schedule untouched.
- `5bb7bdc7c4` (`/model reset`) — Verified. `is_reset`, flag rejection, `--once` restore cleared.
- `3f5612cde6` (approval hidden characters) — Verified. Zero-width, bidi, tag block, ANSI/OSC, CR, padding.
- `c2383a6806` (one prepared summary per compaction) — Verified. Extension passes and stale-tail check removed.
- `e240d617e8` (`refresh-prompts`) — Verified. Continuing sessions adopt updated prompt sources.
- `33cf20374d` (intentional silence per room) — Verified. `gateway/response_filters.py`, `run_config_loaders.py`.
- `d74fe78713` (missing-credential seams) — Verified. `ProviderProfile.missing_credentials_hint()`.
- `ee394bd921` (cancelled ledger row) — Verified. Terminal `cancelled` status with reason; removal cancels the live run.
- `bfa67878cd` (per-job timezone) — Verified. Pinned IANA zone drives schedule compute and due-scan.
- `b81e71addd` (per-job grace_minutes) — Verified. Optional per-job catch-up override.
- `3166c4a4b1` (actual run time in prompt) — Verified. Date-granularity timestamp replaced.
- `25a6a44c4a` (Docker ships Webapp renderer) — Verified. `.dockerignore` + `Dockerfile` + a bundle test.
- `a382af23f4` (gpt-6.1-terra) — Verified. OpenRouter + Nous Portal, 1.05M context, catalog manifest regenerated.
- `2edfd0d369` (session token authorizes dashboard) — Verified. Token-only deployment path.
- `016bc1df65` (Telegram video notes) — Verified. Adapter + 126-line test.
- `3a73078f3a` (Slack thread emoji reactions) — Verified.
- `31d5f1d44d` (WhatsApp delivery/read status logging) — Verified.
- `afe3a9111f` (hosted-room budgets) — Verified.
- `f219708926` / `c975831fbc` (TTS voice search) — Verified.
- `47f0fc99d2` (Desktop profile sidebar sections) — Verified.
- `18d8b4a65d` (Webapp attention notifications) — Verified.
- `0c06e8e6a6`, `e5ea5fd9cc`, `db2054f48e`, `4b4b679416`, `43ee1b0872` (the 5 reverts) — Verified. All revert a named prior commit; subjects match the diffs.

Anchor arithmetic re-verified independently this run: `4f22543509..d050956633` = 32,581; `d050956633..c7c7e0fa92` = 2,035; `4f22543509..c7c7e0fa92` = 34,616; `65a477f12e..c7c7e0fa92` (body sync → HEAD) = 48,592.

---

**Prepared by:** cron drift-review pass
**Date:** 2026-11-01
**Status:** Changelog-as-scaffolding; SSOT body untouched per operator directive
**Next action:** Mirror to the public mirror repo; then schedule the full re-compile
