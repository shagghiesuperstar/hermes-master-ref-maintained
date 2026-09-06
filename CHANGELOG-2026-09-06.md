# Hermes Architecture Reference — Drift Changelog 2026-09-06

## Scope of this review

- **Local Hermes version:** v0.21.0 (2026.8.31) — local `3ca096de5f` (on `main`, **5,224 commits behind** `origin/main` at review time; `hermes --version` also reported upstream tip `089bb328` before fetch)
- **Upstream HEAD reviewed:** `c5594ec4b3` (origin/main as of 2026-09-06)
- **Prior SSOT anchor (last drift review upstream HEAD):** `4f22543509` (CHANGELOG-2026-08-30)
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **5,800** (~87 feat, ~974 fix, ~3,387 refactor, ~55 simplify/compat, rest test/chore/docs/perf/style/fmt; **1** explicit whitespace revert)
- **Window:** 2026-08-30 → 2026-09-06 (~1 week)
- **Releases in window:** **`v0.21.0` (2026.8.31 / tag `v2026.8.31`)**
- **Full SSOT drift (`cdcbc3a31` → origin/main):** **17,618** commits pending
- **Methodology:** lightweight-local model (per operator/cron directive) + git-log area-bucketing + spot-verification (`git show --stat` + body) on high-impact commits. Official docs spot-check (CLI commands page + configuration page, HTTP 200). **Changelog-as-scaffolding** per the drift review decision tree (5,800 commits ≫ 500; operator said "lightweight local model only"). The main reference (`Hermes_Architecture.md`) is left untouched.

> **Why a changelog and not a full re-compile.** Delta is far above the 500-commit re-compile threshold, the full SSOT is now ~11 weeks / ~17.6k commits stale, and the cron directive is lightweight-local only. This pass continues the changelog chain (07-05 → 07-12 → 07-19 → 07-26 → 08-09 → 08-16 → 08-23 → 08-30 → **09-06**). **A full re-compile remains critically overdue** and should be authorized on the next non-lightweight pass. Note: ~58% of this window is mechanical `refactor(...)` / `simplify(compat)` from the Sep 2026 decomposition — architecture-facing behavior still changed materially (schema 26→30, local runtime, tool deferral, plugin compat window, bot-mode durability).

## High-impact architecture deltas (`4f22543509` → `c5594ec4b3`)

### Schema versions & state.db migrations

- **SCHEMA_VERSION 26 → 30** on tip (`hermes_state_common.py`). Progression verified by walking `hermes_state_common.py` history in-window:
  - **v27** (`238b6c1ab9`) — `sessions.compression_recovery_deadline REAL` so anti-thrash recovery survives gateway agent rebuilds / cache eviction (wall-clock epoch, not process-local monotonic).
  - **v28** (`8e4366d358`) — `sessions.tool_names` JSON column freezes availability-gated `tools[]` across agent-cache eviction; `/reload-mcp` (and compaction / `/new`) is the conscious re-probe hatch.
  - **v29** (`ea65fcd980`) — exclude **cron** sessions from trigram FTS (`messages_fts_trigram_src` view + triggers).
  - **v30** (`2b55ded1ac`) — also exclude **subagent/delegate-child** transcripts from trigram FTS (shared `FTS_TRIGRAM_EXCLUDED_SOURCES` / `fts_trigram_session_sql`); standard word FTS still indexes children. Fan-out-heavy DBs drop dramatically (spot claim: 2k×2KB children 22.4→12.5 MB).
- **FTS_STORAGE_VERSION 1 → 2** (`5a7bee0fa8`) — structured `tool_calls` JSON excluded from trigram projection while remaining in standard FTS; independent of SCHEMA_VERSION.
- **SessionDB further modularized** (`d15c61b5dc` and follow-ons): domain mixins / free-function modules (`hermes_state_{common,schema,search,messages,gateway,maintenance,repair,dbfile,compression,registry,portability}.py`). Facades then stripped in the simplify/compat wave — **external plugins must not import old paths after 2026-09-14** (see Plugins).
- **state.db operational hardening:** process-wide shared writer registry; single-flight opens; refuse open/writes on deleted-WAL generation; quarantine after structural corruption / 0-byte truncated DB; fail-loud when state.db replaced under a live process; admission lock fail-closed; bound FTS indexing for large tool results; auto-prune **on by default** (90d) + freelist-ratio gate before VACUUM (`73f68362b3`, #54189).
- **Non-SessionDB schemas still present:** observer/middleware telemetry strings; relay `hermes.relay.runtime.v1`; dashboard lockfile schema v2; local_runtime catalog schema v1; verification evidence schema v1.

### Configuration & provider routing

- **Managed local models / llama.cpp runtime** (`43e67d872f`) — first-class `hermes_cli/local_runtime/` (catalog, hardware probe, GGUF fit/quant, engine install, resumable download, supervisor, router mode + SSE load progress). Desktop Settings → Providers → Local models (GUI behind `hermes desktop --local`); CLI/backend always live.
- **OpenRouter per-model `provider_routing.models.<id>` overlays** (`12871bd01e`) — same only/ignore/order/sort/require_parameters/data_collection keys as flat `provider_routing`; one chokepoint `_provider_preferences_for_agent` for CLI/gateway/TUI/Desktop/cron/`/model`/fallback/delegates. Docs note: **provider_routing is OpenRouter-only**; Nous Portal rejects the provider object (`0a195aa463`, `ba7d2a8633`).
- **Nous anthropic wire default flip** (`54e24ae1fa`) — `nous.anthropic_wire` defaults anthropic/* to **chat/completions** (not native `/v1/messages`) after measured 14–20% stuck prompt-cache pairs on the native portal route under concurrent tool loops.
- **External-process providers out-of-tree** (`1bd0b8ed41`, `1131b22856`) — `auth_type == "external_process"` + profile-supplied `process_command` / args / env; copilot-acp becomes one profile, not a hard-coded name branch.
- **Model catalog churn:** GPT-6 Astra + Astra Pro with fast/flex tiers (`f159e581c7`); Claude Fable 5.1 (`9f069a1175`); Gemini 3.7/3.8 Flash (`b0adce1cbf`, `0ee98eda52`); Qwen3.8-Max-0902 (`d20a8e4475`); Meta Muse Spark 1.3 (`7e60d0c042`, `7f2aa70add`); picker catalogs refresh every 20m with gateway warm path (`c2954c8934`).
- **Fast mode bounded windows** (`c7e2e0b779`) — `/fast auto|cold` + `agent.service_tier` / `agent.fast_auto_seconds` (default 60s); route-aware gate so OpenRouter/Nous/Copilot/Azure/Bedrock/custom never get inappropriate service_tier/speed kwargs; prompt cache untouched.
- **Config keys:** `context_file_read_timeout` (default 5s, `cfe88a1f7d`); `skills.create_dir` (`42c2838674`); `display.bell_on_prompt` collapsing clarify/approval bells (`552159d222`); `display.resume_last_session` (`7aff724e56` / `648c664eb3`); `gateway.trust_env` for aiohttp proxy-env honoring (`45b0d8cab5`); `terminal.docker_snap_compat` opt-out (`256bd1adc9`); `cron.delivery.notify` (`00a7115a02`).
- **Docs-confirmed (live site 2026-09-06):** CLI page still documents `hermes peer`, `hermes project`, `hermes proxy`, `hermes kanban`, `config get`; configuration page shows `context_file_read_timeout`, `provider_routing`, `tail_mode`, telemetry. Not yet surfaced on those two pages at crawl time: `hermes pause`/`resume`, `hermes cron doctor`, `hermes worktree`, `skills.create_dir`, `failure_deliver`, `cron.delivery.notify`, `tool_search` core-deferral story, local-runtime one-click (may live on other doc pages / release notes).

### Compression / context

- Anti-thrash recovery deadline durable across gateway rebuilds (schema v27, above).
- Tools[] freeze across cache eviction (schema v28, above).
- **Subdirectory AGENTS.md ceiling 8k → 32k** with head+tail truncation + WARNING (`d61cff60e3`); root AGENTS.md split into root + per-area files ≤~8k target (`4441a2a28d`) after the 100k monolith broke the old ceiling.
- Oversized file-ref fallback + simplify findings fold (`cfa1a90b8c`, `07b161e241`).
- Compression checkpoint deferral / native-compaction eval harness work continues (`cd71ee0708`, `8d4b7f874d`).

### Skills

- **`skills.create_dir`** routes agent-created skills to a configured directory; folded into `get_all_skills_dirs()` (`42c2838674`).
- Bundled skills: reddit-reading + rss-feeds + multi-platform sweep guidance (`4dc9988f78`).
- Instruction paths render the configured create dir wherever the path is named (`f709bd88b6`).
- Desktop: built-in optional-skills catalog with one-click install (`ada7213cca`).

### MCP

- **MCP SDK 2 input schema cache write-through fix** (`0c62b297c6`) — `mcp_field(t, "input_schema", "inputSchema")` so lazy servers no longer advertise empty parameter schemas after pydantic renamed the model field.
- MCP tools merge when alias collides with a static toolset name (`3bc1780f2d`, `245e48008f`).
- Perf: skip npx resident parent when package cached; one parent-death supervisor per process; apply `-t` spawn filter before MCP SDK import (`11ed840431`, `2d783a15eb`, `87597d30c8`).
- CLI: filter MCP server spawning by `-t/--toolsets` + orphan subprocess leak fix (`e73257ccc1`).
- Large mechanical split of `tools/mcp_tool` into siblings + facade drop (simplify/compat) — treat old import paths as temporary only.

### CLI

- Local runtime / managed llama.cpp (above).
- `hermes plugins compat [--json] [path]` + doctor/update notices for import-path breakage (`0a5164cebe`).
- OSC 9 + Konsole OSC 777 notifications ride bell flags (`632078bca7`).
- Fast mode `/fast auto|cold` surfaces (above).
- Serve path dispatch without full parser tree (`5180601a6a`).

### Cron

- **`hermes cron doctor`** + overdue `next_run_at` silent-non-firing flag (`b028fe632e`, `f2f7a3bf15`).
- **Per-job `failure_deliver`** — route or suppress failure notices; shares deliver grammar; `local` = structural silence (`c9491e6a7d`, NS-788).
- **`cron.delivery.notify`** configurable; UNVERIFIED live deliveries surface in list/doctor (`00a7115a02`).
- Multiplex: credentialless satellite cron delivery through primary adapter for exact `profile_routes` targets (`a6351a71e5`, #101113); docs: delivery for routed profiles rides shared bot only for exact routed targets (`eed481bd34`).
- Trigram FTS cron exclusion (schema v29).
- Cron sessions excluded from trigram; preflight reads primary config via `read_user_config_raw()` integration fix.

### Gateway

- **Remote MEDIA delivery** — `MEDIA:` files inside ssh/modal/daytona/singularity/vercel sandboxes fetched via `BaseEnvironment.fetch_file` + `gateway/media_fetch.py` with host denylist parity (`70a64db532`, #466).
- **Media denylist expands** to state.db / kanban.db / WAL-SHM / legacy sessions / browser-profile cookie store / named-board kanban DBs (`4139695c97`, #41071) — whole-tree HERMES_HOME deny still not reintroduced.
- `gateway.trust_env` single key for aiohttp proxy-env honoring (`45b0d8cab5`).
- Inbound media cache writes offloaded (`cb0b66c161`).

### Platform adapters

- **Slack:** interactive Block Kit `/model` picker (`1e69c12b64`).
- **Email:** configurable IMAP/SMTP transport security (tls/starttls/plain) + TLS verify toggle (`92a9864517`).
- **Photon / iMessage:** read-receipt toggle + receipt-type alias (`d63f996a75`, `9744fc0c99`).
- **Buzz:** edit_message/delete_message for streaming replies; thread-topology cluster (reply_in_thread opt-out, NIP-10 root anchoring) (`34c10f83c3`, `972f0314de`).
- **Webhook:** accept standard webhook signatures (`cb3b5bb64c`); outbound payloads stamp emitting profile (`98c27aa25c`).
- Large platform plugin refactors (Telegram/WhatsApp/WeCom/Teams/etc.) are mostly LOC compaction with payload-parity claims — treat behavior as stable unless a fix commit says otherwise.

### Bot Mode / peer / hosted rooms

- Durable Group Chat authority + replay (`cbc67b939f`); authority lineage on replay pages + byte-bounded replay (`cc4b5ba1fc`); survive authority gateway death via log replication + fenced takeover (`e730deedd1`); same-gateway Group Chats without Desktop (`93c7089f70`); scoped cross-gateway Group Chat transport (`e7433910e9`).
- Perf: cold DM hops skip live `/models` probe; group chat rooms answer in time of one bot not sum (`32fe129324`, `fb5023950e`).
- Canonical Bot Chat protected from auto-title; provenance-blind guard documented (`fb9a294794`, `89e2e4f572`).

### Delegation / MoA

- **Per-task completion groups** — ungrouped subagents return as they finish; optional `group` keeps merge-sets together; capacity still one pool slot per original call (`028fe2c4c8`). Mid-unit crash keeps finished children (`c5594ec4b3` tip fix).
- Batch numbering per conversation not per process (`0cb996d977`); subagents never inherit 1h prompt-cache tier (`0edb6b928a`).
- Perf: finished delegate children no longer pin transcripts in parent heap; shared httpx transport pool; shared child timer scheduler (`c96568f66c`, `c3b411dfb7`, `561b053f79`).
- MoA: review-fix restored `_relay_moa_reference_event` / quiet-reference test (`3a8e3a2e88`); heavy moa_loop/model_switch compaction in simplify wave.

### Approvals & security

- **`approvals.deny` applies inside isolated containers too** (`1c37b9c457`, #91002) — deny list is never-bypassable intent; evaluated before container skip; built-in dangerous heuristics still skip in sandbox.
- Corrupt-token / docker_env warnings stop logging credential material (`aa0beef684`, #102308).
- Desktop: deny window-open side-effect opens (GHSA-9f4c-93c8-jc8g) (`77ca6a6d12`).
- Approval-mode zap on desktop status bar by default (`2bb9c4e693`).

### Computer-use / browser / web

- Lightpanda: on-disk HTTP cache; honor `browser.engine=lightpanda` in Browser Use mode (`6973ce3ac2`, `e3a85ae5a0`).
- **Perplexity Search API** as keyed `web_search` + `web_extract` backend (`f1ccf436a2`) — not keyless-ring.
- **Tavily** web search/extract provider (`428e084dcd`).
- Fast discovery file ordering for search (`413fd2a1fc`).
- File-ops native POSIX read fast path + collapsed shell probes (`d3275acf80`, `6a2b3f1ebf`, `9fbb3b716a`, `8cab422ab0`).

### Plugins / Agent Plugins

- **Sep 2026 decomposition + temporary plugin import-path shims** (`2776813df3`) — single revert-on-schedule commit: 332 facade PLUGIN-COMPAT blocks, 1,172 lazy PEP 562 names, restored deleted publics, 3 stub modules. **In-tree must not depend on shims** (`scripts/check_compat_pointers.py` in CI).
- **Compat window enforcement** (`0a5164cebe` et al.): `COMPAT_REMOVAL_DATE = 2026-09-14`; `hermes plugins compat`, doctor section, update notice, Desktop one-shot dialog; after date PluginManager skips hitting external plugins unless `plugins.allow_deprecated_imports: true`.
- Opt-in **shared-metrics / telemetry exporter** returns (`180291162f`, #95278) with consent windows, send-state columns, install_id HMAC identity, tools toggle — prior NeMo relay shared-metrics revert from 08-09 is superseded by this opt-in design (verify consent defaults before enabling in ops).
- Image gen: Meta Muse Image provider plugin (`d7e92ab7e3`).

### Desktop / TUI

- Session automation / structured session controls; foreign coding-agent transcript import view; drag-to-create sessions; bot roster user sections + polish; comment mode in in-app browser (selector/markup/styles); Russian locale; cache-hit rate + tokens/sec status bar (off by default); Message Bubble transparency; first-open consent for real-profile browsing (`dffd8d62c2`, `9186e3ebc5`, `6b1e12c7f4`, `3d0ac691af`, `10f2a20966`, `a922dad9d8`, `3e63367175`, `ada7213cca`, …).
- TUI gateway: extensive method compaction / hosted-room folds (mechanical).

### Tool search / core tools

- **Core-tool deferral** (`e16ad33a9d`) — curated ~19-tool set behind the bridge by default; renames `todo_list` / `cronjob_manage` / `process_manage` / `gui_tour` / `show_tip` with legacy aliases; desktop schemas ~13.4K → 6.9K (−49%).
- Tips/tours withdraw when user switches them off (`b293157f7d`).

### Worktree / update / recovery

- Pushed open-PR worktree lanes reclaim disk; cron tick prunes worktrees (`3a351a9665`).
- Post-update state.db guard covers every profile not just root (`d0f0afb009`).
- Recovery banners name real state.db paths; stop pointing `.recover` at live DB; lazy tables in one schema map (`914d8a0bd6`, `5d9a2110ba`, `5dfd1e77a8`).

### Providers / Auth (additional)

- Kimi Code fallbacks use Anthropic Messages wire (`4ee2c78324`, #77256).
- GLM-5.3 on Nous/OpenRouter no longer 400s when thinking disabled (`f6bd1633f7`).
- OpenCode: `x-opencode-session` on every request for backend affinity (`139396995a`).
- Meta-AI live-first model catalog + generic contributor warning (`83ecb6e695`).
- Qwen OAuth login reaches `_mark_qwen_oauth_active` in moved owner (`38742342b2`).
- DeepSeek thinking replay contract for third-party Anthropic proxies (`0543fa2f1b`).
- Codex: masked `invalid_prompt: Request blocked.` replay rejection reaches replay-strip recovery (`e0592b6d43`).

### Docker / Photon

- `terminal.docker_snap_compat` AppArmor opt-out (`256bd1adc9`, #9730).
- Photon read receipts (above).

### Secrets

- Secret-material log scrubbing on corrupt-token/docker_env warnings (above).
- Mechanical secret-persistence unifications in web routers / setup paths (simplify wave) — no new SecretSource plugin claimed in feat inventory this window.

### Reverts

- `696662997b` — Revert whitespace-only hosted_rooms SQL constant reflow (parity noise).
- `2031c819fe` — simplify(compat) re-land of a gateway commit previously reverted by a stale-index commit (not a product feature revert).
- Plugin compat shims (`2776813df3`) are **scheduled for revert on 2026-09-14** — operational deadline for external plugins.

### Salvaged PRs (selected)

- #91002 / #91029 approvals.deny in containers (@fangliquanflq)
- #466 / #68506 remote MEDIA fetch (@tokou design)
- #41071 media denylist for SQLite stores (@pprism13)
- #95278 shared-metrics exporter
- #13996 skills.create_dir (@giwaov)
- #100745 desktop bot roster polish
- #91451 / #102129 MCP SDK 2 schema cache
- #24495 / #100711 OpenRouter per-model provider_routing design
- #100185 compression recovery deadline (minimal salvage)
- OpenRouter provider_routing schema samples from community PRs (co-authored)

## Concentration map (commit subjects)

Top scopes by count (illustrative of where the week went):

| Count | Scope |
|---|---|
| 607 | refactor(hermes_cli) |
| 308 | refactor(tools) |
| 261 | refactor(gateway) |
| 152 | fix(desktop) |
| 131 | refactor(agent) |
| 104 | fix(gateway) |
| 89 | refactor(state) / refactor(gateway/platforms) |
| 77 | refactor(computer_use) |
| 71/70 | refactor(tui_gateway) / refactor(tui) |
| 55 | simplify(compat) |
| 51 | refactor(mcp) |
| ~19 | feat(desktop) |
| ~7 | feat(models) |
| ~6 | feat(telemetry) |
| ~5 | feat(bot-mode) |

## Action items for next full reference re-compile

1. **Rewrite SessionDB / schema section** for SCHEMA_VERSION **30**, FTS_STORAGE_VERSION **2**, mixin module map, trigram exclusion sources (`cron`,`subagent`), tool_names + compression_recovery_deadline columns, auto-prune defaults + freelist VACUUM gate, shared writer registry / WAL-generation guards.
2. **Add Local Runtime architecture page** — `hermes_cli/local_runtime/*`, desktop `--local`, catalog schema v1, supervisor/router SSE.
3. **Document plugin compat timeline** — decomposition, COMPAT_REMOVAL_DATE 2026-09-14, `hermes plugins compat`, `plugins.allow_deprecated_imports`, facade retirement; remove any "import from hermes_state / run_agent barrel" guidance.
4. **Tool surface:** core-tool deferral defaults, renamed tool aliases, tool_search bridge token economics; `/reload-mcp` re-probes availability.
5. **Provider routing:** OpenRouter-only `provider_routing` + per-model overlays; Nous `anthropic_wire` default chat/completions; external_process provider profile contract; GPT-6 Astra tier pin behavior.
6. **Fast mode:** static vs auto/cold windows; route-aware service_tier gate; config keys.
7. **Cron:** doctor, failure_deliver, delivery.notify, UNVERIFIED ack surfacing, multiplex SharedRouteAdapters.
8. **Gateway media:** remote sandbox fetch path + expanded SQLite denylist.
9. **Bot Mode:** durable group authority/replay/takeover; canonical Bot Chat guards.
10. **Delegation:** completion groups; subagent FTS/heap policy; no 1h cache tier inherit.
11. **Config key catalog refresh** for all keys listed above; drop stale claims (schema 26, MoA council if still absent, cross-profile write guard retired earlier, etc.).
12. **AGENTS.md / subdirectory hints:** 32k ceiling, per-area files, head+tail truncate semantics.
13. **Telemetry:** opt-in shared-metrics exporter consent model (supersedes older "reverted NeMo relay metrics" one-liner if left unqualified).
14. **Consume entire changelog chain 2026-07-05 … 2026-09-06** as merge map; do not partial-edit Hermes_Architecture.md.

## Known limitations of this pass

- Official docs crawl was two pages (CLI reference + configuration) via static extract — not a full docs site inventory; some v0.21 features may be documented elsewhere.
- Local install remains **5,224 commits behind** tip; runtime behavior on this machine is v0.21.0 at `3ca096de5f`, not tip `c5594ec4b3`. Changelog describes **upstream tip**, not necessarily the running binary.
- ~3.4k refactor commits were not individually spot-checked; behavior claims prefer `feat`/`fix`/`perf` with `git show` bodies.
- Schema intermediate commits briefly show non-monotonic version numbers in intermediate trees during simplify forward-ports; **tip is SCHEMA_VERSION 30 / FTS_STORAGE_VERSION 2**.
- Public mirror sync is a distribution step after local skill patch.

## Decision

- **Changelog-as-scaffolding:** YES (`references/CHANGELOG-2026-09-06.md`)
- **Full `Hermes_Architecture.md` re-compile:** NO (threshold + lightweight directive)
- **Skill frontmatter bump:** version `1.11.0` → `1.12.0`; `last_drift_review: 2026-09-06`; `repo_commit` points at prior full SSOT anchor with new upstream tip + pending count
