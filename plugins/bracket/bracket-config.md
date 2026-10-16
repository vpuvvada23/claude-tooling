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

## Pre-build snapshot

Before the run's first builder dispatch (spike-only or build mode), write
`.claude/bracket-runs/<run-name>/pre-build.md` yourself, unless it already exists. Never rewrite
it on a resume or fix round. The verifier uses it to tell the builder's changes apart from what was
already in the working tree.

```
# Pre-build snapshot · <YYYY-MM-DD HH:MM>
Base: <commit>
HEAD: <commit>
Untracked before build:
<one path per line, or "none">
```

- **Base:** run `git stash create`. If it prints a commit, the working tree had uncommitted
  changes, and that commit records them. It doesn't change the working tree, the index, or the
  stash list. If it prints nothing, Base is the output of `git rev-parse HEAD`.
- **Untracked before build:** the output of `git ls-files --others --exclude-standard`.
- With no commits yet, write `Base: none (no commits)`. Outside a git repo, write
  `Base: none (not a git repo)`. Leave out the other lines in both cases.

## Dispatching

Pass `model` and `effort` explicitly from the confirmed pre-flight config on **every** dispatch,
including the spike-only builder pass. Don't rely on frontmatter or the session.

Also pass the **output path** on every dispatch: the file in the run folder the agent writes its
report to (plan-speed.md, spec.md, spike-results.md, change-summary.md, verification.md,
reverification.md). Pass input files by path too, and let the agent read them, rather than pasting
their contents into the prompt. The agent writes the full report itself and returns only a short
status, so you never write report files yourself.

## Terminal output

The user wants a clean terminal: what's running, then a summary they can dig into. Everything
detailed lives in the run folder.

- **Never echo a report.** Don't paste an agent's report into the conversation, and don't write or
  rewrite a report file yourself. The only exception is **Keeping the record current**. When you
  need a report's contents (the spike gate, the fix round, the final summary), read the file.
- **One line per stage.** Before each dispatch, print one line, e.g. `→ builder running (sonnet,
  high)`. After it returns, print one line with its status, e.g. `✓ spec written · 3 Blocking
  Unknowns` or `✗ spike gate blocked: BU-1 NOT RUN`. No other narration between dispatches.
- **Quiet bookkeeping.** Make each run-stats.md update a single small edit, with no commentary
  around it. Don't re-pad earlier rows (see **Run stats**).
- Questions to the user (clarifying questions, pre-flight, a halt that needs a decision) and the
  ending (see **Ending an invocation**) are the only longer messages.

## Agent status line

Every agent ends by returning a status of at most three lines, and nothing else:

```
<outcome value from the run-stats table> · <one-line headline>
Coverage: <the report's Coverage line>        (spike and verifier only)
Wrote <output path>
```

If the agent couldn't write the file, it says so in the status. Then stop and tell the user. Don't
write the file yourself from the agent's reply.

## Build gate

After the build-mode builder returns, read the `## Spec Issues` section of change-summary.md
yourself. Don't rely only on the status line. The verifier runs only on a finished build:

- **`built` or `built · n ASSUMED`, and no BLOCKED issue in the summary:** dispatch the verifier.
  If there are ASSUMED issues, the after-dispatch line names them, e.g.
  `✓ built · ⚠ 2 assumptions (SI-1, SI-3), confirm at the end`.
- **`BLOCKED`, any BLOCKED issue in the summary, or `error`: STOP.** Don't dispatch the verifier,
  since it would only test a half-built feature. Print `✗ build blocked: SI-2 <the decision
  needed>` (or `✗ builder error: <headline>`), then end the invocation with state `BUILD BLOCKED`
  (or `BUILD ERROR`). The partial changes stay in the working tree.

If the user resolves a BLOCKED issue in conversation, add their decision to spec.md as a dated
entry under `## Amendments` at the end (create the section if needed), naming the SI it resolves.
That edit, the **Pre-build snapshot**, and **Keeping the record current** are the only times you
write to a run file yourself.
Then dispatch the builder in build mode again, with spec.md, spike-results.md (if it exists), and
the existing change-summary.md, so it resumes rather than starting over. It gets its own
run-stats row. From there the run continues as normal: build gate, verifier, fix round, ending.
If the decision changes a Blocking Unknown, or is too big to record as an amendment, recommend
re-planning instead.

## Fix round

After the first verifier dispatch, read verification.md yourself. Don't rely only on the verdict
line.

- **The verdict is PASS, or `## Fix Requests` is "None.":** no fix round. Go to the final summary.
  SPEC ISSUE, UNVERIFIED, and PRE-EXISTING findings are the user's to decide, not the builder's,
  so an UNCONFIRMED verdict never triggers a fix round.
- **There is at least one fix request:** run exactly one fix round, even if `## For the User` also
  has findings. Those carry into the final summary.
  1. Dispatch the builder in **fix mode**, using the `builder` role's model and effort, with
     spec.md, spike-results.md (if it exists), change-summary.md, and the `## Fix Requests`
     section of verification.md. The builder appends `## Fix round 1` to change-summary.md.
  2. Dispatch the verifier in **re-verify mode**, using the `verifier` role's model and effort,
     with spec.md, the updated change-summary.md, spike-results.md (if it exists), pre-build.md,
     and verification.md. It writes `.claude/bracket-runs/<run-name>/reverification.md`.
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
table header. Then, after each dispatch completes, append its row, padding each cell to the
header's column width:

```
## bracket:plan · 2026-10-12 09:14

| Role            | Model  | Effort | Tokens | Duration | Completed | Outcome
|-----------------|--------|--------|--------|----------|-----------|--------
| plan-speed      | opus   | high   | 145.3k | 7m 39s   | 09:21     | plan written
| plan-extensible | opus   | high   | 186.1k | 11m 37s  | 09:25     | plan written
```

Outcome is last and open-ended, so a long outcome never forces the earlier rows to be re-padded.
Each new row is a one-line append. A pinned model ID longer than the Model column just runs
wider. Don't re-pad other rows for it. Sections written before 0.6.0 keep their old layout. Don't
convert them.

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
  | builder         | `built`, `built · n ASSUMED`, `BLOCKED (SI-n, …)`, `fixed n/m` (+ ` · n ASSUMED`), `error` |
  | verifier        | `PASS`, `UNCONFIRMED`, `FAIL` + coverage, `error`                   |

  `fixed n/m` is the fix-mode builder: n of m fix requests FIXED. `GATE BLOCKED` lists every BU
  that isn't CONFIRMED, not only the first. `BLOCKED` lists every BLOCKED Spec Issue.

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
- `FAIL`: at least one check failed here. Failures the verifier marked PRE-EXISTING don't count.
  If they're the only ones, the status is `PASS` with a note, e.g.
  `Unit tests: PASS (2 failing before this build)`.
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

The ending (see **Ending an invocation**) includes the run cost, not the stats table.
It is one line, totaling every row in run-stats.md (the whole run, not only this
invocation):

```
Run cost so far: 1.48M tokens · 2h 16m agent time · 7 dispatches. Details in run-stats.md.
```

Agent time sums durations across dispatches, so parallel planners count twice. It's the cost, not
the wall-clock time. The orchestrating session's own tokens are not included. Print the full table
only if the user asks for it.

## Ending an invocation

Every invocation ends with the same shape, whether it finished or halted early (a judge conflict,
a blocked spike gate). It's short, and it ends with **what the user does next**, so they never
have to dig through the run folder to find it. Read the last report to fill it in. Don't paste
from it.

```
## <run-name>: <VERDICT or STATE>

**Coverage:** <the last report's Coverage line>

**Why it's <verdict>:** <1–3 plain sentences: what's proven, what isn't, and what that means for
whether the feature works at all>

**⚠ Spec issues:** <n BLOCKED, n ASSUMED: SI ids with a few words each> (only if any)

**Pre-existing bugs:** <n> found, not caused by this build: <test names or a few words each> (only if any)

**Fix round:** <tally, per Fix round> (only if one ran)

**Built:** <one sentence on what changed> · <n> files · details in change-summary.md

Run cost so far: … Details in run-stats.md.
Run folder: .claude/bracket-runs/<run-name>/

---

### Your next steps

1. **<Action, starting with a verb>.** <Why: what it unblocks or proves, e.g. "This is BU-1,
   the core premise. Until it passes, the verdict stays UNCONFIRMED.">
   - <the concrete steps, one line each>
2. **Decide: <the question>.** <Why it needs you.> Options: **(a)** …, **(b)** …. I'd go with …
   because …
3. **Report back.** Tell me what you saw (or append `## Manual Verification` to verification.md),
   and I'll update the record and the verdict.
4. **Review and commit.** Nothing is staged or committed. Review the diff and commit when you're
   happy.
```

**State.** Use the verdict (PASS, UNCONFIRMED, FAIL) when a verifier ran. Otherwise use `SPEC
READY`, `HALTED: plan conflict`, `GATE BLOCKED`, `BUILD BLOCKED`, or `BUILD ERROR`. For a halt,
"Why" explains what stopped the run, and "Built" is omitted, except for `BUILD BLOCKED` and
`BUILD ERROR`, where it becomes **Built so far:**, noting what's done, what isn't, and that the
partial changes are uncommitted in the working tree. Omit Coverage when no spike or verifier
report exists yet.

**Spec issues.** The builder's Spec Issues (change-summary.md → `## Spec Issues`, including any
from the fix round) always reach the ending. Never leave them only in the file:
- Each BLOCKED issue is a **Decide** step, first in the list: the gap in one line, the options
  the builder named, and your recommendation. Choices always include amending the spec (I record
  the decision in spec.md and resume the build), re-planning with `/bracket:plan`, or abandoning the
  run and discarding the partial changes.
- ASSUMED issues share one step, **Confirm the builder's assumptions**, with one line each: the
  SI id, what the spec left open, and what the builder chose. Say why it matters: the verifier
  checked the code against these choices, not against what you intended. If one is wrong, say
  so and it becomes a spec amendment plus a builder fix.

**Next steps**, the part the user acts on:
- Number them, in the order the user should do them. Put first whatever turns the verdict into
  PASS, or unblocks the run.
- Each step opens with a bold action and says why it matters. Explain "why it's still
  UNCONFIRMED" in terms of the specific steps that would confirm it.
- Include the actual actions. Condense the verifier's Manual Steps and still-needed checks into
  one line each under the step they belong to, so the user doesn't have to open
  verification.md to know what to do. Point to the file only for long command blocks or full
  detail, naming the section, e.g. "full commands in verification.md → Manual Steps 6–10".
- Every builder Spec Issue (see above), verifier SPEC ISSUE, UNVERIFIED item, DECLINED or STILL
  FAILING fix request, and REGRESSION becomes a step or sits under one. PRE-EXISTING bugs go in
  one step, e.g. **Fix bugs that predate this build**, since they don't block this feature but
  still need fixing. A decision lists the choices and your recommendation.
- Leave out anything the user doesn't need to act on. Passing checks and the full file list stay
  in the run folder.
- Always end with the review-and-commit step after a build, except after `BUILD BLOCKED` or
  `BUILD ERROR`, where the code is unfinished and shouldn't be committed. After `/bracket:plan`, end with running
  `/bracket:build-verify <run-name>`.
- On a PASS with nothing left to do, steps are only "Review and commit". Say so plainly.
