# CHANGELOG 2026-11-08: post-rebuild delta — schema v33, CJK FTS removal, transcript unification
> Historical snapshot, not current truth. See [Hermes_Architecture.md](Hermes_Architecture.md) for current state.
>
Scope: documentation only. Nothing was installed or executed, and no Hermes Agent runtime behavior was tested. Verification consisted of reading the upstream source and the official documentation at the commits named below.

This pass exists because the 2026-11-06 rebuild landed in the **public** repository but never reached the **local** skill. The local `references/Hermes_Architecture.md` was still the 2026-06-21 body (116,827 bytes) while the public copy had been rebuilt to 526,847 bytes. That gap is now closed locally. The delta between the rebuild anchor and current upstream is recorded below.

## Anchors

| Fact | Value |
|---|---|
| Date | 2026-11-08 |
| Local Hermes | v0.23.0 (2026.9.24) · local commit `2bd378ed1b8d6bf2192cb26e0a55ff3091e6bd19` |
| Local install behind upstream | 1,339 commits (normal — the local checkout trails `main`) |
| Local checkout path | <hermes-agent-checkout> |
| Upstream repository | NousResearch/hermes-agent, `main` |
| Upstream HEAD at this pass | `acdc8f1ba65c0231ae8277e846a4d55928913991` (2026-11-08 23:28 +0530) |
| Prior review anchor (rebuild) | `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (observed 2026-11-06) |
| Delta since that anchor | **938 commits** (114 feat, 323 fix, 29 refactor, 1 revert) |
| Cumulative delta since the 2026-06-21 body | 37,608 commits |
| Latest upstream release | [v0.23.0, "The Hearth Release"](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0), published 2026-11-05 (unchanged since the rebuild) |
| Official docs | <https://hermes-agent.nousresearch.com/docs/> (session-storage page re-read 2026-11-08) |

## The finding that mattered: the local SSOT never received the rebuild

The 2026-11-06 pass performed a full rebuild of `Hermes_Architecture.md` and pushed it to the public mirror repository (commit `5326c8545`, "docs: rebuild architecture reference and sanitize public content"). That commit rewrote the main reference by **+4,060 / −2,447 lines**, added `README.md`, renamed two mis-dated changelogs, and sanitized operator-specific content out of the tracked files.

It was never copied down into the local skill. Verified state at the start of this pass:

| Artifact | Local (before this pass) | Public |
|---|---|---|
| `Hermes_Architecture.md` | 116,827 bytes, body dated **2026-06-21** | 526,847 bytes, rebuilt **2026-11-06** |
| `SKILL.md` | v1.20.0, `last_drift_review: 2026-11-01` | v2.0.0, `last_drift_review: 2026-11-06` |
| Changelog filenames | `CHANGELOG-2026-11-10.md`, `CHANGELOG-2026-11-24.md` (impossible dates) | renamed to `CHANGELOG-2026-10-18.md`, `CHANGELOG-2026-10-25.md` |
| `README.md` | absent | present |
| `context-engine-plugins.md` | present (local-only, not in public) | absent |

A separate, unrelated copy at `~/.hermes/skills/devops/hermes-architecture-reference/` is a stale **v1.0.0** snapshot from 2026-06-14 and is **not** the SSOT; the live SSOT is the `ss` profile's copy at <skill-dir>/. The two are separate real directories with distinct inodes, not symlinks.

**Action taken:** the rebuilt reference was adopted into the local SSOT (see "Local changes" below), the two impossible-date changelogs were renamed locally to match the public correction, and the local-only `context-engine-plugins.md` was preserved (it documents the LCM-X / context-engine migration procedure and was never part of the public repo).

## Verified delta since the rebuild anchor `9f23c4311a`

Every item below was spot-verified with `git show --stat <sha>` and a source read at `acdc8f1ba6`. Nothing here was executed.

### Session storage — schema v33 (breaking for readers, data-only for users)

- **`SCHEMA_VERSION` 32 → 33** (`hermes_state_common.py`). The v33 comment reads: *"v33: data-only — re-tag delegate subagent rows that lost `_delegate_from` (see `_run_data_migrations`)."* Bumping commit `f483ce2667` — `fix(delegation): keep _delegate_from on reserved subagent rows (#150198)`. The bump is **data-only**: it records a migration, not a new shape.
- **Two new state modules**, both added in this window:
  - `hermes_state_delegate_children.py` — resolves delegate subagents owned by one session's lineage (recursive, including compression continuations). The walk up stops at a delegate or branch row, so a subagent never claims its parent's children. Its docstring records the reason: the TUI kanban poller uses it to reclaim subscriptions bound to a subagent whose row lacked the `_delegate_from` marker.
  - `hermes_state_stored_reads.py` — the store reads behind `session.transcript`, the `session.list` picker fields, and a delegated child's lookup by `subagent_id`, bound onto `SessionDB` through `SessionMessagesMixin`.
- **New trigger `messages_display_revision_update`** (`93dfe63df3`, `fix(state): a reaction still moves the transcript revision after an older Hermes opens the store`). It fires on `codex_reasoning_items`, `codex_message_items`, `display_kind`, `display_metadata` updates. It has its own name on purpose: an older build's open, which recreates the neighbouring trigger with that build's column list, leaves this one in place.
- **`FTS_STORAGE_VERSION` is unchanged at 3** — verified directly at `acdc8f1ba6`, same as at the rebuild anchor. This is the counter that several earlier passes mis-reported; it is independent of `SCHEMA_VERSION`.
- **The CJK FTS5 tokenizer was removed from core** (`0a0dab0686`, `refactor(state)!: remove the CJK FTS5 tokenizer from core`, 2026-11-07). Deleted: `native/fts5_cjk/`, the tokenizer loader, the `sessions.cjk_fts` key and its `HERMES_CJK_FTS` bridge, `HERMES_FTS5_CJK_SO`, the `messages_fts_cjk` table and triggers, its stale breadcrumb, quarantine, backfill and reset paths, and the `cjk_unicode61` cases in the FTS error classifiers and the repair probe. The commit body records why it was safe: **no installer, package or image ever shipped `libfts5_cjk.so`**, so the loader returned `False` on every install and CJK queries already took the trigram and `LIKE` routes; those routes are unchanged. `hermes_state_fts.py` retains only `_RETIRED_CJK_TRIGGERS` / `_RETIRED_CJK_OBJECTS` — cleanup code that drops the index an **older release** built.
- **`messages_display_metadata` merges instead of overwriting.** New `_MERGE_DISPLAY_META` constant: a row-addressed rewrite must not wipe a key another writer added meanwhile (a reaction).

### Gateway — one stored-transcript read, one profile-resolution verb

- **`session.transcript` is now the one stored-transcript read** (`4d89901ef4`, `feat(gateway): one session.transcript pages, polls and reads delegated children; handoffs are rows`). `session.compactions` and `subagent.transcript` **never ship** — verified by grep: neither string appears in upstream source. Forward polling uses an oldest-order cursor that is never null; it holds the segment it read and the `transcript_cursor_state` of the rows it covered, and while both hold, the next read continues after it (`reset: false`, appends only). An in-place compaction, a rotation, a rewind, or a rewrite or deletion of a covered row answers `reset: true` from the first page.
- **Row kinds carry `row_kind`** (`gateway/session_transcript_rows.py`): `'message'` is a display row exactly as `session.resume`'s `display_messages` projects it; `'compaction'` is a compaction handoff at its position in the transcript, with `role: 'system'`, the summary text a reader sees (without the preamble, end marker, re-quoted user messages and recovery footer written for the model), `compacted_messages`, `timestamp` and the stored `row_id`. The display projection drops a handoff; the transcript read puts each one back as its own row just before the row it was stored on.
- **A delegated child's stored transcript is read under the parent's authority** (`gateway/session_transcript_subagent.py`, ported from #140477). A running child is the one the parent's roster names (`subagent.list` rule, nested children included); a finished one is found by the `subagent_id` stamped on its row, and **only when its ancestry reaches this conversation** through delegate-child or compression-continuation hops. Anything else answers `available: false`.
- **A delegated child stays addressable by `subagent_id`** (`2216779700`); `subagent.list` rows now name its session.
- **A stored session can be read without attaching** (`95973b61d2`) — picker fields, transcript pages and compactions, without claiming the session.
- **A client can name itself on `session.create`** (`d0b49116ec`); the shared gateway client takes its session source and client name (`9b939d4761`).
- **New control-socket verb `resolve-profile`** (`7d608e9647`, `gateway/control_resolve.py`). The control socket now answers which gateway serves a profile home and how to dial it, so a local client no longer has to spawn `hermes gateway attach --json` (a Python start) for every connect. `RESOLVE_VERB = "resolve-profile"`, `RESOLVE_PROTOCOL = 1`. The answer carries **no credential** — the ticket comes from `session-ticket` on the same socket. The module docstring names it part of the **public local control-socket contract** in `website/docs/developer-guide/gateway-internals.md`.
- **Group Chats / Desktop / Webapp** — `75d15b6040`, `21cd0af57d`, `9a7931dc31`, `d477427cae`: a group chat that runs without Desktop, every new group chat runs on a gateway, any served profile can be lent to another gateway's group chat, and bots from other computers can be seated in group chats. Webapp/connections: `44fff7e17b` (one machine-wide saved-gateways store for Desktop and the Webapp), `44901b1db9` (saved connections reached through the forwarding proxy), `965172c498` (relay a signed-in Webapp to other gateways), `7479098100` (reach a saved SSH gateway from Hermes, not Electron), `88f5bc6299` (sign in to an oauth-mode saved gateway from the server), `0a6a2a34b3` (reach saved gateways through the reach module).

### Plugins, memory, vault, CLI, compatibility

- **`redact_sensitive_text` is public plugin API** (`79109635df`). A plugin that sends user text to a model (`ctx.llm`) needs to mask secrets first; `agent.redact.redact_sensitive_text` did that but was a private import, so `hermes plugins validate` warned on it. It is now listed in `hermes_cli/data/plugin_api.json` and the public API page is regenerated.
- **Memory providers get the previous user message on turn start** (`7d96784e6d`). A provider that prepares the next turn's recall at the end of the current one (`queue_prefetch` keyed on the message just answered) has nothing prepared when a process resumes a session: the resumed process starts at turn N without having run turn N-1. A provider configured **not to save messages** has none to fall back on. Follow-up `16f70a2b33` — `previous_message` is the typed ask after a multimodal turn.
- **Real-profile saved passwords are a read-only browser login source** (`75b5168822`, `feat(vault): real-profile saved passwords as a read-only browser: login source`). The real Chrome/Brave profile's saved logins are exposed under the handle prefix `browser:`, enabled exactly when `browser.use_real_profile` is on — the same consent that already snapshots the profile's cookies and Login Data. The entries appear in the in-app Browser bar's key-button picker beside the Hermes vault / Bitwarden / 1Password sources. Gate confirmed in source: `hermes_cli/config_defaults.py` has `"use_real_profile": False`.
- **Moved modules are served under their old names as one module object** (`296308546a`, `hermes_agent/compat/finder.py` + `moves.py`). A `meta_path` finder maps an old dotted name to its current one by the longest prefix that is a key of the moves table, and returns the real module object. An old name and its current name therefore **share state**, `mock.patch` through either name reaches the real code, and `reload()` re-executes the real file. Keys are only added; when a target moves again only its value changes.
- **New CLI command `hermes clipboard image --json`** (`63cdc15eb3`). Third-party clients previously read the clipboard image by running the TUI's private helper script with `PYTHONPATH` pointed at the source tree. The same extractor (`hermes_cli.clipboard.save_clipboard_image`) is now a documented, device-local command with a JSON reply and stable reason codes.
- **One plugin API surface change with a migration note**: the Sep 2026 module decomposition removed old internal import paths, so a plugin importing them fails with `ImportError` — the `compat` finder above is the mitigation path.
- **Catalog growth dominates the window by volume**: 105 `feat(plugin-catalog)` + 96 `feat(catalog)` commits add plugin and model-provider catalog entries (`fitbit-mcp`, `whoop-mcp`, `oura-mcp`, `wally`, `Pareto Inference`, `LLM API`, `hermes-secret-gate`, `solana-token-facts`, `trust-ladder`, `canary-trap`, `citation-guard`, and many more). These are catalog metadata, not core behavior changes; they are listed here only to explain the commit-count concentration.

### Reverts

- `3ffff15d71` — `revert(cli): drop the classic CLI one-turn snapshot change`. One revert in the window. It removes the classic CLI's one-turn snapshot behavior; the gateway-authority path is unaffected.

## Sections of the main reference that changed and how

The rebuilt reference carries one `> Sources:` line per section, pinned to `9f23c4311a`. This pass updated the sections whose facts changed, and re-pinned the affected source links to `acdc8f1ba6`:

| Section | Change applied in this pass |
|---|---|
| 15 — Session Storage | `SCHEMA_VERSION` 32 → 33; v33 added to the migration table (data-only re-tag of delegate subagent rows); the `messages_fts_cjk` index removed from the table listing and the FTS narrative, with the retirement path described; `hermes_state_delegate_children.py` and `hermes_state_stored_reads.py` added to the module table; the `messages_display_revision_update` trigger and `display_metadata` merge recorded; `FTS_STORAGE_VERSION` 3 confirmed |
| 18 — Gateway Internals | `session.transcript` documented as the one stored-transcript read (cursor semantics, `reset` conditions, `row_kind` values); `session.compactions` and `subagent.transcript` recorded as never shipped; delegated-child read under the parent's authority with the ancestry rule; `resolve-profile` control verb (protocol 1, no credential) added; client self-naming on `session.create`; saved-gateways / relayed-Webapp / group-chat-on-gateway features recorded |
| 20 — Plugin System | `redact_sensitive_text` added to the public plugin API surface; the `ImportError` note on removed internal import paths retained and cross-linked to the `compat` finder |
| 13 / 21 — Memory System | `previous_message` on turn start added to the provider contract, with the resume-without-turn-N-1 rationale |
| 25 — Security Model | real-profile saved passwords as a read-only `browser:` vault source, gated on `browser.use_real_profile` |
| 22 — Adding Custom Tools | `hermes clipboard image --json` recorded as a documented device-local CLI surface |
| 26 — File Dependency Chain | `hermes_agent/compat/finder.py` + `moves.py` recorded as the moved-module alias mechanism |
| 27 — Key Constants | `SCHEMA_VERSION` = 33 and `FTS_STORAGE_VERSION` = 3 restated at the new anchor |

Sections 1–14, 16–17, 19, 21, 23–24 and the parts of 25–27 not listed above were re-read against the delta and left unchanged; the 938-commit window did not alter their subject matter.

## Verification limits

- Docs-and-source only. No Hermes command, gateway, plugin, tool or API was run, so nothing here is a claim about observed runtime behavior.
- The docs' session-storage page still says `Current schema version: **31**` (re-read 2026-11-08, upstream `website/docs/developer-guide/session-storage.md` line 312). The source at `acdc8f1ba6` says **33**. The source value is authoritative; the docs table lags by two.
- The public repository's rebuilt reference and this local copy were reconciled by content, not by blind copy: the local `references/` layout (a `references/` subdirectory, which is what Hermes' `skill_view` linked-files resolver expects — `references`, `templates`, `assets`, `scripts`) differs from the public repo's flat layout, and the local-only `context-engine-plugins.md` was preserved.
- The public repo's `SKILL.md` was rewritten to a flat-link, generic-voice v2.0.0 with a `README.md`. The local skill keeps its `references/`-relative links and its ss-specific paths in the operational notes, since those are correct for this machine. Both carry the same anchor facts.
- Community sources were not treated as authoritative for any numbered section. The `hermeshub.xyz` / `hermeshub.com` domains were already recorded as unavailable in the 2026-11-06 changelog and were not re-probed.
- The local Hermes checkout trails upstream by 1,339 commits. That is a stale local install, not un-reviewed drift: the review delta that matters is `9f23c4311a..acdc8f1ba6` = 938 commits, which was reviewed.

## Action items for the next review

- Compare `acdc8f1ba65c0231ae8277e846a4d55928913991..origin/main` using `drift-review-runbook.md`.
- Re-check whether a release newer than v0.23.0 has shipped.
- Watch for upstream catching up the docs' `Current schema version` line (still 31 at the time of this pass).
- Consider whether the `messages_fts_cjk` retirement needs a note in the State DB Recovery page; it is currently only in source.

## Source URLs

- Docs: <https://hermes-agent.nousresearch.com/docs/>
- Release: <https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0>
- Commits: <https://github.com/NousResearch/hermes-agent/commit/acdc8f1ba65c0231ae8277e846a4d55928913991>
- Rebuild anchor: <https://github.com/NousResearch/hermes-agent/commit/9f23c4311a4c4837078081db4f3cd8deab23c0fe>
- Public mirror: the public mirror repository
