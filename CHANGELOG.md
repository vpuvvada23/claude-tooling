# Changelog

The plugin was named `dev-pipeline` before 0.4.0.

## bracket 0.4.5
- Bracket asks where the run is headed when the idea doesn't make it clear: a prototype, a growing project (and how you expect it to grow), or not sure. The answer is saved at the top of `idea.md`.
- The judge's bias follows that direction. Prototypes favor speed, except for hard-to-undo choices like the data model, public interfaces, and auth. Growing projects favor extensibility only along the expansion directions you named. "Not sure" keeps the previous extensibility default.
- `plan-extensible` targets the expansion directions you named and calls out hard-to-undo choices.
- The judge weighs speed's advantage by scope (files, steps, new abstractions) instead of effort or time.

## bracket 0.4.4
- The judge no longer writes time or effort estimates, which were far off for agent work. It marks steps that need your hands-on involvement (like testing on a physical device) instead. `run-stats.md` records the real cost after a run.

## bracket 0.4.3
- Commands no longer print the full stats table when they finish. They print a one-line run cost total and point to `run-stats.md` for the breakdown.
- `run-stats.md` rows are agent dispatches only, named by their config role (the spike pass is `spike`). Steps run by hand, like a device test you do yourself, get no row.

## bracket 0.4.2
- Run folders moved from `.claude/pipeline-runs/` to `.claude/bracket-runs/`. Commands offer to move an existing run's folder from the old location when you name it. Update `.gitignore` to match.
- Renamed `pipeline-config.md` to `bracket-config.md`, and dropped the remaining "pipeline" naming from skill and agent descriptions and the run-stats header.

## bracket 0.4.1
- Merged `run-config.md` into `run-stats.md`. Each command's section in `run-stats.md` starts when you confirm pre-flight and already lists the model and effort per dispatch, so the separate record was redundant. Existing `run-config.md` files in old run folders are left alone.

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
