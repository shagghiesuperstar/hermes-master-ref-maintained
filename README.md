# Hermes Architecture Reference

An offline, documentation-only reference for the architecture and internals of [Hermes Agent](https://github.com/NousResearch/hermes-agent). It is written for people and agents who troubleshoot, design, implement, optimize, or review Hermes Agent setups and want a single searchable document plus the procedures used to keep it current.

This repository contains Markdown only. There is no runtime code to install or run.

## Source authority

1. **Authoritative:** the official documentation at <https://hermes-agent.nousresearch.com/docs/> and the upstream repository <https://github.com/NousResearch/hermes-agent> (source, release notes, and the docs source under `website/docs`).
2. **Secondary (community):** upstream issues and pull requests, and the Skills Hub listing on the official docs site (<https://hermes-agent.nousresearch.com/docs/skills>, official host, community-contributed entries), and third-party "HermesHub" sites (the `hermeshub.xyz` and `hermeshub.com` domains were unavailable or non-resolving when probed on 2026-11-06, so none is cited). These may add context, but anything taken from them must be labelled as community/secondary and must not be presented as official behavior.
3. **This repository** is derived from the two authoritative sources. If it disagrees with the live docs or the upstream source, the live sources win; please fix this repository.

## Last sync

| Item | Value |
|---|---|
| Last synced | 2026-11-06 |
| Upstream repository | NousResearch/hermes-agent, `main` |
| Upstream commit | `9f23c4311a4c4837078081db4f3cd8deab23c0fe` ([tree at that commit](https://github.com/NousResearch/hermes-agent/tree/9f23c4311a4c4837078081db4f3cd8deab23c0fe)) |
| Latest release at sync | [v0.23.0, "The Hearth Release"](https://github.com/NousResearch/hermes-agent/releases/tag/v0.23.0), published 2026-11-05 |

The same anchor is recorded in the `repo_commit` and `last_drift_review` fields of [SKILL.md](./SKILL.md). Nothing in this repository was verified by running Hermes Agent; checks were made by reading the official docs and the upstream source at the commit above.

## Where to start

| You want | Read |
|---|---|
| To use this as an agent skill, or know when and how to load it | [SKILL.md](./SKILL.md) |
| Architecture, config, tools, memory, MCP, gateway, cron, plugins, security | [Hermes_Architecture.md](./Hermes_Architecture.md) (large; search or jump by table of contents rather than reading end to end) |
| To run or repeat a drift review | [drift-review-runbook.md](./drift-review-runbook.md), with the protocol in [SKILL.md](./SKILL.md) |
| Task-specific playbooks | [approvals-mode-quirks.md](./approvals-mode-quirks.md), [citation-verification.md](./citation-verification.md), [config-migration-and-provider-routing.md](./config-migration-and-provider-routing.md) |
| What changed and when | The dated changelogs below, newest first |

Suggested order: skim SKILL.md, look up the topic in Hermes_Architecture.md, then confirm anything that affects commands or config keys in the official docs before acting.

## Changelog index

Changelogs are **historical snapshots**: each records what one review pass observed on its own date. They are an audit trail and a source of commit SHAs, not a description of current behavior. Numbers inside them (commit counts, schema versions, "behind upstream", staleness) are valid only for that date. Use Hermes_Architecture.md and the official sources for current facts.

- [CHANGELOG-2026-11-06.md](./CHANGELOG-2026-11-06.md) (rebuild of the reference at the sync above)
- [CHANGELOG-2026-11-01.md](./CHANGELOG-2026-11-01.md)
- [CHANGELOG-2026-10-25.md](./CHANGELOG-2026-10-25.md) (renamed, see note)
- [CHANGELOG-2026-10-18.md](./CHANGELOG-2026-10-18.md) (renamed, see note)
- [CHANGELOG-2026-10-14.md](./CHANGELOG-2026-10-14.md)
- [CHANGELOG-2026-10-04.md](./CHANGELOG-2026-10-04.md)
- [CHANGELOG-2026-09-27.md](./CHANGELOG-2026-09-27.md)
- [CHANGELOG-2026-09-20.md](./CHANGELOG-2026-09-20.md)
- [CHANGELOG-2026-09-13.md](./CHANGELOG-2026-09-13.md)
- [CHANGELOG-2026-09-06.md](./CHANGELOG-2026-09-06.md)
- [CHANGELOG-2026-08-30.md](./CHANGELOG-2026-08-30.md)
- [CHANGELOG-2026-08-23.md](./CHANGELOG-2026-08-23.md)
- [CHANGELOG-2026-08-16.md](./CHANGELOG-2026-08-16.md)
- [CHANGELOG-2026-08-09.md](./CHANGELOG-2026-08-09.md)
- [CHANGELOG-2026-07-26.md](./CHANGELOG-2026-07-26.md)
- [CHANGELOG-2026-07-19.md](./CHANGELOG-2026-07-19.md)
- [CHANGELOG-2026-07-12.md](./CHANGELOG-2026-07-12.md)
- [CHANGELOG-2026-07-05.md](./CHANGELOG-2026-07-05.md)

Note on the renamed notes: two historical notes were originally mis-dated: their file names carried dates later than 2026-11-06, which cannot be correct for notes written on or before that day. On 2026-11-06 they were corrected and renamed `CHANGELOG-2026-11-10.md` to `CHANGELOG-2026-10-18.md` and `CHANGELOG-2026-11-24.md` to `CHANGELOG-2026-10-25.md`. Links to the old names should be updated.

## Update cadence

- Review after every upstream release, and at least weekly.
- Each review follows the protocol in [SKILL.md](./SKILL.md) and the commands in [drift-review-runbook.md](./drift-review-runbook.md): record anchors, list the delta, bucket it by architecture area, spot-verify high-impact commits, then either write a dated changelog or rebuild the main reference.
- After a review, bump `version`, `last_drift_review`, and `repo_commit` in SKILL.md and update the "Last sync" table above.
- Prior changelogs are never overwritten and never dated in the future.

## Contributing

This is a docs-only repository. Contributions should:

- **Verify against authoritative sources.** Check each claim against the official docs or the upstream source before adding it. Do not guess CLI flags, config keys, tool names, or defaults.
- **Cite immutable URLs.** Link upstream files with a commit-SHA URL such as `https://github.com/NousResearch/hermes-agent/blob/<full-40-char-sha>/<path>` rather than a branch name, and name the release or docs page when relevant.
- **Label secondary sources.** Anything from issues, PRs, hub listings, or other community material must be marked as community/secondary.
- **State what was and was not verified.** Reading docs or source is not the same as running the software; say which one you did.
- **Use relative links.** The layout is flat; link sibling files as `./File.md`.
- **Keep private data out.** See below.

### Privacy gate

Examples must be generic. Do not include real hostnames, IP addresses, usernames, personal or home-directory paths, profile or machine rosters, tokens or keys, account identifiers, or business-private details. Use placeholders such as `<profile>`, `<hermes-agent-checkout>`, `example.com`, and `$HERMES_HOME`. Before submitting, search the changed files for anything that identifies a person, machine, or organization and remove it. Never paste secrets, even to illustrate a redaction.

### Validation before submitting

- `git diff --check` is clean.
- Every relative link and `#anchor` resolves to a file or heading that exists.
- The privacy search above finds nothing.

## License

MIT, as declared in the [SKILL.md](./SKILL.md) frontmatter.
