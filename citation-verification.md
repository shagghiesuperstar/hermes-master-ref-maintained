# Citation verification — protocol for shared docstrings and external instructions

Lesson learned 2026-06-28.

## The pattern that bit me

The operator sent a documented procedure in a chat message:

```bash
hermes config set approvals.mode off --profile <profile> && \
hermes config get approvals.mode --profile <profile>   # should print: off
```

I almost executed the docstring verbatim. Both commands fail on the live
Hermes build:

- `hermes config get` — subcommand does **not** exist. The actual read
  subcommand is `hermes config show` (per `hermes config --help`).
- `hermes config set approvals.mode off` — succeeds but writes the
  **boolean `False`**, not the string `'off'`. Hermes `config set` parses
  its VAL argument through YAML 1.1 truthiness, so `off`/`no`/`false` all
  coerce to boolean `False`. The "✓ Set approvals.mode = False"
  confirmation string lies about the schema. The docs explicitly require
  the **string** `'off'`.

## Rule

**Whenever the user (or any message) hands you a multi-line procedure that cites an external system — a docstring, a Stack Overflow answer, a `README.md`, a skill's own reference files — replicate the verification before executing any step:**

1. **Source of truth for Hermes:** `curl -fsSL https://hermes-agent.nousresearch.com/docs/` (the rendered Docusaurus site). For one-page citations, the LLM-friendly `llms-full-*.txt` may be linked from the index HTML; fetch both `user-guide/configuration` and `reference/cli-commands` for CLI surface.
2. **Cross-check against the local Hermes build.** `hermes --help`, `hermes <subcommand> --help`, and `hermes config --help` are the fastest way to confirm what subcommands actually exist.
3. **For config key claims**, grep the YAML schema and probe with `hermes config show | grep <key>` or `python3 -c "import yaml; ..."`. The `config show` output is sometimes incomplete — don't trust it as a full mirror.
4. **Fail loud.** If the docstring's command doesn't match the live build, stop and report. Don't patch around it with a "best guess" repair.

## Don't

- ❌ Execute a docstring because the operator said "follow these steps." They will not want you burning time on a broken invocation.
- ❌ Quote the docstring back as proof it worked. Cite what the live build actually returned.
- ❌ Patch the failing command into a working invocation without telling the operator the docstring was wrong. They sent the wrong procedure — that fact itself is the report.

## Cite-able docs already loaded for this class of work

The `hermes-agent` skill (bundled, read-only) lists canonical pitfalls
like:

> **Pitfall — verify before answering confidently:** When the user asks
> an architecture or configuration question and responds with "are you
> sure about all that...?" — they're signaling I may be reciting theory
> without verifying. ALWAYS check the actual state (file contents,
> process lists, config values) before answering with confidence rather
> than guessing from memory.

This is the same class of lesson — extend it to: **also verify
user-supplied docstrings before executing**.
