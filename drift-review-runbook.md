# Hermes Drift Review Runbook

Class-level reference for the weekly Hermes architecture drift review. Companion to `SKILL.md` (which owns the protocol); this file owns the mechanical execution details that repeat every Monday.

## 1. Anchor probes (always first)

Run these four commands up front. The output goes in the changelog header verbatim.

```bash
# 1. Local Hermes version (semver + commit SHA)
hermes --version

# 2. Local repo commit (if Hermes is installed at ~/.hermes/hermes-agent)
git -C ~/.hermes/hermes-agent rev-parse HEAD

# 3. Upstream HEAD — fetch first, then read
git -C ~/.hermes/hermes-agent fetch origin main
git -C ~/.hermes/hermes-agent rev-parse origin/main

# 4. Local-behind-upstream count (not the same as drift count!)
git -C ~/.hermes/hermes-agent rev-list --count HEAD..origin/main
```

Then compute the actual drift:

```bash
PRIOR=$(grep -E '^repo_commit:' <skill-dir>/SKILL.md \
        | head -1 | grep -oE '[0-9a-f]{7,}' | head -1)
git -C ~/.hermes/hermes-agent log --oneline ${PRIOR}..origin/main | wc -l
```

If `PRIOR` is the same as `origin/main`, drift is zero — no changelog needed (just bump `last_drift_review`).

### 1b. Body-provenance probe (MANDATORY — added 2026-11-01)

**The anchor in `repo_commit` is a REVIEW anchor, not a BODY-sync anchor.** They are different dates and drift apart. Reporting "the reference is N days stale" from the review anchor understates the SSOT body's real age — this caused two consecutive passes (2026-10-18 and 2026-10-25 [dates corrected 2026-11-06; the files were originally misnamed 2026-11-10 and 2026-11-24]) to report a stale reference that was actually ~2 months older than claimed.

Read the body's own provenance header, and report staleness from **that** date:

```bash
# The SSOT body's own last-sync date (authoritative for "how stale is the reference?")
grep -E '^### (Compiled|Updated):' \
  <skill-dir>/Hermes_Architecture.md

# Commits between the body's last sync and upstream HEAD
BODY_SYNC=$(git -C ~/.hermes/hermes-agent log --format='%H' origin/main \
  --before="$(date -j -f '%Y-%m-%d %H:%M:%S' '2026-06-21 12:09:00' '+%Y-%m-%d %H:%M:%S')" -1)
git -C ~/.hermes/hermes-agent rev-list --count ${BODY_SYNC}..origin/main
```

Report **both** numbers in the changelog, labelled separately:
- *review-anchor age* — days since the `repo_commit` anchor
- *body age* — days since the body's own last sync (this is what "the reference is stale" means)

### 1c. Schema-version probe (MANDATORY — added 2026-11-01)

**Never infer the schema version from commit-message greps.** Two passes in a row reported "v31 unchanged" while upstream was already at **v32** — the bump commit (`1eda335ae0`, 2026-10-17) was inside their window but its message said nothing about a schema bump. Read the constant directly:

```bash
# SCHEMA_VERSION and FTS_STORAGE_VERSION at upstream HEAD — the only source of truth
git -C ~/.hermes/hermes-agent show origin/main:hermes_state_common.py \
  | grep -E '^(SCHEMA_VERSION|FTS_STORAGE_VERSION) = '

# Same, at any historical anchor, for a before/after comparison
git -C ~/.hermes/hermes-agent show <anchor>:hermes_state_common.py \
  | grep -E '^(SCHEMA_VERSION|FTS_STORAGE_VERSION) = '
```

If a bump is suspected but invisible in commit messages, diff the constant across the window instead of trusting the message text.

### 1d. Schema/constant drift in the SSOT body

Grep the SSOT body for the constants it claims and diff them against upstream:

```bash
grep -nE 'SCHEMA_VERSION|FTS_STORAGE_VERSION|schema_version' \
  <skill-dir>/Hermes_Architecture.md
```

A body that names no version number is not wrong — but a changelog that *claims* a version must match the constant, not the commit messages.

## 2. Commit-type concentration probe

The fastest way to see where activity concentrates in a window:

```bash
PRIOR=<prior_anchor>
git log --oneline ${PRIOR}..origin/main --pretty="%s" \
  | grep -oE '^[a-z]+\([^)]+\)' \
  | sort | uniq -c | sort -rn | head -30
```

This gives you a table like:

```
   14 fix(gateway)
   11 fix(mcp)
    6 fix(auxiliary)
    5 fix(cron)
    5 fix(agent)
    5 chore(release)
    ...
```

Use the top entries to triage which architecture sections need the deepest look.

## 3. Feat-only inventory

For the "what's actually new" pass, isolate `feat:` commits:

```bash
git log --oneline ${PRIOR}..origin/main --pretty="%s" | grep -E "^feat"
```

Spot-check the top 3-5 with `git show --stat <sha>` before claiming anything in the changelog. Common false-positive shapes to watch for:

- `Revert "feat(X)"` — should land in the "Reverts" section, not the feature inventory.
- `feat(X): salvaged #NNNN` — first-time-into-main port; usually a single-line behavior addition, not a system change.
- `feat: add N (alias M)` — CLI alias; usually not worth a full architecture callout unless the new command is material.

## 4. Architecture area buckets (canonical, 1:1 with Hermes_Architecture.md sections)

When grouping deltas in the changelog, use these exact section names. The next re-compile reads the changelog and merges by section — so the names have to match.

| Area | Subsumes |
|---|---|
| Configuration & provider routing | `config.yaml`, `.env`, provider model, fallback chains |
| Schema versions & state.db migrations | state.db version bumps, new tables, healing-on-startup |
| Approvals & security | approvals.*, dangerous-command guards, webhook body limits, OAuth scope, env sanitization |
| Skills | SKILL.md format, disabled-gate, preload, curator, skill_view behavior |
| MCP | server lifecycle, OAuth, parked-revival, preflight, connect_timeout |
| Sessions & state | session storage, prune/filter, active-row invariants, message INSERTs |
| Compression / context | TTFT, autoraise, native compaction paths, summary compressor |
| CLI | `hermes` subcommands, new flags, removed flags, behavior changes |
| Cron | run-claim TTL, scheduling, past-timestamp rejection, one-shot durable claims |
| Gateway | routing index, pairing, session-key guards, multiplex, /stop sentinel, last-resolved-model cache |
| Platform adapters — Discord | streaming body, REST reads, component labels, edit-rate-limit dedup |
| Platform adapters — Telegram | bot-token redaction, polling bounds, typing-failure cooldown |
| Platform adapters — WhatsApp / Feishu / MS Graph / SMS | webhook body limits, client_max_size |
| Computer-use | cua-driver, env sanitization across spawn sites |
| Providers / Auth | OAuth providers, Z.AI, Codex quota, Nous, Anthropic, OpenRouter routing |
| Web / search providers | Firecrawl, exa, parallel, tavily, brave-free, dashboard model picker |
| Docker / Photon | docker_network, sidecar dep self-heal, pairing-dir ownership |
| Desktop / TUI | mermaid preview, update markers, HERMES_DESKTOP_CWD, model picker |
| Secrets | SecretSource plugin system, Bitwarden, 1Password |
| Reverts | always call out by PR number |
| Salvaged PRs | cherry-picks preserving authorship — see `AUTHOR_MAP` |

If a delta doesn't fit any of these buckets, add a new bucket — don't shoehorn it. The next re-compile will inherit whatever buckets exist.

## 5. Spot-verification commands

Before quoting a commit in the changelog:

```bash
# Minimum: confirm the file count and what changed
git show --stat <sha>

# Better: confirm the body matches the message
git show <sha> | head -40

# Best: confirm the public API surface (look for class/function signatures)
git show <sha> -- '*.py' | grep -E "^(class|def|async def) "
```

For the highest-impact commits (typically top 5-10 in a 200-commit window), do the "better" check. For everything else, the stat check is enough.

## 6. Frontmatter bump (exact 4 edits)

After writing the changelog, edit `SKILL.md` frontmatter:

```yaml
version: <semver-minor-bump>          # e.g. 1.4.0 → 1.4.1
last_drift_review: <YYYY-MM-DD>       # today
repo_commit: <prior_anchor> (last fully-synced anchor; upstream now <head>, +<n> commits pending; SSOT untouched per skill protocol)
```

Then under `## Targeted quirks`, add a pointer to the new changelog with the canonical warning:

```
- [CHANGELOG-<YYYY-MM-DD>.md](CHANGELOG-<YYYY-MM-DD>.md) — <one-line summary of the window>. **Read this before assuming any section of the main reference is current.**
```

If a `Reverts` section is non-empty in the changelog, also bump the skill description's trigger list if a reverted feature was in it.

## 7. Verification (always end with this)

```bash
# Confirm the changelog is readable and self-references correctly
ls -la <skill-dir>/CHANGELOG-<YYYY-MM-DD>.md
head -20 <skill-dir>/CHANGELOG-<YYYY-MM-DD>.md

# Confirm the SKILL.md is well-formed (no broken frontmatter)
python3 -c "import re; print(re.match(r'^---\n.*?\n---', open('<skill-dir>/SKILL.md').read()).group())"

# Confirm the skill still loads via hermes (if available)
hermes skills list | grep hermes-architecture-reference
```

## 8. Public mirror sync (when the operator authorizes)

**The mirror stores files FLAT AT THE REPO ROOT** — `CHANGELOG-<date>.md`, `Hermes_Architecture.md`, `SKILL.md` — *not* under any subdirectory. Copying into a subdirectory lands the file in a path that does not exist in the mirror. This is the single most-repeated mistake in past passes.

Use a single canonical clone of the mirror (`<mirror-clone>`). If you also find a second, stale clone with uncommitted edits or untracked files, do not reuse it; re-clone rather than building on stale state.

```bash
cd <mirror-clone>
git fetch origin main

# Reconcile a diverged local branch before copying anything in
git log --oneline HEAD..origin/main          # commits on remote we lack
git log --oneline origin/main..HEAD          # commits we lack on remote
git merge origin/main                        # 0 conflicts in practice; resolve if not

# Copy the new changelog to the repo ROOT (flat layout), not a subdirectory
cp <skill-dir>/CHANGELOG-<YYYY-MM-DD>.md .

# Also sync SKILL.md when the frontmatter or the Targeted-quirks list changed
cp <skill-dir>/SKILL.md .

git add -A
git commit -m "Weekly drift sync: <YYYY-MM-DD> upstream changes"
git push origin main
```

**Backfill gaps.** The mirror is frequently *behind* the local skill by more than one pass — earlier changelogs and whole reference files (e.g. the 2026-10-14 and 2026-10-18 changelogs, `approvals-mode-quirks.md`, `citation-verification.md`, `config-migration-and-provider-routing.md`, `drift-review-runbook.md`) were never pushed. After the new-file push, diff the file lists and copy anything missing (the local skill directory may itself be nested differently; compare file names only):

```bash
# Files in the local skill that are absent from the mirror
diff <(ls <skill-dir>/) \
     <(ls <mirror-clone>/)
```

Push the backfilled files too. A mirror that is missing reference files is worse than one that is merely a week stale — other agents pull from it.

**Verify the push landed** (never assume):

```bash
git ls-remote origin HEAD
git ls-tree -r --name-only origin/main | grep CHANGELOG-<YYYY-MM-DD>
```

Skip this step if the operator hasn't authorized the public mirror — it's a distribution concern, not a local SSOT concern.

## 9. Common failure modes (verified 2026-07-12)

| Symptom | Root cause | Fix |
|---|---|---|
| `git fetch origin main` fails with "could not resolve host" | Network blip or VPN off | Retry once; if still failing, document in changelog "Known limitations" and proceed with last-fetched origin/main |
| `git rev-parse origin/main` returns nothing | `git fetch` was never run in this shell | Run `git fetch origin main` first; never assume the prior session's fetch state |
| Local is on a feature branch (`fix/cron-update-pin-snapshots`) | Install uses long-lived working branches | Don't merge to main — just use the branch's HEAD as the local anchor. The drift count is `<branch_HEAD>..origin/main` |
| `hermes --version` returns a different commit than the git HEAD | Hermes is installed from a pip wheel, not a checkout | Use `pip show hermes-agent \| grep Location` to find the install path; the install path's `__init__.py` has the version constant. Or just trust `hermes --version`. |
| `last_drift_review` already updated today | Two cron runs in one day (timezone / DST) | OK — second run is a no-op as long as the changelog filename is unique. Never overwrite the prior file. |
| Changelog filename is dated a month *after* the window it describes | The pass named the file from a stale "next scheduled run" date instead of the actual run date | Name the file for the **actual run date**. Verify with `date` at the start of the pass, and record the window's upstream anchor commit date in the changelog header so the chain can be ordered by anchor, not filename. (Two passes did this on 2026-10-18 and 2026-10-25.) |
| Changelog claims "no schema bump" but upstream moved | The bump commit's message doesn't mention "schema", so a message grep misses it | Read the constant directly — `git show origin/main:hermes_state_common.py \| grep -E '^(SCHEMA\|FTS_STORAGE_VERSION) = '`. Never infer a version from commit messages. (Two passes reported "v31 unchanged" while upstream was v32.) |
| Reported staleness is ~2 months shorter than reality | Staleness measured from the `repo_commit` review anchor instead of the body's own last-sync date | Run the body-provenance probe (§1b) and report body age and review-anchor age separately. |
| Mirror push "succeeded" but the file isn't on GitHub | File was copied into a subdirectory (which doesn't exist in the mirror) instead of the repo root | Mirror layout is FLAT. Copy to `.`, then verify with `git ls-tree -r --name-only origin/main`. |