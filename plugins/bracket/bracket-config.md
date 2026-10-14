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

## Run folder

Each run lives in `.claude/bracket-runs/<run-name>/`. Before 0.4.2, runs lived in
`.claude/pipeline-runs/`. If the run name's folder isn't in `.claude/bracket-runs/` but exists at
`.claude/pipeline-runs/<run-name>/`, tell the user and offer to move it (`mkdir -p
.claude/bracket-runs && mv .claude/pipeline-runs/<run-name> .claude/bracket-runs/`). Don't move it
without their confirmation, and don't read or write the old location otherwise.

## Dispatching

Pass `model` and `effort` explicitly from the confirmed pre-flight config on **every** dispatch,
including the spike-only builder pass. Don't rely on frontmatter or the session.

## Fix round

After the first verifier dispatch, read verification.md yourself. Don't rely only on the verdict
line.

- **The verdict is PASS, or `## Fix Requests` is "None.":** no fix round. Go to the final summary.
  SPEC ISSUE and UNVERIFIED findings are the user's to decide, not the builder's, so an
  UNCONFIRMED verdict never triggers a fix round.
- **There is at least one fix request:** run exactly one fix round, even if `## For the User` also
  has findings. Those carry into the final summary.
  1. Dispatch the builder in **fix mode**, using the `builder` role's model and effort, with
     spec.md, spike-results.md (if it exists), change-summary.md, and the `## Fix Requests`
     section of verification.md. The builder appends `## Fix round 1` to change-summary.md.
  2. Dispatch the verifier in **re-verify mode**, using the `verifier` role's model and effort,
     with spec.md, the updated change-summary.md, spike-results.md (if it exists), and
     verification.md. Save its output to `.claude/bracket-runs/<run-name>/reverification.md`.
     Keep verification.md as the first round's record.
  3. Stop. There is never a second fix round, whatever the re-verify verdict. Anything still
     failing, any REGRESSION, and any DECLINED fix request goes to the user in the final summary,
     with the choices: fix via a new builder dispatch, amend the spec, or abandon the run.

A fix round is part of the same invocation. It uses the roles already confirmed at pre-flight, so
don't ask again before dispatching it. The pre-flight table needs no extra rows for it.

In the final summary, the verdict and headline come from the last verifier report
(reverification.md if a fix round ran). Then add a short **Fix round** line: how many fix
requests, how many FIXED, STILL FAILING, or DECLINED, and any REGRESSIONs, named.

## Run stats

`.claude/bracket-runs/<run-name>/run-stats.md` is the run's record: the settings each
invocation used, what each dispatch cost, and what it produced. It's read-only history. No skill
reads settings back from it. If the file doesn't exist, create it with this header:

```
# Run stats: <run-name>

One section per bracket command, one row per agent dispatch.
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

- One row per agent dispatch, and only agent dispatches. Every dispatch gets a row, including
  the fix round's builder and verifier and ones that do setup work before another agent can run (e.g. the
  spike pass building the app so a Blocking Unknown can be tested). Work that isn't an agent
  dispatch, such as the user or the orchestrating session running a step by hand, gets no row,
  since it has no model, effort, or usage. Its result goes in the relevant output file (e.g.
  spike-results.md).
- Role: a name from the Role column of the default config table, with no qualifiers. The
  spike-only builder pass is `spike`. The fix round's dispatches are `builder` and `verifier`, and
  their Outcome values tell them apart.
- Tokens and duration come from the dispatch's completion usage (`subagent_tokens`,
  `duration_ms`). Format tokens as e.g. `48.2k` and duration as `Xm Ys`. If usage wasn't
  reported, write `n/a`. Never estimate.
- Outcome: one value from this list for the role. Add no other detail (for example, which plan a
  planner preferred goes in that planner's output file, not here):

  | Role            | Outcome values                                                      |
  |-----------------|---------------------------------------------------------------------|
  | plan-speed      | `plan written`, `error`                                             |
  | plan-extensible | `plan written`, `error`                                             |
  | plan-judge      | `spec written`, `halted: conflict`, `error`                         |
  | spike           | `ALL CONFIRMED`, `GATE BLOCKED (BU-n, …)` + coverage, `error`       |
  | builder         | `built`, `fixed n/m`, `error`                                       |
  | verifier        | `PASS`, `UNCONFIRMED`, `FAIL` + coverage, `error`                   |

  `fixed n/m` is the fix-mode builder: n of m fix requests FIXED. `GATE BLOCKED` lists every BU
  that isn't CONFIRMED, not only the first.

  **Coverage** comes from the report's Coverage line (see **Coverage** below). Add it to spike
  and verifier outcomes whenever the report names more than one environment, or names one as
  NEEDS TESTING. Use short environment names, for example
  `UNCONFIRMED · sim PASS · device NEEDS TESTING` or
  `GATE BLOCKED (BU-1, BU-3) · build PASS · device NEEDS TESTING`.

- Completed: local time (`HH:MM`). The section heading carries the date.
- Don't edit earlier sections, except for the two updates allowed in **Keeping the record
  current**: an Outcome update and the Current state block.

## Coverage

A verdict on its own doesn't say how far testing got. UNCONFIRMED can mean nothing ran, or that
everything passed on the simulator and only a device check remains. So every spike report and
verifier report puts a **Coverage** line right under its verdict. The line lists each environment
the checks needed, with one status each:

- `PASS`: every check that needs this environment ran here and passed.
- `FAIL`: at least one check failed here.
- `NEEDS TESTING`: nothing ran here yet. Say what blocked it, and name the BUs or checks that need it.

```
**Verdict: UNCONFIRMED**
**Coverage:** Simulator (iOS 27): PASS · Device: NEEDS TESTING (BU-1, BU-3, manual steps 6–10)
```

Environments are whatever the stack uses, for example build, unit tests, simulator, device,
local server, staging, or production deploy. The verdict rules stay the same. Coverage explains
the verdict and never replaces it.

## Keeping the record current

A run's files must show where the run stands now, not where it stood when each agent finished.
Results often arrive after the dispatch that wrote the report, for example:

- a device session run by hand,
- a spike re-run by the orchestrating session,
- the user's `## Manual Verification`,
- a spec amendment.

Whenever new results arrive, in this conversation or a later one, the orchestrating session
updates the record before it reports back to the user:

1. **The report that covers those results** (spike-results.md, verification.md, or
   reverification.md):
   - Add the new results as a dated section at the end, saying who ran them and what was
     observed.
   - Rewrite the summary at the top to match: the status table, the overall verdict, and the
     Coverage line. Add a note `Updated YYYY-MM-DD: <what changed>` under the verdict.
   - Fix every status further down that is now out of date (for example a `### BU-1: NOT RUN`
     heading). Change it to the current status and point it to the dated section. A reader
     should never meet a status that's no longer true.
   - Change the verifier's verdict only when the new results meet the verdict rules on their own
     terms. For example, UNCONFIRMED becomes PASS only if every NEEDS TESTING item has now been
     observed passing. Cite the evidence.
2. **run-stats.md:**
   - Update the Outcome of the dispatch row whose result changed. Keep its original value and add
     ` → <current> (MM-DD)`, for example
     `GATE BLOCKED (BU-1, BU-2, BU-3) · device NEEDS TESTING → device: BU-1, BU-3 CONFIRMED; BU-2 REFUTED (10-13)`.
   - Don't change any other column, and don't add a row for work that wasn't an agent dispatch.
   - Rewrite the **Current state** block (below).

### Current state block

run-stats.md has a `## Current state` block between the file header and the first invocation
section. Create it with the first spike or verifier row, and rewrite it each time a row is added
or an Outcome changes. It has one line per stage that has run, and it always shows the latest
status:

```
## Current state · updated 2026-10-14 09:50

- Spike gate: cleared on device 10-13. BU-1 and BU-3 CONFIRMED, BU-2 REFUTED (fallback per A1).
- Verification: UNCONFIRMED · Simulator PASS · Device NEEDS TESTING (BU-1, BU-3 in the built app).
```

This block and the Outcome updates are the only edits allowed in earlier parts of run-stats.md.

## Run cost line

At the end of every invocation, including ones that halt early, don't print the stats table.
Print one line instead, totaling every row in run-stats.md (the whole run, not only this
invocation):

```
Run cost so far: 1.48M tokens · 2h 16m agent time · 7 dispatches. Details in run-stats.md.
```

Agent time sums durations across dispatches, so parallel planners count twice. It's the cost, not
the wall-clock time. The orchestrating session's own tokens are not included. Print the full table
only if the user asks for it.
