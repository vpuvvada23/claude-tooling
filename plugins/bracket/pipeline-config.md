# Bracket — Agent Config and Run Stats

Shared by the bracket plugin's full, plan, and build-verify skills.

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

## Dispatching

Pass `model` and `effort` explicitly from the confirmed pre-flight config on **every** dispatch,
including the spike-only builder pass. Don't rely on frontmatter or the session.

## Run stats

`.claude/pipeline-runs/<run-name>/run-stats.md` is the run's record: the settings each
invocation used, what each dispatch cost, and what it produced. It's read-only history. No skill
reads settings back from it. If the file doesn't exist, create it with this header:

```
# Run stats: <run-name>

One section per pipeline invocation, one row per agent dispatch.
Model and effort are the settings confirmed at pre-flight. The runtime can override them
without reporting it, so treat them as requested, not guaranteed.
```

As soon as the user confirms the pre-flight config, append a section for the invocation, headed
`## bracket:<skill> · <YYYY-MM-DD HH:MM>` (skill is `full`, `plan`, or `build-verify`), with the
table header. Then, after each dispatch completes, append its row, with the columns padded so the
table lines up as plain text:

```
## bracket:plan · 2026-10-12 09:14

| Role            | Model | Effort | Tokens | Duration | Outcome      | Completed |
|-----------------|-------|--------|--------|----------|--------------|-----------|
| plan-speed      | opus  | high   | 145.3k | 7m 39s   | plan written | 09:21     |
| plan-extensible | opus  | high   | 186.1k | 11m 37s  | plan written | 09:25     |
```

Re-pad the section's earlier rows if a new row is wider.

- Tokens and duration come from the dispatch's completion usage (`subagent_tokens`,
  `duration_ms`). Format tokens as e.g. `48.2k` and duration as `Xm Ys`. If usage wasn't
  reported, write `n/a`. Never estimate.
- Outcome: exactly one value from this list for the role, with no extra detail (put details like
  which plan a planner preferred in its own output file, not here):

  | Role                 | Outcome values                                  |
  |----------------------|-------------------------------------------------|
  | plan-speed           | `plan written`, `error`                         |
  | plan-extensible      | `plan written`, `error`                         |
  | plan-judge           | `spec written`, `halted: conflict`, `error`     |
  | builder (spike-only) | `ALL CONFIRMED`, `GATE BLOCKED (BU-n)`, `error` |
  | builder              | `built`, `error`                                |
  | verifier             | `PASS`, `UNCONFIRMED`, `FAIL`, `error`          |

- Completed: local time (`HH:MM`). The section heading carries the date.
- Every dispatch gets its own row, including re-dispatches after a FAIL. Never edit earlier
  sections.

## Final stats table

At the end of every invocation, including ones that halt early, print the full contents of
run-stats.md (all sections for this run, not only this invocation's), followed by a **Run total**
line summing tokens and duration across every row. Durations are summed across dispatches, so
parallel planners count twice. That's the cost, not the wall-clock time. Note that the
orchestrating session's own tokens are not included.
