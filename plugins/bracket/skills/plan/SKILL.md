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
each dispatch, the run-stats format, how to keep the terminal quiet, and how to end the invocation.
Follow it throughout this skill.

### Step 1 — Get the run name, idea, and overrides
Parse the skill's arguments: the run name, any `--agents ...` overrides, and the feature idea.
If the run name or idea is missing, ask for it.

### Step 2 — Size check, clarifying questions, and pre-flight
First do the **Size check** in bracket-config.md. If the request is small, recommend skipping
bracket and stop until the user chooses.

Then, before creating anything, review the idea for ambiguity — scope, edge cases, which existing
files/systems it touches, anything a planner would need to know to avoid guessing. Keep this
tight — only ask what's actually needed. If the idea is already clear and complete, say so.

Also settle the run's **direction**, unless the idea or project already makes it clear (e.g. "quick
test of…", a scratch repo, a stated roadmap):
- **prototype** — a throwaway or market test; no long-term plans yet
- **growing** — expected to expand; ask how (the specific directions, e.g. "add Android next",
  "multi-user later")
- **not sure** — a fine answer; don't push for one

If you inferred it rather than asked, state it so the user can correct it.

In the same message, show the pre-flight agent config table (per bracket-config.md) with only
plan-speed, plan-extensible, and plan-judge. Build-phase settings are chosen later, at
/bracket:build-verify's own pre-flight. Ask the user to confirm or change it. Do not dispatch
anything until both are settled.

### Step 3 — Create the run folder
mkdir -p .claude/bracket-runs/<run-name>

Write the feature idea and clarifying Q&A to:
- .claude/bracket-runs/<run-name>/idea.md

Start idea.md with the direction as labeled lines, before the idea itself:
```
Direction: prototype | growing | not sure
Expected expansion: <the directions named, or "none stated">
```

Start this invocation's section in the run stats record (per bracket-config.md):
- .claude/bracket-runs/<run-name>/run-stats.md

### Step 4 — Dispatch both planners in parallel
Call the plan-speed and plan-extensible subagents in parallel, each with the path to idea.md
(which holds the idea and clarifications from Step 2). Each writes its own plan to:
- .claude/bracket-runs/<run-name>/plan-speed.md (plan-speed)
- .claude/bracket-runs/<run-name>/plan-extensible.md (plan-extensible)

### Step 5 — Dispatch the judge
Call plan-judge with the paths to idea.md and both plans. It writes:
- .claude/bracket-runs/<run-name>/spec.md, or conflict.md if it halts

**If plan-judge halts due to a genuine irreconcilable conflict: STOP HERE.** Read conflict.md and
end the invocation (per bracket-config.md) with state `HALTED: plan conflict`. The next steps lay
out the disagreement as a decision with the options. Wait for their decision, and don't proceed
further.

### Step 6 — Stop and report
**Do NOT dispatch builder or verifier.** This skill's job ends when spec.md is saved.

End the invocation per **Ending an invocation** in bracket-config.md, with state `SPEC READY`:
- "Why" becomes a one-to-two sentence summary of the chosen approach and the bias the judge applied.
- No Coverage or Built lines.
- Next steps: one step per Blocking Unknown marked `Needs: device` or `Needs: you` (what to have
  ready, e.g. a physical device, and that `/bracket:build-verify` will ask at its pre-flight about
  device access and any answers they've already seen), any decision the spec flagged for the user,
  then reviewing spec.md, then running `/bracket:build-verify <run-name>`. Note that the remaining
  Blocking Unknowns run automatically as spikes and gate the build.
