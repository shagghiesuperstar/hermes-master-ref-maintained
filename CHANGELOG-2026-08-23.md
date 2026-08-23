# Hermes Architecture Reference — Drift Changelog 2026-08-23

## Scope of this review

- **Local Hermes version:** v0.20.5 (2026.8.19) — local `fcbd1076a` (on `main`, **310 commits behind** `origin/main`)
- **Upstream HEAD reviewed:** `f293e7206` (origin/main as of 2026-08-23)
- **Prior SSOT anchor (last drift review upstream HEAD):** `7095e23eb` (CHANGELOG-2026-08-16)
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **1,540** (~244 feat, ~739 fix, rest chore/test/refactor/docs/ci/style; **11** reverts, mostly CI cache-bust noise + 2 agent reverts)
- **Window:** 2026-08-16 → 2026-08-23 (~1 week)
- **Releases in window:** `v0.20.2` (2026.8.16) → `v0.20.3` (2026.8.16.2) → `v0.20.4` (2026.8.18) → **`v0.20.5` (2026.8.19)**
- **Full SSOT drift (`cdcbc3a31` → origin/main):** **10,169** commits pending
- **Methodology:** lightweight-local model (per operator/cron directive) + git-log area-bucketing + spot-verification (`git show --stat` + body) on ~30 high-impact commits. Official docs spot-check (home, CLI commands, configuration). **Changelog-as-scaffolding** per the drift review decision tree (1,540 commits ≫ 500; operator said "lightweight local model only"). The main reference (`Hermes_Architecture.md`) is left untouched.

> **Why a changelog and not a full re-compile.** Delta is above the 500-commit re-compile threshold, the full SSOT is now ~9 weeks / ~10.2k commits stale, and the cron directive is lightweight-local only. This pass continues the changelog chain (07-05 → 07-12 → 07-19 → 07-26 → 08-09 → 08-16 → **08-23**). **A full re-compile remains critically overdue** and should be authorized on the next non-lightweight pass.

## High-impact architecture deltas (`7095e23eb` → `f293e7206`)

### Configuration & provider routing

- **`agent.max_turns` default → unlimited (`null`).** `c32119b12` + foundation `504628286`:
  - `resolve_turn_limit()` is the single normalization point: ints, numeric strings, and unlimited spellings (`none` / `unlimited` / `infinite` / `∞` / `-1` / `0` / `inf` / `infinity` / YAML `null`) → `sys.maxsize` sentinel or int.
  - Default flipped from numeric cap to unlimited across CLI / agent_init / subagents; silent mid-task truncation was worse than the cap.
  - Docs + DEFAULT_CONFIG updated. **Note:** prior changelog still said 90→500; current truth is **unlimited by default**.
- **OpenCode Free provider + keyless-auth policy.** `28a9b6c56` + `2a2307e68` + `930be347f` + `ca06b8768`:
  - New `plugins/model-providers/opencode-free/` — free models via models.dev filter; keyed `OPENCODE_FREE_API_KEY` or keyless with opencode User-Agent.
  - Keyless providers count as authenticated everywhere (`auth.py`, `model_switch.list_authenticated_providers`, desktop inventory explicit-only filter) so zero-setup free tiers appear in `/model` and desktop pickers.
- **Ox Alpha / free stealth models.** `c01cd26f9` (OpenRouter `stealth/ox-alpha`, 1M ctx) + OpenCode Zen twin `x-preview-f-free` + picker alias `1bf8bd2c7` ("ox alpha") + free-model star/`-100%` discount column `bd93a5f31`.
- **Codex GPT window policy reversed to opt-in 900K.** `63a9c26fb`:
  - Base Codex slugs (gpt-5.6-sol/terra/luna, gpt-5.4) back to **272K** default (cost control).
  - Picker synthesizes explicit `<slug>-900k` variants; suffix stripped before wire.
  - Companion compression fix: `-900k` keeps global 50% threshold; 85% autoraise stays on 272K base (`933c209e9`).
- **GLM-5.3 replaces GLM-5.1** in OpenRouter/Nous catalogs (`907da145b`) + Z.AI 1M context / reasoning_effort (`01d8562fc`).
- **Bedrock OpenAI Responses / GPT-5.6 Sol-Terra-Luna Mantle routing.** `e5b96fcb1` / `16476fad1`.
- **Custom OpenCode-family provider hardening:** case-insensitive matching, `opencode-go-*`, per-model `api_mode` + `/v1` healing (`f9cd51eb6`, `a83c3915a`, `49780391a`).
- **Sibling-profile config migration on `hermes update`.** `706f33d42` — non-interactive safe migration sweeps every profile home via ContextVar `HERMES_HOME` override so fleet no longer drifts `_config_version` silently.
- **Docs-confirmed (live site 2026-08-23):** `hermes peer`, `hermes project`, `hermes proxy`, `hermes egress`, `hermes lsp`, `hermes security audit`, `hermes approvals`, `hermes dump`, `hermes prompt-size`; Bot Mode first-class; `agent.max_turns` unlimited spellings; keyless web backends; Cursor-style `${env:VAR}` SecretRef still valid.

### Schema versions & state.db migrations

- **SCHEMA_VERSION remains 26** (`hermes_state_common.py` on origin/main). No new schema bump in this window.
- **state.db durability / FTS self-heal cluster:** unbounded repair loop stop (`27d661e17`), macOS write barriers on every repair connection (`ca28a69ad`), defer FTS rebuild under foreign WAL holders (`fc72d6c71`), gateway FTS rebuild guards (`c45e2b19c`), generic corruption match restore (`987064caa`), handoff leg data-loss + surface corruption to users (`59f302fef`).
- Git-metadata generation (v26 column) remains the latest structural change from prior week.

### Compression / context

- **Identical tool re-calls → reference stubs.** `761990b78` — duplicate payloads no longer re-enter context; stubs point at prior result (tool_guardrails + tool_result_storage).
- **Would-grow / salvage compression cluster:** salvage grown candidates before refusal (`fb96247ea`), count would-grow as ineffective strike (`62016a1b0`), `/compress` refusal no longer reports success (`7bf66ec39`), restore prune runway when refusal keeps transcript (`4c76ec81a`), retire stale vision images in protected tail (`7ff2fe8bc`), shared image-strip policy (`b7544dba0`).
- **Mid-turn uncompressed overflow guardrail** when compression disabled (`4d1fc6ca0` / `db5d5dffe`, #89297).
- **Prompt-cache:** `apply_anthropic_cache_control` idempotent on pre-decorated input (`0fc52b055`).
- **Native compaction:** preserve compression summary messages during pre-checkpoint pruning (`fb27614ad`); hide compaction scaffolding/carriers from clients (`97e32d49a`, `a2a23a8f7`).
- **Daybreak Codex** 900K metadata + auto-raise threshold (`f8e5949f6`, `ac8dff4fb`).

### MCP

- **Migrate to mcp 2.x SDK** (revision 2026-07-28). `11a9dcf56`:
  - FastMCP → `MCPServer`; snake_case field access via `mcp_field()` dual-spelling helper; `sdk_httpx()` resolves httpx flavour; OAuth callback returns `AuthorizationCodeResult` + RFC 9207 `iss`.
- **Speak 2026-07-28 stateless protocol.** `382060f02`:
  - Per-server `protocol:` `auto` (default, handshake-first) | `stateless` | `legacy`.
  - SEP-2549 list caching TTL (`ttlMs`/`cacheScope`) bound to lazy schema cache.
  - SEP-837 OAuth `application_type=native`; SEP-2577 Sampling marked upstream-deprecated (kept functional).
- **MCP tool results spill at 50K** (tighter than generic 100K) + upstream-elision warnings + hard 2M-char allocation cap. `09e657793`. Config: `tool_budget.mcp_result_size_chars`.
- **Per-server `oauth.user_agent`** for token-endpoint requests only. `a6bada232` (#75576).
- **MCP CIMD auth** `0b588cb3a`; elicitation schema field-name fix `23a86594c`; Protocol-Version seeded from handshake `77ed1bbf4`; strip invisible Unicode TAG chars from MCP content (Goose port) `8bbda8ff3`; surface tool-result `_meta` minus reserved keys (Kimi port) `c031fec36`.
- Docs: Hermes Cloud Portal MCP management guide.

### Cron

- **Memory enabled for cron agents** (parity with kanban/delegate/gateway). `ef04d846e` — `skip_memory=False`; memory toolset no longer hard-denied. Follow-up: still do **not** load MEMORY.md into scheduled jobs (`fc9cbc872`).
- **Per-job `reasoning_effort` override** in job definitions + `hermes cron create/edit --reasoning-effort` + tool create/update. `4e1dd1a74`. Model-facing cronjob tool schema keeps it off (`991af03f4`); CLI lane restores it (`43c6dace5`).
- **`deliver='bot-chat[:<profile>]'`** pseudo-platform — cron output lands as inbound turn in a bot's canonical Bot Chat. `a2da0ab79`. Timeout: `cron.bot_chat_delivery_timeout_seconds` (default 600s).
- Continuable `in_channel` seed / mirror opt-in carve-outs + Slack workspace `scope_id` carry + scale-to-zero idle counts cron/API work (`85b89451f`, `3c52d3589`, `743dc935f`, many follow-ups).

### Gateway / Bot Mode / Relay

- **Gateway-owned control socket.** `60b626914` (#92091 step 1):
  - Unix domain socket `$HERMES_HOME/gateway.sock` (pointer-file fallback; Windows named pipe).
  - Verbs: `identify` (pid/profile/home/code_sha/supervisor/start_time), `status` (live runtime payload).
  - Fleet consumers (`collect_fleet_versions`, `hermes update --plan`) prefer socket over PID scans. 0600 ACL; non-fatal bind failure.
- **Bot Mode agent-to-agent:**
  - `message_agent` tool — Bot-Chat-only, session-gated, not in global tool registry (`e26d91dc1`).
  - `hermes peer` CLI — cross-machine bot DMs over peer api_server (`6229683b6`); peers in `bot_peers` + `HERMES_PEER_<NAME>_KEY`.
  - Cross-connection desktop bots can message each other (`d3e087fd8`).
  - Reclaimed bot chat self-resumes (`d5281f598`); DM tempfile isolation + orphan reap; group-room rename awareness.
- **`GATEWAY_RELAY_ALLOW_DIRECT_PLATFORMS`** opt-out for relay-exclusive mode (`2d3f5c155`).
- **Loop-liveness watchdog** knobs + 3-strike default + reconnect-stall tolerance (`361614572`, `aa08cb8cc`).
- Multiplex `/p/<profile>/` on non-multiplex gateway **fails closed** (`6cb1085d3`, `265bdcac8`).
- Handoff recovery idempotent; claim ledger / resume_pending cleared inline before abandonable boot-send (`83b09ebd0`, `684e95a00`).
- ContextVar isolation for kanban dispatcher + supervised watchers (`f12cd0401`, `bf3a0bb99`).

### Update / fleet reliability (#91277)

- **Structured update receipts** + post-update fleet version matrix. `1d74833d8` — `~/.hermes/logs/update_receipts/` + `latest.json`; desktop/dashboard read receipt instead of inferring success (`8804e7835`).
- **`hermes update --plan`** read-only fleet inventory (`0aecadc17`); plan becomes restart worklist — every planned runtime must be accounted for (`18b7fc82b`).
- **In-place branch update** for unmerged commits + `--switch-branch` opt-out + `updates.parked_branch_strategy` (`91096bb2f`, `4fad27a10`, `70151dd54`).
- Sibling config migration (see Configuration).
- Gateway killed-and-not-replaced fails fleet check; launchd supervision verify; no ZIP-fallback on dep failures/dirty trees; stop gateway we cannot relaunch (`168487786`, `1bf93660f`, `dfcef7006`, `eac3f645e`).
- Dashboard refuses model picker with clear 503 when code is stale after update (`f293e7206`, #86207) — tip of reviewed HEAD.

### Process identity / worktrees / CLI

- **Positive process identity.** `95fa81426` — `HERMES_SPAWN` tags, machine spawn ledger (`spawn-ledger.json` keyed pid+create_time), Windows job-object self-attach; update reaper uses ledger before heuristics.
- **`hermes worktree list/prune`** attended reclaim + `/worktree prune` + startup WARNING when `.worktrees/` >10 trees or 5GB. `f309f92d3`. Content-gated branch GC (not name-gated); untracked-only scratch archived under `~/.hermes/archive/worktree-prune/`.
- Inspired-by Copilot: **`/worktree`** create isolated trees mid-session (`0043a484e`).
- CLI UX: declutter `/help` + Ctrl+P palette; `/status` shows reasoning + approval mode + context usage; rotating composer placeholder; type-to-fuzzy-filter `/model` list; compact multi-question clarify panel (`05b4ab0ce`, `9bdff6ab6`, `645f85c2f`, `e931081a4`, `b4526d46d`).
- **`hermes version` consolidated into `hermes --version`** — subcommand removed (`e69b8e561`).
- **`hermes tools`** free keyless vs paid keyed endpoint picker for Exa/Parallel (`f08d3e400`).
- Skills uninstall gains `--yes/-y` (`cd3f9b64d`).

### Web / search providers

- **Keyless web tier → 5-vendor round-robin ring.** `4ea69d9d2` — Exa, Parallel, Tavily, Firecrawl, **Keenable** (new bundled provider). Per-process RR cursor; pin = start-there; multi-hop failover on rate limits; `served_by` marks actual vendor.
- One-shot keyless rescue on keyed-backend failure — never sticky (`d1eefe6ac`).
- Honor stored web backend selection; no silent swaps (`d7119ea2a`).

### Browser / computer-use

- **Authenticated browser control broker** + extension controller actions + scoped artifact endpoints / permission gates / companion journal (`5df1d0e11`, `c9fd5223f`, `1977c3d2e`). Config: `browser.extension_control`.
- Profile-scoped artifact stores; Developer Mode for privileged caps; orphan artifact sweep; vision embed cap for history reuse.
- **Computer-use screenshots exposable for chat delivery** (`188d47919`).
- `drive_preview` / `annotate_preview` — agent can drive the page it opened (`c57581cd0`).

### Kanban

- **Memory-aware dispatch guard** + memory-derived default concurrency cap (OOF-30). `4beca7a94`:
  - Unset `kanban.max_in_progress` → `clamp(MemTotal/512MiB, 2, 8)` on Linux; None (uncapped) when memory unreadable (macOS/Windows dev).
  - Live pressure: critical → spawn nothing; elevated → at most one; reclaim/promote still runs.
- Host-level cap accounting + review-lane reservation (`e1e472d29`); `max_in_progress` before both ready and review queues (`1f8c05783`).
- Worktree reap at task complete/archive; kill tmux worker before worktree remove; drop `--force` from worktree remove (TOCTOU) (`00c184170`, `395eedb18`, `77ed972f4`).
- Desktop native OS notifications on task completion + blocker/failure (`a30636f81`, `9aa141378`).

### Delegation

- Running subagents stay list/steer-visible across parent AIAgent rebuilds via durable `owner_agent_session_id`; child process notifications carry delegation attribution (`b95ec1cb5`).
- **Honor pinned `delegation.provider`** — no silent parent-fallback substitution; missing command → loud spawn refusal (`184cddb44`, #80450).
- Keep worktree when git inspection fails + tell parent (`2b490a051`, `38ea711fd`).
- CLI exposes delegate_task subagent model in auxiliary-models picker (`b7329647a`).

### Approvals & security

- **Coalesce identical concurrent gateway approval prompts** (OpenCode port) — followers wait on leader event (`08d982850`).
- **Plugin install/update security scan** (Cowork-inspired). `d44a29549` — `tools/plugin_guard.py`; safe/caution/dangerous; `plugins.scan_on_install` default true; dangerous blocks even with `--force` on install, deactivates on update.
- Widen Tier-1 advisory scan to license + security (`2c2697b52`).
- `/yolo` reports locked-ON under process-frozen YOLO (`b03503658`).
- execute_code approval CLI fall-through alignment (`f0ffcbc75`, `16af3bed8`).

### Memory

- Cron memory enable (above); boolean config parse consistency; independent built-in store permissions; unify store flags + bypass stale availability cache; profile-only config gets narrow USER_PROFILE_GUIDANCE; drop dead memory tool/guidance when stores off; release handles before profile rmtree (`c809d964d`, `2cf7b36e1`, `5a5d6b966`, `481bc9391`, `d5cddae18`, `4f354c27b`).
- `hermes memory status` aligned with memory tool gate (`b38c40319`); recognize memory nudge interval (`5c03fbedc`).

### Desktop / TUI (architecture-touching only)

- Desktop concentration is the bulk of the window (~211 fix + ~97 feat desktop) — mostly Bot Mode UX, theme/glass/accent plugins, pane ownership, client-direct voice (no audio relay), Send Diagnostics redacted bundle, connector consent card, React Compiler, multi-target update, work profile deletion persistence.
- Architecture-relevant: workspace-scoped pane ownership; per-profile remote overrides from profile rail; plugins theme surface SDK; multi-connection bot roster; error cards name failing layer + recovery actions.
- **Reverts:** Nous support link on Portal-auth error card (`3903428a7` reverts `e3d46bb5f`).

### Providers / Auth / Models (summary)

- OpenCode Free, Ox Alpha, GLM-5.3, Bedrock Responses/GPT-5.6 family, Codex 272K default + `-900k` opt-in, free-model picker UX, keyless auth policy — see Configuration section.
- Vertex explicit-config scoped to Hermes signals not ambient ADC (`b9f17ba3f`); pool-only Anthropic OAuth visible in desktop pickers; Bedrock surfaces when AWS env creds set.

### Skills / Plugins

- Plugin guard on install/update (security section).
- Desktop plugins >512 KiB no longer load truncated; theme switch from outside React; accent picker plugin (off by default).
- hermes-agent skill routes unknown-feature questions to `llms.txt` (`dc8481b78`).
- OpenRouter Image API surface on openrouter image_gen backend (`d6e6e8b60`).
- Relay native dynamic plugin cutover / 0.7 APIs (multiple commits).

### Nix / packaging

- Home Manager `programs` module + desktop app (`76f6ba370`); wait for backend bind target before start (`fd3a783a3`).

### Reverts

| SHA | What |
|---|---|
| `3903428a7` | Revert desktop error-card Nous support link on Portal-auth sessions |
| `250232ff9` | Revert agent canonical tool-call deduplication harden |
| `587ad8748` | Revert preserve local reasoning timeout opt-out |
| `79e6d3e6d` + 5 more | CI workflow parse-cache buster thrash (noise; not architecture) |

### Salvaged / ported

- OpenCode approval coalesce (#40869), Goose MCP TAG-char strip, Kimi MCP `_meta` surface, Claude Cowork plugin scan pattern, Copilot `/worktree`, Keenable free-tier proposal (#49758), fattchris `resolve_turn_limit` foundation (#67696).

## Docs surface drift (live site vs offline SSOT)

Official docs now foreground **Bot Mode**, `hermes peer`, `hermes project`, `hermes proxy`, `hermes egress`, `hermes lsp`, `hermes security audit`, desktop installer path, and `/llms.txt` + `/llms-full.txt` machine indexes. Offline `Hermes_Architecture.md` still reflects ~June baseline and will mis-state `agent.max_turns`, MCP SDK generation, keyless web ring, control socket, fleet update receipts, and Bot Mode A2A unless changelogs are read first.

## Action items for next full reference re-compile

1. **Config defaults:** document `agent.max_turns: null` (unlimited) + `resolve_turn_limit` spelling table; remove stale 90/500 defaults.
2. **Providers:** OpenCode Free keyless policy; Ox Alpha; GLM-5.3; Codex 272K + `-900k` variants; Bedrock Responses path; 5-vendor keyless web ring + Keenable.
3. **MCP:** SDK 2.x + 2026-07-28 protocol negotiation (`protocol:` key); 50K spill; `oauth.user_agent`; CIMD.
4. **Gateway:** control socket identify/status; Bot Mode `message_agent` + `hermes peer`; multiplex fail-closed; loop watchdog knobs.
5. **Update subsystem:** receipts, `--plan`, sibling config migration, fleet matrix, process-identity ledger.
6. **Cron:** memory-on; per-job reasoning_effort; bot-chat delivery target.
7. **Kanban:** memory-derived max_in_progress + live pressure guard.
8. **CLI:** worktree list/prune; peer; version consolidation; /status fields; plugin_guard.
9. **Browser:** control broker + extension controller + artifacts.
10. **Compression:** identical-recall stubs; would-grow salvage; mid-turn overflow guard.
11. **SCHEMA_VERSION:** confirm still 26; document git_metadata_generation only if not already from 08-16 scaffold.
12. Re-compile `Hermes_Architecture.md` from upstream + consume full changelog chain 07-05…08-23; bump `repo_commit` to `f293e7206`.

## Known limitations of this pass

- Lightweight-local only — no full `Hermes_Architecture.md` rewrite.
- Spot-verified ~30 commits; desktop UI churn summarized, not line-audited.
- Local install is on v0.20.5 / `fcbd1076a` (310 behind tip); behavior claims for post-local commits are from git+docs, not runtime probes on tip.
- Official docs sampled (home, CLI, configuration) — not full docs crawl.
- Public mirror layout is flat (root-level CHANGELOG/SKILL/Architecture), not `references/` — sync matches existing mirror convention.

## Files updated this pass

- Local skill: `references/CHANGELOG-2026-08-23.md` (new)
- Local skill: `SKILL.md` frontmatter + Targeted quirks pointer (version 1.9.0 → 1.10.0)
- Public mirror: `shagghiesuperstar/hermes-master-ref-maintained` — same changelog + SKILL.md
- `references/Hermes_Architecture.md`: **untouched** (changelog-as-scaffolding)
