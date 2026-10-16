# Changelog

The plugin was named `dev-pipeline` before 0.4.0.

## bracket 0.8.2
- Bugs that predate the build no longer make an environment show FAIL on the Coverage line. If they're its only failures, it shows PASS with a note, e.g. "Unit tests: PASS (2 failing before this build)". The verdict is unchanged: a run with only pre-existing failures is PASS.
- The ending has a new **Pre-existing bugs** line under the summary, shown only when there are any, so you see them without reading down to the next steps.

## bracket 0.8.1
- Before the first builder pass, the run saves `pre-build.md`: the commit the build started from (including your uncommitted changes, recorded with `git stash create`) and the files that were untracked then. The verifier uses it to tell the builder's changes from yours, including new files that plain `git diff` misses.
- A failing existing test is always run against the code from before the build, in a temporary git worktree, instead of judging from the diff. It counts as PRE-EXISTING only if it runs there and fails the same way. If it can't run there, the cause is unclear.
- When the verifier can't tell whether the build caused a bug, it goes to you as UNVERIFIED and the check fails. It isn't sent to the builder, and it isn't filed as PRE-EXISTING.
- The test-suite check passes when its only failures are PRE-EXISTING. Those tests are recorded as known failures in the regression baseline, so a re-verify doesn't call them regressions.
- PRE-EXISTING bugs now reach you: re-verify carries them forward, and the ending lists them as a step.
- Worktree cleanup uses `git worktree remove --force` and `git worktree prune`, so test caches no longer leave a stale worktree behind.

## bracket 0.8.0
- The verifier can send a bug back to the builder even when no spec line covers it, as long as this build introduced it and the right fix is obvious. Examples: an existing test that now fails, or a crash or data loss in the new code. The fix request is marked **Introduced bug** and cites the line in the builder's diff that caused it. If fixing it needs a choice, it still goes to you as a Spec Issue.
- Bugs the verifier finds that were already there before the build no longer fail the run. They're listed for you as **PRE-EXISTING**.
- The verifier now always runs the project's whole existing test suite, not only the new tests. For each failing test, it checks whether this build caused it. When the diff doesn't make that clear, it runs the test against the code from before the build, in a temporary git worktree.

## bracket 0.7.0
- The builder records every gap it hits in the spec as a numbered Spec Issue, at the top of change-summary.md. **ASSUMED** means it filled in a small detail and kept going. **BLOCKED** means the spec's approach won't work, or it needs a decision bigger than a small assumption, so the builder stops and leaves its partial work in place.
- A blocked or failed build no longer goes on to verification. The run ends with state `BUILD BLOCKED` (or `BUILD ERROR`), and each blocker is a decision in Your next steps: amend the spec and resume, re-plan, or abandon. When you decide, your decision is added to spec.md under `## Amendments` and the builder picks up where it stopped.
- Assumptions are no longer buried in change-summary.md. The terminal flags them as soon as the builder returns, the ending has a **⚠ Spec issues** line, and a **Confirm the builder's assumptions** step lists each one.
- New builder outcomes in `run-stats.md`: `built · n ASSUMED` and `BLOCKED (SI-n, …)`.

## bracket 0.6.0
- A quieter terminal. Each agent writes its own report to the run folder and returns a status of at most three lines, so full reports and change summaries no longer scroll through the terminal as file diffs. Between dispatches, the orchestrating session prints one line per stage.
- Every command ends the same way: a short summary (verdict or state, coverage, why it's that verdict, what was built, run cost), then a numbered **Your next steps** list. Each step starts with a bold action and says why it matters. Manual steps, decisions, and unresolved fix requests are listed right there, so you don't have to open verification.md to find them. Halts (judge conflict, blocked spike gate) and `/bracket:plan` use the same shape.
- When the judge halts, it writes the disagreement to `conflict.md` instead of spec.md.
- In `run-stats.md`, new sections put Outcome last, so a long outcome no longer forces earlier rows to be re-padded. Existing sections keep their layout.

## bracket 0.5.1
- Spike and verifier reports put a Coverage line under the verdict, listing each environment as PASS, FAIL, or NEEDS TESTING, e.g. "Simulator: PASS · Device: NEEDS TESTING (BU-1, BU-3)". The verifier also adds a Coverage table that names what's still needed. The final summary shows the Coverage line with the verdict.
- Results that arrive after a dispatch, like a device session, the user's Manual Verification, or a spec amendment, now update the record in place. The report's top summary, verdict, Coverage line, and any outdated status headings are rewritten, and the new results go in a dated section.
- `run-stats.md` gains a Current state block at the top. Spike and verifier outcomes include coverage. When later results change a row's outcome, the row keeps its original value and adds ` → <current> (MM-DD)`. `GATE BLOCKED` lists every BU that isn't CONFIRMED.

## bracket 0.5.0
- The verifier never edits project code. It sorts each failure as a build defect (the spec was clear and the code got it wrong), a spec issue, or an unverified assumption, and writes numbered fix requests with repro steps for build defects.
- Build defects get one automatic fix round. The builder fixes only the fix requests (a new fix mode) and checks each repro. The verifier then re-runs every check, marks each request FIXED or STILL FAILING, and flags regressions in `reverification.md`. The run then stops, with no second round.
- Spec issues and unconfirmed assumptions still go straight to you. An UNCONFIRMED verdict never triggers a fix round.
- `run-stats.md` adds a `fixed n/m` outcome for the fix-mode builder.

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
