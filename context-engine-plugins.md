# Context engine plugins: selection, migration, tuning

Applies to any `context.engine` change (built-in `compressor` <-> a plugin engine such as LCM-X, or
between plugin generations). Source of truth: `agent/agent_init.py::_select_context_engine`,
`plugins/context_engine/__init__.py`, and the plugin's own `config.py` / operator guide at the exact tag.

## Research before recommending (do this first, every time)

1. Clone the plugin into scratch and check out the exact release tag. Record the full SHA; tags move, and
   catalog pins lag.
2. Read its operator guide sections: install, activate, migrate, rollback, threshold ownership, and
   large-window tuning. Then read `config.py` to see which YAML keys it actually honors. Docs and the
   real config surface often differ.
3. Compare the Hermes repo's `plugin-catalog/<name>.yaml` pin on local vs `upstream/main`. The locally
   installed catalog can trail by a release. Install by git at the verified SHA when it does.
4. Triage the plugin's open issues by label (P0/P1, data-integrity, security) through the API, not by
   skimming titles. Also search the Hermes repo for open issues and PRs naming the plugin.
5. Probe the real store on a COPY. Snapshot with the SQLite online backup API (`src.backup(dst)`, safe
   under WAL). Open the copy with the new version's store classes, then assert that row counts are
   unchanged and `pragma quick_check` returns ok. Never open the live DB with a new build first.
6. Check dependency claims against PyPI JSON (wheel tags for the host Python), not against issue
   comments. Then do one real enable on a canary. The Hermes PM resolver adds its own constraints
   (an `exclude-newer` window, cross-env markers), so a wheel existing does not guarantee admission.

## Loader facts that drive the procedure

- **Resolution order:** bundled `plugins/context_engine/<engine-name>/` FIRST, then the general plugin
  system. A stale dir in the bundled path whose name equals the configured engine name shadows any
  installed plugin. Legacy alias names therefore keep loading the old copy.
- **Activation needs both keys:** the plugin's manifest name in `plugins.enabled` AND its runtime
  engine name in `context.engine`. These two names often differ (e.g. `hermes-lcm-x` / `lcm-x`).
- **The engine name is bound when the plugin loads.** The host matches `candidate.name ==
  context.engine` exactly. Editing config under a live process leaves the old binding in place, and the
  host silently falls back to the lossy compressor (log: `Context engine '<x>' not found`). After any
  engine edit, reload plugins (gateway control socket `reload_gateway_plugins(home)`) or restart.
  Then verify the log line `registered context engine: <x>`.
- **Plugins and the DB are profile-scoped:** `$HERMES_HOME/plugins/` and `$HERMES_HOME/lcm.db`. Every
  profile needs its own install or symlink. Root `~/.hermes` and `profiles/default` are different
  homes; inventory both.
- **The install target must be durable.** Point plugin symlinks at a stable checkout
  (e.g. `~/.hermes/plugins-src/<name>`), never into a kanban scratch workspace. Scratch is pruned when
  the card completes, and every profile would silently fall back to the compressor.
- **Retire a shared legacy copy last.** The gateway resolves the bundled dir for every new agent of
  every profile that still names it. Move it (never delete) only after the last consumer's config is
  switched, inside the final stopped window before restart.

## Config surface (LCM-X shape; verify per plugin)

- Top-level `lcm:` YAML honors only a short allowlist (`context_threshold`, `model_thresholds`, the
  reasoning-effort keys, and the embedding prefixes). Everything else is an `LCM_*` var in the profile
  `.env`. A nested `context.lcm:` block is ignored. `lcm_status.ignored_config_yaml_lcm_keys` lists
  dead keys.
- Threshold precedence: `LCM_CONTEXT_THRESHOLD` -> `lcm.context_threshold` -> `compression.threshold`
  floored to the plugin default. Built-in `compression.*` ratios (target_ratio, protect_last_n) are
  not read by the plugin.
- `model_thresholds` keys are case-sensitive substrings of the active model name. The longest match
  wins, and the match beats the base threshold.
- Edit config surgically (text edit + `yaml.safe_load` assert + readback diff). A full YAML round-trip
  re-indents lists and produces a huge diff that no reviewer can verify. Watch for flow-style
  `disabled: []` siblings when appending to `plugins.enabled`, and preserve the file's trailing-newline
  state.

## Picking the compaction threshold

1. Measure the real per-session token distribution from the existing store (p50/p95/p98/p99/max).
   Count how many sessions exceed candidate triggers.
2. Resolve every route's window with Hermes's own `agent.model_metadata.get_model_context_length`
   (run in the Hermes venv). This covers primary, fallbacks, and custom-provider entries. Custom
   bridges can serve the same model name with a smaller window.
3. Choose one absolute trigger for large-window routes, above roughly p98, so ordinary sessions never
   compact. Never compacting means a warm prompt cache and no summariser cost or latency. Implement it as
   per-model fractions (`window x fraction = trigger`), not one global fraction: a single ratio lands
   wrong on mixed 131k/200k/500k/1M routes.
4. Keep the plugin default fraction for routes of about 272k or less.
5. Re-check each profile's actual route. A model key that is correct on a 1M route can give a far too
   small trigger on a 200k bridge serving the same model name. Drop that key for that profile only.
6. Verify after cutover: `lcm_status` `context_threshold_source = model_thresholds:<key>` and the
   expected `threshold_tokens`.

## Summary route must be proven live

- An open summary-route circuit pauses compaction silently. Prove `auxiliary.compression` with one
  real call per profile, not by reading config. Also check the credit balance for metered routes.
- Provider ids that share a model family are separate credential-pool keys with no cross-fallback
  (e.g. an API-key provider vs its `-oauth` sibling). A plan-exhausted API key does not fall through
  to a live OAuth login. Pin the provider id that probed live.

## Rollout shape

Per-profile backups (config, `.env`, plus a DB backup with row counts) -> canaries on low-traffic
profiles -> an independent judge on the exact diffs -> production homes last, in one stopped window.
The production gateway restart is run by the operator from an external shell; Hermes refuses a
restart issued from inside a gateway session.
