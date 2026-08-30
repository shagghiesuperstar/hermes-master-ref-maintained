# Hermes Architecture Reference — Drift Changelog 2026-08-30

## Scope of this review

- **Local Hermes version:** v0.20.6 (2026.8.27) — local `a9e72f1b58` (on `main`, **332 commits behind** `origin/main`)
- **Upstream HEAD reviewed:** `4f22543509` (origin/main as of 2026-08-30)
- **Prior SSOT anchor (last drift review upstream HEAD):** `f293e7206` (CHANGELOG-2026-08-23)
- **Prior full SSOT anchor:** `cdcbc3a31` (v2026.7.1) — `references/Hermes_Architecture.md` last synced 2026-06-21
- **Commits between prior upstream and current upstream:** **1,649** (~152 feat, ~967 fix, ~74 refactor, rest chore/test/docs/style/fmt; **7** reverts)
- **Window:** 2026-08-23 → 2026-08-30 (~1 week)
- **Releases in window:** **`v0.20.6` (2026.8.27)**
- **Full SSOT drift (`cdcbc3a31` → origin/main):** **11,818** commits pending
- **Methodology:** lightweight-local model (per operator/cron directive) + git-log area-bucketing + spot-verification (`git show --stat` + body) on ~35 high-impact commits. Official docs spot-check (CLI commands, configuration, cron). **Changelog-as-scaffolding** per the drift review decision tree (1,649 commits ≫ 500; operator said "lightweight local model only"). The main reference (`Hermes_Architecture.md`) is left untouched.

> **Why a changelog and not a full re-compile.** Delta is above the 500-commit re-compile threshold, the full SSOT is now ~10 weeks / ~11.8k commits stale, and the cron directive is lightweight-local only. This pass continues the changelog chain (07-05 → 07-12 → 07-19 → 07-26 → 08-09 → 08-16 → 08-23 → **08-30**). **A full re-compile remains critically overdue** and should be authorized on the next non-lightweight pass.

## High-impact architecture deltas (`f293e7206` → `4f22543509`)

### Configuration & provider routing

- **New / expanded providers.** Ramp Router plugin (`804f8b4732`), Nebius Token Factory (`13bad590f3`), hy4-preview + tokenplan (`0fb5cab0d4`), Alibaba CN curated picker lists (`93b6cf2ef8`, `46076b2d6b`), GLM-5.3-Flash on z.ai / OpenCode Go / OpenRouter / Nous (`a9611f3c6f`, `64424a16a2`), qwen3.8-flash (`48d2528066`), Inkling free + minimax-m3:free OpenRouter entries (`51df117def`, `6607f70673`).
- **`delegation.request_overrides` complete.** `bacb90fe20` — honored on all three resolution branches (direct endpoint, named provider, parent-inherit) with explicit-over-runtime merge precedence; deep-copy isolation; DEFAULT_CONFIG + docs.
- **Hermes-Agent User-Agent on Router requests** (`ceb54f5a52`).
- **Docs-confirmed (live site 2026-08-30):** CLI still documents `hermes peer`, `hermes project`, `hermes proxy`, `hermes egress`, `hermes lsp`, `hermes security audit`, `hermes approvals`, `hermes dump`, `hermes prompt-size`, `hermes migrate`, `hermes send`; global flags include `--in`, `--ignore-user-config`, `--ignore-rules`, `--tui`/`--cli`; config docs still show `config get`/`unset`, managed scope, database journal_mode, runtime nofile soft limit.

### Schema versions & state.db migrations

- **SCHEMA_VERSION remains 26** (`hermes_state_common.py` on origin/main). No new state.db schema bump in this window.
- Related non-state schemas: observer/middleware telemetry schema version strings; relay runtime `hermes.relay.runtime.v1`; dashboard lockfile schema v2; verification evidence schema v1 — not SessionDB migrations.

### Compression / context

- **Lean tail retention is the DEFAULT.** `6e5413844e` — compaction keeps clamped 2.5%-of-window tail (10K floor / 25K cap), not the legacy threshold×target_ratio hoard (could keep 100–240K+ on big windows). Explicit `compression.tail_mode: legacy` restores old behavior. `update_model()` recompute bug fixed (was silently reverting lean → legacy). Gateway cache-busts on `compression.tail_mode`.
- **System prompt always rebuilds at compaction commit boundary.** `514707ff3e` (#98426) — keep-prompt gated on byte equality of LIVE builder output; plugin sections re-render (fail-open to last good bytes). Forever-sessions finally pick up identity/config prompt changes.
- **Dynamic tool schemas rebuild at compaction commit.** `c30ac90a92` (#97073) — MCP/tool schema refresh on forever-sessions.
- **Context size anchors on provider-reported usage.** `d3a1c46510` — `capture_usage_anchor()` / `anchored_context_tokens()`; estimate only messages since last main-loop usage; structural fail-closed on transcript rewrite; MoA uses pre-fold aggregator usage; aux/advisor never anchor.
- **Two-line conversation clock.** `11b98a1429` (#97930) — anchored "Conversation started" via session-id embedded timestamp → session_start → now; rebuild-day line separate so compression/resume no longer advances birth date. Compacted Bot Mode forever-chats keep original birth (`514707ff3e` companion).
- **Default identity rewritten as behavior spec.** `f89f0a2eaa` (#97926) — sizing rule, named prohibitions, anti-sycophancy, earned depth; exploration-thrift line **removed** (models under-explored).
- **Prompt diets (token-per-call):** platform-hint diet −657 tok (`e387cbc0aa`); desktop hint diet 442→307 (`9d9f44d638`); memory/skills guidance 537→255 (`a2e19d484c`).
- Tip-of-tree fix: lean compaction makes exactly one auxiliary request per attempt (`4f22543509`).

### Skills / commands / tool surface

- **Shipped-skills slim (−26% index).** `c49fa88b80` (#98539) — 15 skills → optional-skills (creative comfyui/ascii-art/excalidraw/pretext/sketch/touchdesigner-mcp; ALL mlops; research-paper-writing; openhue; blogwatcher). DELETED session-librarian. **github** six skills merged into one `software-development/github`. **pdf** absorbs ocr-and-documents + nano-pdf. Channel-gated teams pipeline. Desktop skills index ~1,900→~1,400 tok/call.
- **`skill_manage` → `operations[]` IS the call.** `72874b0675` (#97295) — each op names its skill; atomic cross-skill rollback; single op = list of one; intra-batch same-file clobber guard; staged as one pending write under approval gate.
- **`/plan` graduates from bundled skill → built-in CommandDef.** `0f3fcacd3f` — guaranteed core-tier menu slot on every platform (Telegram/Discord caps were dropping the skill-tier entry). `agent/plan_prompt.py`; PROTECTED_BUILTIN_SKILLS now empty (mechanism kept). Salvages #67292 / closes #67264.
- **`/review` built-in** — independent reviewer subagent on every surface (`12395e57b4`, `agent/review_engine.py`); review slot in every aux-model picker; briefing carries parent skills + project context files.
- **`/btw` full-context side questions** via background-review cache-parity fork (`578f85cfb0`); `/background` renamed `/bg` (`74a95a3ddf`).
- **Cross-profile write guard RETIRED.** `eff97a8a05` (#97165) — maintainer decision: profiles are not isolated; mirror lost-write guards survive; patch/write_file schemas drop `cross_profile` (−83 tok/call). **Architecture impact:** prior reference "profile isolation" write-guard claims are stale.
- Optional skills: decision-questionnaire, setup-wizard-generator, plan-interrogation (renamed from grill-me), impeccable hub entry, publish-site.

### Tool schema diets & delegation

- **`delegate_task` tasks-only interface + depth-derived delegation.** `9dfbde19db` (#96424) — 1,201 → 773 tok/call (−36%).
- **Patch V4A mode gated to OpenAI-family mains** — base schema replace-only (`dadbfd8990`, 365→195 for everyone else).
- **Process / todo / skill_manage / execute_code / image_generate / video_generate / read_file** schema diets with capability gating (multiple SHAs; read_file also bundles firecrawl-anydoc 0.2.4 + typed NeedsOcrError / FIRECRAWL_API_KEY-gated hosted OCR — `a9e72f1b58`).
- **A2A client tools config-gated** — served only when a2a_agents configured / inbound enabled / A2A_PORT set (−561 tok/call unconfigured) (`3340bbbdad`).
- **Todo nested subtasks** via optional `parent` field (`b6bd681e89`).
- Subagents embed workspace project context files (`7526bd39a8`).

### Code execution / terminal

- **Remote kernel host** for docker/ssh/modal session persistence (`5f75ec197b`, #96991) — `tools/code_kernel_remote.py`.
- **Session kernels always on** — `kernel_mode` knob retired (`4e7eb39947`); earlier session-persistent kernels feat `b39d76d902`.
- **Stdout spillover** — truncated execute_code full text to cache/exec with read_file recipe (`ae8c976032`).
- **Session temp root default** `~/.hermes/cache/terminal` (off tmpfs `/tmp`); 72h auto-prune; `terminal.temp_dir` config (`95cf7dc9e8`, `d7be3f649d`).
- **Pluggable terminal environment backends** via plugin registry (`0484910787`) — `agent/terminal_env_provider.py` + `terminal_env_registry.py`.
- **Shared Docker container identities** (`7a67bd07a7`, `82b32f32ef`) — profile-scoped resolver + MEDIA delivery.

### MCP

- **Large remote MCP catalog expansion** — 34 official vendor-hosted + Better Stack/Railway + Cloudflare API + Grafana Cloud + 18 more live-verified (`753f362a23`, `99f5aec733`, `9a37325717`, `90fd9a838b`, `56f94faaef`).
- **Cloudflare curated exclude list** + glob tool filters + `default_excluded` manifests + `?codemode=false` pin for tool_search surface (`3a1a3a1c8f`, `53015d3eb5`).
- **Desktop MCP OAuth remote callback relay.** `0f5dd5c46e` — client-side 127.0.0.1 listener + `client_redirect_uri` + `mcp.servers.oauth.callback` RPC so SSH/Tailscale remote backends complete OAuth (mirrors native gateway login pattern).

### Cron

- **Durable failure incidents + ack.** `9de5460c12` (#95017, salvage #94692) — `cron/incidents.py` in executions.db; signature dedup (job_id + normalized error); lifecycle detected→alerted→reviewed→closed; acked signatures suppress re-ping; `hermes cron incidents` / `incidents ack <id>`.
- **Explicit one-shot re-arm** (`a0ca7c1920`).
- Docs still confirm: per-job reasoning_effort pin; model pin fail-closed drift guard; memory-on for cron agents (prior week); bot-chat delivery.

### Gateway / Bot Mode / Update / fleet

- **Updaters pause gateways over control socket** instead of tree-kill (`03537d69dc`, #92091 step 2) — `pause-for-update` verb; Windows drain ACK extends wait to gateway-declared budget; legacy force-kill fallback retained.
- **Image/package-managed installs refuse in-place updates** through shared `evaluate_update_admission()` (`4860978115`, #91277 Phase 3) — image-provenance marker authoritative; exit 2 + refused receipt; docker-pull guidance.
- **Network-bound serve backends survive hermes update** on recorded endpoints (`27385e586b`, #63206).
- **Bot Mode reliability cluster (#93091):** typed failure-reason codes; envelope TTL + offline fast-fail; push-notified relay drain; per-profile turn lock (concurrent deliveries queue); retry/resume + compress-and-resume on overflow; needs-attention badge for background bot failures; A2A typed failures to sending agent.
- **WS slim path / event replay:** slim WS-only desktop boot path landed (`434ea57eb0`, `87631bd8ae` seq-stamped replay) then **PR #94245 gw-event-replay was REVERTED** (`9f05b06589` / merge `2ea42a44e0`) — do not document event-replay as current architecture without re-checking tip.
- Gateway ping heartbeat wire contract + socket-generation invalidation (`9a71cb95cb`, `9153be2a51`).

### Web / browser / computer-use

- **TTL result caching** for `web_search` + `web_extract` (`04603fc040`) — 20m default TTL, single-flight, limit bucketing, disk-backed extract index; `web.cache_enabled` / `web.cache_ttl_minutes`; rescue responses never cached.
- **`cache_exempt_hosts`** always-live fetches for staging/tunnel (`ba9fc55e16`).
- **Browser snapshots drop LLM summarization** — truncate-and-store like web_extract; `auxiliary.web_extract` slot removed (`a75ea37dc5`).
- **Real-profile browsing** via agent-browser copy + browser-use CDP; Brave Origin; consent-gated Windows auto-close; Desktop Capabilities toggle (`1f4d095fd8`, `830e4a29be`, `b6d535dd88`, `6cb6aeb168`).
- Computer-use: guide models from full-screen grabs to interactive lanes (`ce9b9a6351`).

### Memory

- **Opt-in fail-closed pre-compress checkpoint contract (API v1).** `1104ffe0b9` — memory provider hook before compaction; config + gateway/slash wiring.

### Tool-search / loops / CLI UX

- **tool_search multi-query + batched describe + Snowball stemming** (`e455e4afd0`).
- **`/loop` first wakeup fires immediately by default** (`1a47a36422`; earlier `--start-now` `796babaaed`).
- **TUI/CLI status bar** — cache-hit %, latency, t/s; `display.status_bar.fields` (`86a2fdc634`, `3548fc809b`, `4bc7e624d6`, `fb786d2f5b`).
- **`-q` seeds a live interactive session** with literal prompt submit (`a5c7eed5f3`).
- Telegram inline command picker — search every command and skill, no menu cap (`5bdaea64ed`).
- Discord exposes `/plan` in native slash picker (`83f4524b42`).

### Kanban / plugins / desktop (architecture-relevant subset)

- **Board export/import** portable archive + REST + Desktop switcher (`3150e444b2`, `5e550838f7`, `72cf8d1fac`); review handoff summary into wake turn (`1f92c5d4ce`).
- **Plugins:** generalize native platform handler registration to every gateway platform; Telegram PTB handlers via `ctx.register_telegram_handler`; wire into a2a/buzz/qqbot (`272f4e4abe`, `c96f830252`, `34393c32aa`).
- Desktop: fleet profile rail; managed SSH remote update engine; in-app Browser OS window/multi-tab; macOS TCC identity command (`hermes desktop --setup-tcc-identity`); OS-keychain encryption opt-in (no more every-launch prompt); Download button on preview files; tips/tours; Bot Mode design-system rebuild. Heavy fix volume (`fix(desktop)` = 292 commits) — treat desktop as fast-moving UI layer, not core SSOT.

### Profiles / security / approvals

- Cross-profile write guard retired (see Skills) — largest security-model semantic change this week.
- Approvals: multiple `fix(approval)` hardening commits; no new default-mode flip spotted in feat inventory.
- macOS Full Disk Access one-switch guidance in doctor/setup (`be85903234`); TCC interpreter anchor later reverted (`2f9e187001`).

### Reverts

- **`Revert "Merge pull request #94245 … feat/gw-event-replay"`** (`9f05b06589`, merge `2ea42a44e0`) — large deletion of tui_gateway event_replay / entry_ws WS-replay path; do not keep event-replay as current.
- **Revert removal of stealth/ox-alpha** from OpenRouter/Nous catalogs (`03c97d984b`) — ox-alpha stays listed.
- **macOS TCC interpreter anchor removed** (`2f9e187001`) — anchored copies could not load libpython.
- Lint shared eslint configs kept out of branch (`531a8cd9c7`).
- Desktop stale-branch restore after revert (`5a285d3436`).
- Test guard against reintroducing generic image-strip fallback after #69078 (`ca02d3c218`).

### Salvaged PRs / authorship notes

- `/plan` salvages #67292 (@webtecnica).
- Cron incidents salvage #94692 (#95017).
- System-prompt conversation clock salvages #96224 (#97930).
- Delegation request_overrides completes #90953 salvage.
- skill_manage batch / compaction prompt rebuild / lean default carry maintainer-directed schema-diet program from prior weeks.

## Concentration (this window)

| Area (approx commit-subject hits) | Count signal |
|---|---|
| desktop | 414 |
| gateway | 150 |
| bot | 121 |
| cron | 80 |
| update | 75 |
| agent | 71 |
| auth | 61 |
| skill | 59 |
| browser | 54 |
| provider / compress | 50 each |
| mcp | 37 |
| memory / approval | 15 each |
| kanban | 8 |

Top conventional scopes: `fix(desktop)` 292, `fix(gateway)` 51, `fix(cron)` 47, `feat(desktop)` 35, `fix(update)` 34, `fix(agent)` 32, `fix(compression)` 24.

## Action items for next full reference re-compile

1. **Compression:** rewrite defaults — `tail_mode: lean` default; provider-usage anchors; always-rebuild system prompt + dynamic tool schemas at commit boundary; two-line conversation clock; identity behavior-spec text.
2. **Skills index:** document shipped-set slim, github merge, pdf+OCR absorption, optional-skills moves, empty PROTECTED_BUILTIN_SKILLS, `/plan`+`/review` as builtins, `skill_manage.operations[]`.
3. **Security/profiles:** remove "cross-profile write guard" as active protection; document maintainer decision that profiles are not write-isolated; keep mirror lost-write guards.
4. **Tool schemas:** document per-tool diet numbers + V4A OpenAI-family gate + A2A config gate + delegate tasks-only + nested todos.
5. **Code execution:** remote kernel host, session kernels always-on, temp_dir default under `~/.hermes/cache/terminal`, pluggable terminal env backends, shared docker identities.
6. **Web:** TTL caches + cache_exempt_hosts; browser snapshot no-summarize; real-profile browsing.
7. **Cron:** incidents store + ack CLI + one-shot re-arm (stack on prior memory-on / reasoning_effort / bot-chat delivery).
8. **Gateway/update:** pause-for-update control-socket verb; image-managed refuse gate; Bot Mode #93091 reliability; **exclude** reverted event-replay unless re-landed.
9. **MCP:** catalog bulk + Cloudflare excludes + Desktop remote OAuth callback relay (stack on MCP 2.x / 50K spill from prior week).
10. **Memory:** pre-compress checkpoint contract API v1.
11. **Providers:** Ramp Router, Nebius, hy4, GLM-5.3-Flash, qwen3.8-flash, free Inkling/minimax entries.
12. **CLI/docs parity:** `/plan`, `/review`, `/btw`/`/bg`, `/loop` immediate first wakeup, status_bar.fields, `hermes cron incidents`, `hermes desktop --setup-tcc-identity`.
13. **SCHEMA_VERSION:** still 26 — confirm no silent migration missed.
14. **Consume entire changelog chain** 2026-07-05 → 2026-08-30 before rewriting `Hermes_Architecture.md`.

## Known limitations of this pass

- Lightweight-local / cron directive — no full `Hermes_Architecture.md` re-compile.
- Spot-verified ~35 high-impact commits; not every feat/fix body.
- Desktop (414 subject hits) summarized at architecture-relevant subset only.
- Official docs extracted via static HTML (Docusaurus); JS-rendered sections may be incomplete — CLI/config/cron pages confirmed present and aligned with cited features.
- Local install is 332 commits behind reviewed origin/main; runtime behavior on this machine may lag tip (v0.20.6 local vs tip `4f22543509`).
- Full SSOT now **+11,818 commits / ~10 weeks stale**.

## Files updated this pass

- `references/CHANGELOG-2026-08-30.md` (this file)
- `SKILL.md` frontmatter + Targeted quirks pointer (version → 1.11.0)
- Public mirror: `shagghiesuperstar/hermes-master-ref-maintained` (same changelog + skill bump)
