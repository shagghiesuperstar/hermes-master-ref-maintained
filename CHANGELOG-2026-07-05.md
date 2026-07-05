# Hermes Architecture Reference — Drift Changelog 2026-07-05

## Scope of this review

- **Local Hermes version:** v0.18.0 (2026.7.1) — local 61e4a5f9
- **Upstream HEAD reviewed:** `5b04a024a` (v2026.7.1, origin/main)
- **Prior local SSOT anchor:** `7cfa2fa1` (v0.17.0) — `references/Hermes_Architecture.md` last_synced 2026-06-21
- **Commits between anchors:** **1,098** (116 feat, 592 fix, 100+ refactor; remainder tests/chore/docs)
- **Public mirror:** `shagghiesuperstar/hermes-master-ref-maintained` synced identically with local SSOT (commit `563803c`, "Weekly drift sync: 2026-06-21 upstream changes")
- **Live `hermes config check`:** clean (config v32→33 update available only)
- **Methodology:** lightweight-local model + targeted git-log review + spot-checks on architecture-relevant areas. The 116KB reference is not a full rewrite candidate in a single cron pass; this changelog captures the deltas a future re-compile must incorporate, with source SHAs for verification.

## High-impact architecture deltas (v0.17.0 → v0.18.0)

### Configuration & provider routing

- **`config.yaml` v32 → v33 migration available.** `hermes config migrate` should be run before relying on new keys. (All other v33 keys are optional.)
- **New: per-provider `extra_headers` for LLM API calls.** `b98baa303` — Named providers and `custom_providers` entries now accept an `extra_headers` dict scoped to that endpoint (reverse proxies, API gateways, custom auth such as Cloudflare Access). Normalized in `hermes_cli/config.py` (`_normalize_custom_provider_entry`); surfaced through `runtime_provider.py` and merged onto the OpenAI client `default_headers` at construction and on every credential swap. Per-provider keys are matched on `base_url` (case/trailing-slash insensitive). OpenAI-wire only — native Anthropic/Bedrock out of scope.
- **New: `config.channel_overrides` (Discord thread/parent → model + system prompt).** `c43aa6301` — `ChannelOverride` dataclass + bridge from `discord.channel_overrides:` YAML. Resolution order: `session /model > channel > global`. Guard added when config lacks platforms (test mocks).
- **New: `display.tool_progress: 'log'` mode.** `39bff6795` — In addition to existing `off`/`status`/`new`/`all`, gateway now supports a log-only mode.
- **New: `gateway.platform_connect_timeout` config key.** `7b1275394` — Was previously hard-coded; now tunable in config.yaml.
- **New: `gateway.typing_indicator` per-platform toggle.** `05ac16778` — Lets noisy platforms suppress the typing indicator.
- **New: xAI Grok OAuth is now device-code-only.** `5ef0b8acb` — Loopback login dropped; the only first-party OAuth flow is device code.
- **New: Google Vertex AI provider for Gemini (OAuth2).** `c73e74386` — Native provider, not via the OpenAI-wire shim.
- **New: API server per-client model routing via `model_routes`.** `4a09b692e` — Lets the API server route different client API keys to different model/provider combinations (salvage from PR #3176).
- **New: STT transcript echo toggle.** `bfc526272`, `4be749d15`, `406eb719c`, `558001307` — `stt.echo_transcripts` surfaced in desktop settings and docs; interrupt STT transcript echoes gated.

### Skills, commands, and CLI

- **Stacked slash-skill invocations.** `9767e19b6` — `/skill-a /skill-b do XYZ` now loads up to **5** leading skills in one turn. New `agent/skill_commands.py: split_stacked_skill_commands()` + `build_stacked_skill_invocation_message()`. Existing bundle scaffolding markers preserved so `extract_user_instruction_from_skill_message()` keeps memory providers storing the user instruction, not N skill bodies. CLI and gateway both dispatch the stacked path. 11 new tests + docs section in `skills.md`. Inspired by Claude Code v2.1.199 (2026-07-02).
- **CLI autocomplete + ghost text for stacked slash-skill invocations.** `2c0820c9f` — UI support for the new stacked form.
- **New `/compact` alias for `/compress`.** `ce9aa869f` — Plus `--preview` and `--dry-run` flags (salvage from PR #3243).
- **New `/sessions search <query>`.** `19d417445` — Search across session transcripts from the gateway.
- **New `/debug [nous|local]` slash command.** `98d550e03` — Toggle diagnostic upload destination.
- **New `hermes journey` command** — but with two `%-d` strftime portability bugs (`ce82b0c3c`, `7e037e1a3`) on Windows. Mac/Linux fine.

### Delegation

- **`delegate_task()` no longer accepts `toolsets` from the model.** `ba0bc01d1` — Toolset selection is a capability-scoping decision; subagents always inherit parent's enabled toolsets. Removed from `delegate_task()` signature, the registry handler, the top-level + per-task JSON schema, and the live dispatch path (`run_agent._dispatch_delegate_task`). Tests flipped to `assertNotIn`; regression test confirms dispatch never forwards a smuggled `toolsets` arg.
- **Delegation concurrency caps unified.** `6e369a376` — `max_async_children` deprecated; replaced by unified caps.
- **Kanban notifications route via owning profile + wake creator agent.** `c69643026`.

### MoA (Mixture of Agents)

- **Per-preset `fanout` cadence.** `9e044cf79` — Two modes:
  - `per_iteration` (default, unchanged): reference fan-out re-runs whenever advisory view changes.
  - `user_turn`: advisors run ONCE per user turn, aggregator acts alone for the rest of the tool loop. Cache signature hashes only the prefix up to the LAST user message, so mid-turn advisory-view growth → cache HIT → zero advisor spend. New user message re-triggers fan-out. **This is the obvious lever on MoA's wall-time/cost multiplier** — advisor generation dominates per-turn latency.
- **`reference_max_tokens` cap on advisor output.** `543d305bb` — Cuts turn latency.
- **Full-turn trace persistence to JSONL (opt-in).** `2e8748ed2`.
- **Unified slot provider-identity on the single `call_llm` chokepoint.** `a653bb0cb`.

### Gateway

- **Per-session `/model` overrides persist across gateway restarts.** `30e947e0a` — `_session_model_overrides` was in-memory only; now persisted to `sessions.json` and lazily rehydrated. Persisted fields: `model`, `provider`, `base_url` (allowlist). **Never `api_key`** — credentials are re-resolved through the normal runtime provider path.
- **Per-channel model + system-prompt overrides (Discord).** `c43aa6301` (above).
- **AsyncSessionDB offload facade.** `ea26f2271` — New gateway session storage abstraction.
- **Suppress home-channel shutdown broadcast on flagged drains.** `b963d3238`.
- **Per-category context breakdown in `/usage`.** `6aefc9d92`.
- **Generic gateway status phrases + busy steer ack configurable.** `4bf5b563b`, `12f03b11f`, `46fbd73f6`, `b9de7044a`, `d111faa3a`, `14c91ade3`, `fddc95f4c`. Long-running notifications can show a status phrase; the busy-steer env override is preserved.
- **Drain async log queue on `os._exit` shutdown backstop.** `ac68a6411`.
- **PATH bootstrap moved below imports, gated to POSIX.** `619db0175` — System dirs added for UV Python compatibility.
- **Native image handoff keyed by session, queue preserved.** `bb24ac6f2`, `e88039648`, `741bd9ba4`, `c13281ab5` — Local file inputs routed through shared credential-read guard.
- **`display.tool_progress` `log` mode** (above).

### Compression / prompt caching

- **Compressor must keep a user turn when compaction would drop the last one.** `24add1db7` — Critical correctness fix; old behavior could leave the conversation with no user turn. Followed by `d8504df7e` (refactor: reuse `_fresh_compaction_message_copy`) and `10ced0567` / `b2c55582e` (regression tests).
- **Pre-API compaction gated through the preflight guard chain.** `90f84144e` — One chokepoint for safety checks.
- **Skip invalid top-level `cache_control` on empty assistant/tool messages on OpenRouter.** `8b797f7a7` — Prevents prompt-cache invalidation.
- **Mid-session `/model` switch breaks prompt caching** (documented warning). `1c156736d` — Explicit docs callout.

### Memory

- **Guard local uploads against credential reads.** `e02cef0d0` — Vision, image-gen, and any local-input path no longer accidentally reads API keys.
- **Hindsight daemons: `journey` routes memory mutations through `MemoryStore` atomic I/O.** `2fc67a3a5`.
- **Auxiliary vision reads model from `config.yaml` before env var.** `149641485` — Priority corrected.
- **Vision local-file inputs routed through shared credential-read guard.** `9ae17b8ac` — Same `e02cef0d0` family.

### MCP

- **First-class MCP tab in the desktop GUI.** `16aa09aca` — Cursor-style manager inside Capabilities:
  - Server list with brand/favicon avatars + live status dot + capability summary.
  - Catalog: one-click install of Nous-approved servers with required-env prompts.
  - GUI OAuth: Authenticate opens the system browser from the TTY-less backend and verifies a token actually lands; header/API-key servers never pushed to OAuth; dirty `mcp.json` cannot drop a freshly-persisted auth field.
  - Full-width `mcp.json` editor (ecosystem document format) + pinned stdio/agent commands.
- **Surface MCP server log notifications in `agent.log`.** `edf8e0ba9`.
- **Fail fast at OAuth redirect/callback boundary when non-interactive.** `755194ffe`, `0c8441c88` — `refactor(mcp)`: DRY the non-interactive OAuth guard (`af0ce1cf8`).
- **DRY the non-interactive OAuth guard + positive-control test.** `af0ce1cf8`.

### Tools

- **Recursively normalize JSON-string tool args by schema** (port from `cline/cline#11803`). `8a04b516a` — Pre-API sanitizer expansion.
- **Pre-tool-call approve action escalates to human gate.** `f512d6f02` + `368e5f197` — `feat(plugins): pre_tool_call approve action` is the new mechanism.
- **`/deny <reason>` relays denial reason to the agent** (port from `nanoclaw#2832`). `cb6c47af0` — Denial reasons now flow back so the model can correct course.
- **Approval hardline token requires exact `./`/`..` segments.** `ebfc49c4d` — Prevents bypass via sloppy path matching.
- **First-class x-api-key providers + hot reload via management API.** `86fcb2fe5` — Egress layer.
- **`resolve_httpx_verify` for custom CA bundle TLS.** `7e957cbd0`.
- **Pool-level keepalive replaces custom socket_options transport.** `8324dd19c`, `51c1ba697` — Standard `httpx` pool-level keepalive expiry.
- **Image-gen: support Codex image inputs.** `ecffd290a`.
- **Image-routing: check stripped `custom:<name>` provider key for vision override.** `5e1162854`, `f6a3d2e90`, `769469a70` — Custom provider slug preserved across routing.
- **TTS: coerce direct-only OpenAI model on the managed audio gateway.** `b53ba0e18`.
- **Vision: parse `(label)` and `= "value"` AX element label forms.** `519ec7b3b` — `computer_use` tooling.
- **`computer_use` re-fetch via CLI when MCP returns silent-empty captures.** `13b75e73f` + fallback to CLI transport when cua-driver MCP bridge hits `EAGAIN` (`7af9abd17`).
- **`cua-driver` installer: drop `shell=True`, download to mkstemp, exec as argv.** `24a754691`.
- **`cua-driver` installer timeouts: group-kill, stale-lock pre-clear, 660s ceiling.** `7fde19afc` + startup timeout raised 15s→30s (`d537d29a6`).
- **`update` skips `cua-driver` refresh when `Applications` is unwritable.** `d8b51269c`.

### Webhooks

- **Timestamp-bound V2 signature for generic webhook replay protection.** `70449a493`, `f3d577408f` — Reject generic V2 signature missing timestamp instead of falling back to V1.
- **Rate-limit V1 deprecation warning + document V2 signature.** `708b57e00`.

### Cron

- **Cron jobs run under the profile secret scope** (not the active process's scope). `fdab380a1` + regression test `def6d6fe1`. This matters: previously a `hermes cron run` from a context with different env vars would see different secrets than the live agent. Now it's the scheduled profile's secrets.
- **Telegram forum vs channel DM topic disambiguation at delivery time.** `fc31f14cd` (fixes #52060). Topics that the bot's session thought were DMs can now be reclassified to forum topics if Telegram's "General" topic ID matches a real forum thread.
- **Flat in-channel continuable cron delivery surface (Slack + cron).** `4b4349eb9`.

### Security

- **Fail fast on TLS certificate verification failures with fix hints.** `4751af0a0` — Error path now tells the operator what to do.
- **MEDIA tag resolution anchored to safe paths in `api_server`.** `16332af60` — Prevents MEDIA path traversal.
- **Generic V2 webhook signature now requires timestamp.** `d577408f3`.

### Web tools

- **Single registry authority for custom-provider availability.** `a9cd0e07c`, `0a9d42ce4`, `e4105a2ff` — No more scattered `is_available()` checks; one path through provider registry.

### Models (curated lists)

- **Added `claude-fable-5`, `claude-sonnet-5`, `fugu-ultra` to curated OpenRouter + Nous lists.** `76a468e51`.

### Desktop

- **Capabilities page unifies Skills/Tools/MCP.** `7e6d60aad`, `e0325cf76`, `929ba007b`, `c7103c637` — Hub-backed by React Query + per-item store. New "Tools" tab inside Capabilities. Skill Hub in Capabilities.
- **CLI/dashboard parity for skills hub, MCP test/toggle/catalog, maintenance ops, log filters.** `c7103c637`.
- **`/journey` opens the memory graph overlay instead of printing text.** `931e2356a`.
- **Collapse long tool-call runs into an auto-scrolling window.** `f36cdd9a4`.
- **Profile rail collapsed to a select past 13 profiles.** `6cffc37b5`.
- **STT transcript echo toggle surfaced in desktop settings.** `558001307` (above).

### WhatsApp

- **Native Baileys polls, clarify-as-poll, locations, rich inbound metadata.** `11627fdcb` + `b0f2bdbe8` (gate poll-vote events to Hermes-created polls).

### Telegram

- **Forward keepalive limits into fallback transport.** `01ee312de` + docs `5b04a024a` (clarify fallback-branch limits wiring vs siblings).
- **Paginate model provider picker.** `7a648a8bf` — Long provider lists now paginate.

### Discord

- **Optional admin-only gate for exec-approval buttons.** `605727e3b` — Bot only shows approve/deny to admins.

### Slack

- **Render markdown tables as native Block Kit table blocks.** `7c7b48981`.
- **Opt-in Block Kit rendering for agent messages.** `b080b93ad`.

### Provider routing / runtime

- **Anthropic-specific guidance for subscription exhaustion.** `e00800fc8` — Classifier knows to surface actionable copy when the user's Anthropic subscription quota is exhausted.
- **Forward `reasoning_effort` for custom providers (GLM-5.2 on ARK).** `f69a33794`, `67df958db` — Custom provider emit at the live profile path.
- **Codex commentary phase out of user-visible text.** `ea125dd62`, `538173f67`, `b3b1e58ad` — Commentary is routed to the reasoning channel (fixes #41293).
- **OpenRouter "no tool use" 404 → `model_not_found` with fallback.** `2bb11adb4` — Cleaner error classification, falls back to next model instead of hanging.
- **Codex runtime: dedupe tool_call_id across pre-API sanitizers.** `dba585c17` (PR #58327), `7203898ce` (PR #58350).
- **Match tool results on `call_id||id` in pre-request repair.** `88f2c0caf` (PR #58168).

### Knowledge cutoff caveat

- Live official docs at `https://hermes-agent.nousresearch.com/docs/` were not fetched during this cron run (lightweight-local directive). The deltas above are sourced from `git log origin/main` between the prior SSOT anchor and current HEAD. A subsequent drift pass should re-fetch the docs and cross-check the descriptions here for any new doc-only changes (e.g. deprecation notices, new integration guides).

## Action items for next full reference re-compile

1. Run `hermes config migrate` on local + M4 + Trinity to land v33 keys before the next reference re-compile.
2. The reference needs a new section on `config.channel_overrides`, `config.extra_headers`, and the persisted `sessions.json` model_override schema.
3. MoA section needs to document the `fanout: user_turn` preset key.
4. Delegation section needs to remove `toolsets` from the documented `delegate_task()` signature.
5. Compression section needs the "always keep a user turn" invariant called out.
6. Skills section needs the stacked-invocation syntax and the 5-skill limit.
7. Gateway section needs the `display.tool_progress: 'log'` mode, the per-channel override resolution order, and the per-session model persistence.
8. Memory section needs the local-upload credential guard.
9. Tools section needs the recursive JSON-string tool-arg normalization (`8a04b516a`).
10. MCP section needs the first-class MCP tab as a sibling to the existing MCP server config section.
11. Cron section needs the profile-secret-scope note.
12. Skills section needs the `/compact` alias and `--preview`/`--dry-run` flags.
13. Knowledge-cutoff gap: re-fetch official docs on next pass.

## Verification commands used

```bash
hermes --version                       # v0.18.0 (2026.7.1) — local 61e4a5f9, upstream 10423297
hermes config path                     # /Users/m5mbp/.hermes/profiles/ss/config.yaml
hermes config check                    # clean (v32→33 update available)
git -C ~/.hermes/hermes-agent log --oneline 7cfa2fa1..origin/main | wc -l   # 1098
git -C ~/.hermes/hermes-agent rev-parse origin/main                         # 5b04a024a
diff -q <local> <public-mirror>                                             # identical
```

## Files updated this pass

- `SKILL.md` — version bump 1.3.0 → 1.4.0; `last_drift_review` 2026-06-30 → 2026-07-05; `repo_commit` updated.
- `references/CHANGELOG-2026-07-05.md` — NEW. This file. Captures the deltas above; feeds the next full re-compile.
- `references/Hermes_Architecture.md` — **NOT modified this pass** (1098 commits is too large for a single cron window; changelog is the durable record of what must be merged in).
- `references/approvals-mode-quirks.md` — unchanged (still valid as of 2026-06-28).
- `references/citation-verification.md` — unchanged.
- `references/config-migration-and-provider-routing.md` — unchanged (no operator-facing changes invalidated it).

## Public mirror

- Repo: `https://github.com/shagghiesuperstar/hermes-master-ref-maintained`
- Action: push the new `CHANGELOG-2026-07-05.md` and the patched `SKILL.md` so other Hermes agents can pull the changelog on their next drift pass.
- Prior mirror commit: `563803c` (2026-06-21 sync). Next: new commit "Drift sync 2026-07-05 — v0.18.0 changelog + skill bump" with both files.
