---
name: dev-pipeline-plan
description: Plans a feature idea through the full planner panel (plan-speed + plan-extensible + plan-judge) and saves the reconciled spec. Use when Vinnie wants to plan before deciding whether to build. Stops after spec.md is saved — does not invoke builder or verifier.
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the dev-pipeline plugin and may appear namespaced in the available-agents list (e.g. dev-pipeline:plan-speed). Dispatch using the name exactly as it appears in that list.


# Dev Pipeline — Plan Only

## Purpose
Take a feature idea through the planning panel and produce a reconciled, buildable spec.
Stops after the judge writes spec.md. Builder and verifier are not invoked — use
dev-pipeline-build-verify when ready to build.

## Instructions

### Step 1 — Get the run name and idea
If not already given, ask for a short run-name (used as the folder name) and the feature
idea itself.

### Step 2 — Ask clarifying questions
Before creating anything, review the idea for ambiguity — scope, edge cases, which existing
files/systems it touches, anything a planner would need to know to avoid guessing. Ask Vinnie
any clarifying questions now, before dispatching planners. Keep this tight — only ask what's
actually needed. If the idea is already clear and complete, skip this step and say so.

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

**If plan-judge halts due to a genuine irreconcilable conflict: STOP HERE.** Show Vinnie the
disagreement summary and wait for his decision. Do not save a partial spec or proceed further.

### Step 6 — Stop and report
**Do NOT dispatch builder or verifier.** This skill's job ends when spec.md is saved.

Tell Vinnie:
- The run name
- That plan-speed.md, plan-extensible.md, and spec.md are saved under .claude/pipeline-runs/<run-name>/
- That he can run dev-pipeline-build-verify with run name "<run-name>" when he's ready to build
