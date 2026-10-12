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
- Omitted parts keep their default.
- If a role, model alias, or effort is not recognized, stop and ask. Don't guess.
- If `--agents` names a role this skill won't dispatch (e.g. `builder` when only planning), tell
  the user it doesn't apply to this invocation.

## Resolving the config

Each invocation resolves its own config. Nothing carries over from earlier invocations:
1. The default table above.
2. `--agents` overrides from this invocation.
3. Any changes the user makes at the pre-flight prompt.

## Pre-flight

Before the first dispatch of the invocation, show a table titled **Pre-flight agent config** with
exactly three columns, Role | Model | Effort. Include only the roles this invocation will dispatch.
Show it in the same message as any clarifying questions, and ask the user to confirm it or
change it. Don't dispatch anything until they've confirmed. This is the user's only chance to
change settings for the invocation.

## Run config record

After the user confirms, append the confirmed settings to
`.claude/pipeline-runs/<run-name>/run-config.md`. It's a read-only historical record: no skill
reads settings back from it. If the file doesn't exist, create it with this header:

```
# Run config: <run-name>

Agent settings confirmed at the start of each pipeline invocation for this run.
This is a record only. Settings are chosen at the pre-flight prompt when a run starts.
```

Then append one section per invocation, with the columns padded so the table lines up as plain
text:

```
## dev-pipeline-plan · 2026-10-12 09:04

| Role            | Model | Effort |
|-----------------|-------|--------|
| plan-speed      | opus  | high   |
| plan-extensible | opus  | high   |
| plan-judge      | opus  | xhigh  |
```

Never overwrite or edit earlier sections.

## Dispatching

Pass `model` and `effort` explicitly from the confirmed pre-flight config on **every** dispatch,
including the spike-only builder pass. Don't rely on frontmatter or the session.

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
