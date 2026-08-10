---
name: dev-pipeline-full
description: Runs a feature idea through a full plan-build-verify agent pipeline. Trigger when the user says "run the dev pipeline", "kick off dev-pipeline for <run-name>", or describes wanting a feature built via the planning panel.
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the dev-pipeline plugin and may appear namespaced in the available-agents list (e.g. dev-pipeline:plan-speed). Dispatch using the name exactly as it appears in that list.


# Dev Pipeline Skill

## Purpose
Take a feature idea from plan through build to verification, using a panel of specialized
subagents, saving every stage's output for the user's end-of-chain review.

## Instructions

### Step 1 — Get the run name and idea
If not already given, ask for a short run-name (used as the folder name) and the feature idea itself.

### Step 2 — Ask clarifying questions
Before creating anything, review the idea for ambiguity — scope, edge cases, which existing
files/systems it touches, anything a planner would need to know to avoid guessing. Ask the user
any clarifying questions now, before dispatching planners. Keep this tight — only ask what's
actually needed to plan, not exhaustive discovery. If the idea is already clear and
complete, skip this step and say so.

### Step 3 — Create the run folder
mkdir -p .claude/pipeline-runs/<run-name>

Write the feature idea and clarifying Q&A to:
- .claude/pipeline-runs/<run-name>/idea.md

### Step 4 — Dispatch both planners in parallel
Call the plan-speed and plan-extensible subagents in parallel with the same feature idea (plus any
clarifications from Step 2).
Save their outputs to:
- .claude/pipeline-runs/<run-name>/plan-speed.md (plan-speed output)
- .claude/pipeline-runs/<run-name>/plan-extensible.md (plan-extensible output)

### Step 5 — Dispatch the judge
Call plan-judge with both plans AND the original feature idea (plus clarifying answers from Step 2).
Save output to:
- .claude/pipeline-runs/<run-name>/spec.md

**If plan-judge halts due to a genuine irreconcilable conflict: STOP HERE.** Show the user the
disagreement summary and wait for their decision before proceeding. Do not dispatch the builder.

### Step 6 — Dispatch the builder (only if spec.md exists and is a resolved spec)
Read spec.md first. If spec.md does not exist: STOP and tell the user. If spec.md appears to be a
partial or conflict-halted output (e.g. ends with an unresolved conflict summary): STOP and flag
it to the user. Do not improvise.

Call builder with spec.md. The builder does not stage, commit, or push anything — it only
writes a change summary. Save output to:
- .claude/pipeline-runs/<run-name>/change-summary.md

### Step 7 — Dispatch the verifier
Call the verifier subagent with both spec.md and change-summary.md. Save output to:
- .claude/pipeline-runs/<run-name>/verification.md

**If the verifier reports FAIL: do NOT re-dispatch the builder automatically.** Present the
findings to the user and let them decide: fix via a new builder dispatch, amend the spec, or
abandon the run.

### Step 8 — Present final summary to the user
Do NOT commit or push anything. Present a brief summary of all artifacts and their locations,
and let the user review spec.md, change-summary.md, and verification.md before deciding to
commit and push the changes themselves.
