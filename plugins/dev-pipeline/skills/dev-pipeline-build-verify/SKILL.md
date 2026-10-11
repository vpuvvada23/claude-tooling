---
name: dev-pipeline-build-verify
description: Builds and verifies a feature from an existing pipeline run spec. Takes a run name, loads the judge spec from .claude/pipeline-runs/<run-name>/spec.md, runs any Blocking Unknown spikes as a gate, dispatches builder then verifier, and presents the final summary. Use after dev-pipeline-plan has produced a spec you're happy with.
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the dev-pipeline plugin and may appear namespaced in the available-agents list (e.g. dev-pipeline:plan-speed). Dispatch using the name exactly as it appears in that list.


# Dev Pipeline — Build and Verify

## Purpose
Pick up an existing pipeline run after planning is done. Reads the reconciled spec produced
by dev-pipeline-plan (or dev-pipeline-full), runs its Blocking Unknowns as spikes, runs the
builder against it, then the verifier, and presents the final summary for the user's review.

## Instructions

### Step 1 — Get the run name
If not already given, ask for the run name. It must correspond to an existing folder at
.claude/pipeline-runs/<run-name>/.

### Step 2 — Load and validate the spec
Read .claude/pipeline-runs/<run-name>/spec.md.

**If spec.md does not exist: STOP.** Tell the user clearly:
> "No spec found at .claude/pipeline-runs/<run-name>/spec.md. Run dev-pipeline-plan first
> to generate a spec, or check that the run name is correct."
Do not proceed, do not improvise a plan, do not look for alternatives.

**If spec.md exists but appears to be a partial or conflict-halted output** (e.g. ends with
an unresolved conflict summary rather than a resolved spec): STOP and flag it to the user.

**If spec.md has no `## Blocking Unknowns` section** (e.g. it predates the spike gate): STOP and
tell the user. Offer to re-run planning, or to proceed with no gate if they confirm the spec
rests on no unverified external assumptions.

### Step 3 — Spike gate (only if Blocking Unknowns is not "None.")
Dispatch the builder in **spike-only mode** with the full contents of spec.md. Save output to:
- .claude/pipeline-runs/<run-name>/spike-results.md

Read the report yourself. Do not rely only on its overall verdict line.
- **Every Blocking Unknown CONFIRMED** with observed evidence: continue to Step 4.
- **Any REFUTED or NOT RUN: STOP.** Do not dispatch the full build. Tell the user which BU failed,
  what was observed, and the spec's "If it fails" fallback. Let them decide: amend the spec,
  run the spike themselves, or abandon the run.

A spike is not confirmed by a template, a placeholder findings doc, or an assumption written into
code. If a "CONFIRMED" entry doesn't show what was actually run and observed, treat it as NOT RUN.

### Step 4 — Dispatch the builder
Call the builder subagent in **build mode** with the full contents of spec.md and, if it exists,
spike-results.md.
Save output to:
- .claude/pipeline-runs/<run-name>/change-summary.md

The builder does not stage, commit, or push anything — it only writes a change summary.

### Step 5 — Dispatch the verifier
Call the verifier subagent with spec.md, change-summary.md, and spike-results.md (if it exists).
Save output to:
- .claude/pipeline-runs/<run-name>/verification.md

**If the verifier reports FAIL or UNCONFIRMED: do NOT re-dispatch the builder automatically.**
Present the findings to the user and let them decide: fix via a new builder dispatch, amend the
spec, or abandon the run.

### Step 6 — Present final summary to the user
Do NOT commit or push anything. Lead with the honest state of the feature, not the file list:
1. **Verdict and headline first.** If the verdict is UNCONFIRMED or FAIL, open with a plain
   statement of what is unconfirmed or broken and what that means — e.g. "Structurally complete,
   but the core premise (BU-1: Photos lists the extension in its share sheet) is still untested.
   Until it is, we don't know the feature works at all." Don't let a verdict label carry that.
2. Spike results (if any ran).
3. Files changed (from change-summary.md).
4. Remaining manual steps (from verification.md).
5. Locations of all artifacts: spec.md, spike-results.md (if any), change-summary.md,
   verification.md.
6. Let the user know they can commit and push these changes themselves once they've reviewed
   everything.

If the user later reports the feature isn't working, reach for the available automation
(simulator tooling, browser automation, curl, logs) to reproduce and debug it yourself before
asking them to run manual test rounds.
