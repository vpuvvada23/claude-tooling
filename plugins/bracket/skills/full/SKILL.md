---
name: full
description: Runs a feature idea through bracket's full plan-build-verify agent flow. Trigger when the user says "run bracket", "run the dev pipeline", "kick off bracket for <run-name>", or describes wanting a feature built via the planning panel.
argument-hint: "<run-name> [--agents role=model[:effort],...] [idea]"
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the bracket plugin and may appear namespaced in the available-agents list (e.g. bracket:plan-speed). Dispatch using the name exactly as it appears in that list.


# Bracket — Full Run

## Purpose
Take a feature idea from plan through build to verification, using a panel of specialized
subagents, saving every stage's output for the user's end-of-chain review.

## Instructions

Read `${CLAUDE_PLUGIN_ROOT}/bracket-config.md` first. It defines the default agent config,
the `--agents` override syntax, the pre-flight confirmation, how to pass model and effort on
each dispatch, and the run-stats format. Follow it throughout this skill.

### Step 1 — Get the run name, idea, and overrides
Parse the skill's arguments: the run name, any `--agents ...` overrides, and the feature idea.
If the run name or idea is missing, ask for it.

### Step 2 — Clarifying questions and pre-flight
Before creating anything, review the idea for ambiguity — scope, edge cases, which existing
files/systems it touches, anything a planner would need to know to avoid guessing. Keep this
tight — only ask what's actually needed to plan, not exhaustive discovery. If the idea is
already clear and complete, say so.

In the same message, show the pre-flight agent config table (per bracket-config.md) with all
six roles, since this skill can dispatch all of them. Below the table, note that spike only runs
if the spec has Blocking Unknowns. Ask the user to confirm or change it. Do not dispatch anything
until both are settled.

### Step 3 — Create the run folder
mkdir -p .claude/bracket-runs/<run-name>

Write the feature idea and clarifying Q&A to:
- .claude/bracket-runs/<run-name>/idea.md

Start this invocation's section in the run stats record (per bracket-config.md):
- .claude/bracket-runs/<run-name>/run-stats.md

### Step 4 — Dispatch both planners in parallel
Call the plan-speed and plan-extensible subagents in parallel with the same feature idea (plus any
clarifications from Step 2).
Save their outputs to:
- .claude/bracket-runs/<run-name>/plan-speed.md (plan-speed output)
- .claude/bracket-runs/<run-name>/plan-extensible.md (plan-extensible output)

### Step 5 — Dispatch the judge
Call plan-judge with both plans AND the original feature idea (plus clarifying answers from Step 2).
Save output to:
- .claude/bracket-runs/<run-name>/spec.md

**If plan-judge halts due to a genuine irreconcilable conflict: STOP HERE.** Show the user the
disagreement summary and the final stats table, and wait for their decision before proceeding.
Do not dispatch the builder.

### Step 6 — Spike gate (only if spec.md exists and is a resolved spec)
Read spec.md first. If spec.md does not exist: STOP and tell the user. If spec.md appears to be a
partial or conflict-halted output (e.g. ends with an unresolved conflict summary): STOP and flag
it to the user. Do not improvise.

If the spec's `## Blocking Unknowns` section is anything other than "None.", dispatch the builder
in **spike-only mode** with spec.md, using the `spike` role's model and effort. Save output to:
- .claude/bracket-runs/<run-name>/spike-results.md

Read the report yourself. Do not rely only on its overall verdict line.
- **Every Blocking Unknown CONFIRMED** with observed evidence: continue to Step 7.
- **Any REFUTED or NOT RUN: STOP.** Do not dispatch the full build. Tell the user which BU failed,
  what was observed, and the spec's "If it fails" fallback, and print the final stats table. Let
  them decide: amend the spec, run the spike themselves, or abandon the run.

A spike is not confirmed by a template, a placeholder findings doc, or an assumption written into
code. If a "CONFIRMED" entry doesn't show what was actually run and observed, treat it as NOT RUN.
Any estimate in the spec stays conditional until this gate clears. Don't quote it to the user as a
settled number before then.

### Step 7 — Dispatch the builder
Call builder in **build mode** with spec.md and, if it exists, spike-results.md, using the
`builder` role's model and effort. The builder does not stage, commit, or push anything — it only
writes a change summary. Save output to:
- .claude/bracket-runs/<run-name>/change-summary.md

### Step 8 — Dispatch the verifier
Call the verifier subagent with spec.md, change-summary.md, and spike-results.md (if it exists).
Save output to:
- .claude/bracket-runs/<run-name>/verification.md

**If the verifier reports FAIL or UNCONFIRMED: do NOT re-dispatch the builder automatically.**
Present the findings to the user and let them decide: fix via a new builder dispatch, amend the
spec, or abandon the run.

### Step 9 — Present final summary to the user
Do NOT commit or push anything. Lead with the honest state of the feature, not the file list:
1. **Verdict and headline first.** If the verdict is UNCONFIRMED or FAIL, open with a plain
   statement of what is unconfirmed or broken and what that means — e.g. "Structurally complete,
   but the core premise (BU-1: Photos lists the extension in its share sheet) is still untested.
   Until it is, we don't know the feature works at all." Don't let a verdict label carry that.
2. Spike results (if any ran).
3. Files changed (from change-summary.md).
4. Remaining manual steps (from verification.md).
5. Locations of all artifacts: idea.md, plan-speed.md, plan-extensible.md, spec.md,
   spike-results.md (if any), change-summary.md, verification.md, run-stats.md.
6. The final stats table (per bracket-config.md).
7. Let the user know they can review everything and commit and push the changes themselves.

If the user later reports the feature isn't working, reach for the available automation
(simulator tooling, browser automation, curl, logs) to reproduce and debug it yourself before
asking them to run manual test rounds.
