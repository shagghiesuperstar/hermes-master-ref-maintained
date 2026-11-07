# CHANGELOG 2026-11-06: Rebuild against official docs and upstream v0.23.0

Scope: documentation only. Nothing was installed or executed, and no Hermes Agent runtime behavior was tested. Verification consisted of reading the official documentation and the upstream source at the commit below.

## Anchors

| Fact | Value |
|---|---|
| Date | 2026-11-06 |
| Upstream repository | NousResearch/hermes-agent, `main` |
| Upstream commit | `9f23c4311a4c4837078081db4f3cd8deab23c0fe` (observed 2026-11-06) |
| Latest release | [v0.23.0, "The Hearth Release"](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0), published 2026-11-05 |
| Official docs | <https://hermes-agent.nousresearch.com/docs/> (index and full text fetched 2026-11-06; the docs source is `website/docs` at the same commit) |

## What changed

1. **Main reference rebuilt.** [Hermes_Architecture.md](./Hermes_Architecture.md) was re-authored against the official docs and the upstream source at the commit above, including the v0.23.0 release notes. This is a full rebuild, not a partial patch. See that file for the content; this entry does not restate it.
2. **Privacy scrub of tracked content.** Tracked Markdown files were reviewed and generalized so that no operator-, host-, account-, or machine-specific details remain. Examples now use placeholders. (This entry deliberately describes the work generically and records no private values.)
3. **Changelog date correction and renames.** Two historical notes carried dates later than 2026-11-06, which is impossible for a review written on or before that day. They were renamed to dates consistent with the review order:
   - `CHANGELOG-2026-11-10.md` renamed to [CHANGELOG-2026-10-18.md](./CHANGELOG-2026-10-18.md)
   - `CHANGELOG-2026-11-24.md` renamed to [CHANGELOG-2026-10-25.md](./CHANGELOG-2026-10-25.md)

   Their bodies are historical snapshots and were not re-verified here. Links to the old names were updated in [SKILL.md](./SKILL.md) and [README.md](./README.md).
4. **SKILL.md updated.** `repo_commit` is now the full upstream SHA above and `last_drift_review` is 2026-11-06. The version was raised to 2.0.0 for the rebuild. All links were converted from the old `references/...` form to flat relative links (`./File.md`), a link to a file that does not exist in the repository was removed rather than recreated, stale "untouched SSOT" and "N weeks stale" assertions and hard-coded per-changelog staleness counts were removed, and archived changelog entries are now explicitly labelled historical snapshots. The author field is a generic maintainer string.
5. **New README.** [README.md](./README.md) states the purpose, reading order, source authority (official docs and upstream authoritative, community material secondary and labelled), last-sync anchor, update cadence, contribution rules, and privacy gate, and lists the changelog index.

## Source URLs

- Docs: <https://hermes-agent.nousresearch.com/docs/>
- Release: <https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0>
- Commit: <https://github.com/NousResearch/hermes-agent/commit/9f23c4311a4c4837078081db4f3cd8deab23c0fe>
- SHA-pinned files (existence confirmed in a local read-only checkout at that commit):
  - [`run_agent.py`](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/run_agent.py)
  - [`toolsets.py`](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/toolsets.py)
  - [`hermes_cli/config.py`](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/hermes_cli/config.py)
  - [`website/docs/developer-guide/architecture.md`](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/developer-guide/architecture.md) (published at <https://hermes-agent.nousresearch.com/docs/developer-guide/architecture>)
  - [`website/docs/user-guide/configuration.md`](https://github.com/NousResearch/hermes-agent/blob/9f23c4311a4c4837078081db4f3cd8deab23c0fe/website/docs/user-guide/configuration.md) (published at <https://hermes-agent.nousresearch.com/docs/user-guide/configuration>)

## Limitations

- Docs-only work: no Hermes Agent process was run, so command output, defaults at runtime, and behavior under real configurations were not exercised.
- Claims were checked by reading the docs and source at the SHA above; later upstream commits are not reflected.
- Community sources (merged PRs, issues, hub listings) were not treated as authoritative. [Hermes_Architecture.md](./Hermes_Architecture.md) adds a labelled "Change context & community sources (secondary)" section with eight merged PRs (2026-10-14 to 2026-11-06), two issues and the official Skills Hub page; the third-party `hermeshub.xyz` / `hermeshub.com` domains were unavailable (HTTP 402 / unresolvable redirect) on 2026-11-06 and are not cited.
- Archived changelogs were not re-verified; they remain historical snapshots.

## Action items for the next review

- Compare `9f23c4311a4c4837078081db4f3cd8deab23c0fe..origin/main` using [drift-review-runbook.md](./drift-review-runbook.md).
- Re-check any release newer than v0.23.0.
- Bump the anchor fields in [SKILL.md](./SKILL.md) and the "Last sync" table in [README.md](./README.md).
