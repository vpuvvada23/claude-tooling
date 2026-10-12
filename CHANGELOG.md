# Changelog

The plugin was named `dev-pipeline` before 0.4.0.

## bracket 0.4.0
- Renamed the plugin from `dev-pipeline` to `bracket` (display name "Bracket Dev Kit"). Commands are now `/bracket:full`, `/bracket:plan`, and `/bracket:build-verify`. Run folders in `.claude/pipeline-runs/` are unchanged.
- Added a plugin-folder README (with a disclosure of what the agents run, write, and send) and LICENSE, as the plugin directory requires.

## dev-pipeline 0.3.2
- `run-stats.md` gets an explanatory header, a section per command, and aligned tables.
- Dropped the Tool uses column and the "(requested)" labels; the header explains that model and effort are what was requested.
- Outcome uses a fixed set of values per role, with no extra commentary.

## dev-pipeline 0.3.1
- The pre-flight table shows only Role, Model, and Effort for the roles the current command will dispatch.
- `run-config.md` is now a read-only, append-only record with an explanatory header and aligned tables. No command reads settings back from it, and each command chooses its own settings at pre-flight.

## dev-pipeline 0.3.0
- Agents use model family aliases (`opus`, `sonnet`) instead of pinned IDs, so they track new releases.
- Defaults moved to `pipeline-config.md`: Opus for planners, judge (xhigh), spike pass, and verifier; Sonnet for the builder.
- Per-run `--agents role=model[:effort]` overrides, with a pre-flight confirmation saved to `run-config.md`.
- Per-dispatch run stats (model, effort, tokens, tool uses, duration, outcome) in `run-stats.md`, printed at the end of every run.

## dev-pipeline 0.2.0
- Planners list external/platform assumptions with verified status.
- The judge promotes unverified, load-bearing assumptions to a required `## Blocking Unknowns` section, and estimates are conditional on them.
- Spike gate: the builder runs each Blocking Unknown before the full build, and the run halts unless all are confirmed with evidence.
- The verifier must try automation before marking checks manual-only, and reports PASS / UNCONFIRMED / FAIL.

## dev-pipeline 0.1.x
- Initial plan-speed / plan-extensible / plan-judge / builder / verifier pipeline and skills.
