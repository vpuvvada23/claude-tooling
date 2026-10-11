# Dev Pipeline — Agent Config and Run Stats

Shared by the dev-pipeline-full, dev-pipeline-plan, and dev-pipeline-build-verify skills.

## Default agent config

| Role | Agent | Model | Effort |
|---|---|---|---|
| plan-speed | plan-speed | opus | high |
| plan-extensible | plan-extensible | opus | high |
| plan-judge | plan-judge | opus | xhigh |
| spike | builder (spike-only mode) | opus | high |
| builder | builder (build mode) | sonnet | high |
| verifier | verifier | opus | high |

Models are family aliases (`opus`, `sonnet`, `haiku`, `fable`), so they track the latest model in
each family. A full model ID (e.g. `claude-opus-5-5`) also works if you want to pin. Efforts:
`low`, `medium`, `high`, `xhigh`, `max`. If a model doesn't support a level, the runtime clamps it
down to the nearest level the model does support.

Each agent's frontmatter carries the same defaults, used only when the agent is dispatched
outside these skills. Keep the two in sync. `spike` has no frontmatter of its own, because it's the
builder agent dispatched with a different model.

## Per-run overrides

Pass `--agents` when starting a skill:

```
--agents builder=opus,verifier=sonnet:xhigh
--agents all=opus:xhigh
--agents all=sonnet,plan-judge=opus:xhigh
```

- Format: comma-separated `role=model[:effort]` or `role=:effort` (effort only). Roles are the
  table's Role column, plus `all`. Specific roles override `all`, regardless of order.
- Omitted parts keep their resolved value.
- If a role, model alias, or effort is not recognized, stop and ask. Don't guess.

## Resolving a run's config

Apply in order, each layer overriding the last:
1. The default table above.
2. `.claude/pipeline-runs/<run-name>/run-config.md`, if it exists (an earlier invocation for this
   run already chose a config).
3. `--agents` overrides from this invocation.

## Pre-flight

Before the first dispatch of the invocation, show the resolved config as a table with columns
Role, Model, Effort, Source (`default`, `run-config`, or `--agents`). Mark which roles this skill
will actually dispatch. Show it in the same message as any clarifying questions, and ask the
user to confirm it or change it. Don't dispatch anything until they've confirmed. Apply any edits
they make, then write the final table to `.claude/pipeline-runs/<run-name>/run-config.md`
(overwriting).

## Dispatching

Pass `model` and `effort` explicitly from run-config.md on **every** dispatch, including the
spike-only builder pass. Don't rely on frontmatter or the session.

## Run stats

After each dispatch completes, append one row to `.claude/pipeline-runs/<run-name>/run-stats.md`
(create it with the header if missing):

```
| Role | Model (requested) | Effort (requested) | Tokens | Tool uses | Duration | Outcome | Completed |
|---|---|---|---|---|---|---|---|
```

- Tokens, tool uses, and duration come from the dispatch's completion usage (`subagent_tokens`,
  `tool_uses`, `duration_ms`). Format duration as `Xm Ys` and tokens as e.g. `48.2k`. If usage
  wasn't reported, write `n/a`. Never estimate.
- Outcome: a short result — e.g. `spec written`, `halted: conflict`, `ALL CONFIRMED`,
  `GATE BLOCKED (BU-1)`, `PASS`, `UNCONFIRMED`, `FAIL`, `error`.
- Completed: local date and time.
- Every dispatch gets its own row, including re-dispatches after a FAIL. Never overwrite earlier
  rows.

Model and effort are what was **requested**. The runtime can change them without reporting it: an
environment variable such as `CLAUDE_CODE_EFFORT_LEVEL` or `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`, an
org model allowlist, or effort clamping. Say so if the user asks.

## Final stats table

At the end of every invocation, including ones that halt early, print the full contents of
run-stats.md (all rows for this run, not only this invocation's) with a **Total** row summing
tokens, tool uses, and duration. Durations are summed across dispatches, so parallel planners
count twice. That's the cost, not the wall-clock time. Note that the orchestrating session's
own tokens are not included.
