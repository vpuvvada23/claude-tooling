---
name: build-verify
description: Builds and verifies a feature from an existing pipeline run spec. Takes a run name, loads the judge spec from .claude/pipeline-runs/<run-name>/spec.md, runs any Blocking Unknown spikes as a gate, dispatches builder then verifier, and presents the final summary. Use after /bracket:plan has produced a spec you're happy with.
argument-hint: "<run-name> [--agents role=model[:effort],...]"
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the bracket plugin and may appear namespaced in the available-agents list (e.g. bracket:plan-speed). Dispatch using the name exactly as it appears in that list.


# Bracket — Build and Verify

## Purpose
Pick up an existing pipeline run after planning is done. Reads the reconciled spec produced
by /bracket:plan (or /bracket:full), runs its Blocking Unknowns as spikes, runs the
builder against it, then the verifier, and presents the final summary for the user's review.

## Instructions

Read `${CLAUDE_PLUGIN_ROOT}/pipeline-config.md` first. It defines the default agent config,
the `--agents` override syntax, the pre-flight confirmation, how to pass model and effort on
each dispatch, and the run-stats format. Follow it throughout this skill.

### Step 1 — Get the run name and overrides
Parse the skill's arguments: the run name and any `--agents ...` overrides. If the run name is
missing, ask for it. It must correspond to an existing folder at .claude/pipeline-runs/<run-name>/.

### Step 2 — Load and validate the spec
Read .claude/pipeline-runs/<run-name>/spec.md.

**If spec.md does not exist: STOP.** Tell the user clearly:
> "No spec found at .claude/pipeline-runs/<run-name>/spec.md. Run /bracket:plan first
> to generate a spec, or check that the run name is correct."
Do not proceed, do not improvise a plan, do not look for alternatives.

**If spec.md exists but appears to be a partial or conflict-halted output** (e.g. ends with
an unresolved conflict summary rather than a resolved spec): STOP and flag it to the user.

**If spec.md has no `## Blocking Unknowns` section** (e.g. it predates the spike gate): STOP and
tell the user. Offer to re-run planning, or to proceed with no gate if they confirm the spec
rests on no unverified external assumptions.

### Step 3 — Pre-flight
Resolve the config per pipeline-config.md (defaults, then `--agents`). Don't read settings from
an existing run-config.md, because it's a record only. Show the pre-flight agent config table with
only the roles this skill will dispatch: spike (only if the spec has Blocking Unknowns), builder,
and verifier. Ask the user to confirm or change it. Do not dispatch anything until they confirm.
Then append the confirmed config to the run config record at
.claude/pipeline-runs/<run-name>/run-config.md.

### Step 4 — Spike gate (only if Blocking Unknowns is not "None.")
Dispatch the builder in **spike-only mode** with the full contents of spec.md, using the `spike`
role's model and effort. Save output to:
- .claude/pipeline-runs/<run-name>/spike-results.md

Read the report yourself. Do not rely only on its overall verdict line.
- **Every Blocking Unknown CONFIRMED** with observed evidence: continue to Step 5.
- **Any REFUTED or NOT RUN: STOP.** Do not dispatch the full build. Tell the user which BU failed,
  what was observed, and the spec's "If it fails" fallback, and print the final stats table. Let
  them decide: amend the spec, run the spike themselves, or abandon the run.

A spike is not confirmed by a template, a placeholder findings doc, or an assumption written into
code. If a "CONFIRMED" entry doesn't show what was actually run and observed, treat it as NOT RUN.

### Step 5 — Dispatch the builder
Call the builder subagent in **build mode** with the full contents of spec.md and, if it exists,
spike-results.md, using the `builder` role's model and effort.
Save output to:
- .claude/pipeline-runs/<run-name>/change-summary.md

The builder does not stage, commit, or push anything — it only writes a change summary.

### Step 6 — Dispatch the verifier
Call the verifier subagent with spec.md, change-summary.md, and spike-results.md (if it exists).
Save output to:
- .claude/pipeline-runs/<run-name>/verification.md

**If the verifier reports FAIL or UNCONFIRMED: do NOT re-dispatch the builder automatically.**
Present the findings to the user and let them decide: fix via a new builder dispatch, amend the
spec, or abandon the run.

### Step 7 — Present final summary to the user
Do NOT commit or push anything. Lead with the honest state of the feature, not the file list:
1. **Verdict and headline first.** If the verdict is UNCONFIRMED or FAIL, open with a plain
   statement of what is unconfirmed or broken and what that means — e.g. "Structurally complete,
   but the core premise (BU-1: Photos lists the extension in its share sheet) is still untested.
   Until it is, we don't know the feature works at all." Don't let a verdict label carry that.
2. Spike results (if any ran).
3. Files changed (from change-summary.md).
4. Remaining manual steps (from verification.md).
5. Locations of all artifacts: run-config.md, spec.md, spike-results.md (if any),
   change-summary.md, verification.md, run-stats.md.
6. The final stats table (per pipeline-config.md). It covers the whole run, including the
   planning rows if they were recorded.
7. Let the user know they can commit and push these changes themselves once they've reviewed
   everything.

If the user later reports the feature isn't working, reach for the available automation
(simulator tooling, browser automation, curl, logs) to reproduce and debug it yourself before
asking them to run manual test rounds.
