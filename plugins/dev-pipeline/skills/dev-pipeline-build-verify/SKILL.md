---
name: dev-pipeline-build-verify
description: Builds and verifies a feature from an existing pipeline run spec. Takes a run name, loads the judge spec from .claude/pipeline-runs/<run-name>/spec.md, dispatches builder then verifier, and presents the final summary. Use after dev-pipeline-plan has produced a spec you're happy with.
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the dev-pipeline plugin and may appear namespaced in the available-agents list (e.g. dev-pipeline:plan-speed). Dispatch using the name exactly as it appears in that list.


# Dev Pipeline — Build and Verify

## Purpose
Pick up an existing pipeline run after planning is done. Reads the reconciled spec produced
by dev-pipeline-plan (or dev-pipeline-full), runs the builder against it, then the verifier,
and presents the final summary for the user's review.

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

### Step 3 — Dispatch the builder
Call the builder subagent with the full contents of spec.md.
Save output to:
- .claude/pipeline-runs/<run-name>/change-summary.md

The builder does not stage, commit, or push anything — it only writes a change summary.

### Step 4 — Dispatch the verifier
Call the verifier subagent with both spec.md and change-summary.md.
Save output to:
- .claude/pipeline-runs/<run-name>/verification.md

**If the verifier reports FAIL: do NOT re-dispatch the builder automatically.** Present the
findings to the user and let them decide: fix via a new builder dispatch, amend the spec, or
abandon the run.

### Step 5 — Present final summary to the user
Do NOT commit or push anything. Present a brief summary:
- Files changed (from change-summary.md)
- Verifier verdict and any manual steps required (from verification.md)
- Locations of all artifacts: spec.md, change-summary.md, verification.md
- Let the user know they can commit and push these changes themselves once they've reviewed
  everything
