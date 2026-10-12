---
name: plan
description: Plans a feature idea through the full planner panel (plan-speed + plan-extensible + plan-judge) and saves the reconciled spec. Use when the user wants to plan before deciding whether to build. Stops after spec.md is saved — does not invoke builder or verifier.
argument-hint: "<run-name> [--agents role=model[:effort],...] [idea]"
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the bracket plugin and may appear namespaced in the available-agents list (e.g. bracket:plan-speed). Dispatch using the name exactly as it appears in that list.


# Bracket — Plan Only

## Purpose
Take a feature idea through the planning panel and produce a reconciled, buildable spec.
Stops after the judge writes spec.md. Builder and verifier are not invoked — use
/bracket:build-verify when ready to build.

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
tight — only ask what's actually needed. If the idea is already clear and complete, say so.

In the same message, show the pre-flight agent config table (per bracket-config.md) with only
plan-speed, plan-extensible, and plan-judge. Build-phase settings are chosen later, at
/bracket:build-verify's own pre-flight. Ask the user to confirm or change it. Do not dispatch
anything until both are settled.

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
disagreement summary and the final stats table, and wait for their decision. Do not save a
partial spec or proceed further.

### Step 6 — Stop and report
**Do NOT dispatch builder or verifier.** This skill's job ends when spec.md is saved.

Tell the user:
- The run name
- That idea.md, plan-speed.md, plan-extensible.md, spec.md, and run-stats.md are
  saved under .claude/bracket-runs/<run-name>/
- The spec's Blocking Unknowns, listed up front — these run as spikes and gate the build. If the
  spec includes an estimate, present it as conditional on them.
- The final stats table (per bracket-config.md)
- That they can run /bracket:build-verify with run name "<run-name>" when ready to build
