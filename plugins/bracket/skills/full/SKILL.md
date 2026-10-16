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
tight — only ask what's actually needed to plan, not exhaustive discovery. If the idea is
already clear and complete, say so.

Also settle the run's **direction**, unless the idea or project already makes it clear (e.g. "quick
test of…", a scratch repo, a stated roadmap):
- **prototype** — a throwaway or market test; no long-term plans yet
- **growing** — expected to expand; ask how (the specific directions, e.g. "add Android next",
  "multi-user later")
- **not sure** — a fine answer; don't push for one

If you inferred it rather than asked, state it so the user can correct it.

In the same message, show the pre-flight agent config table (per bracket-config.md) with all
six roles, since this skill can dispatch all of them. Below the table, note that spike only runs
if the spec has Blocking Unknowns. If the project targets physical hardware, ask the **device
access** question too (see **Device access and answers you already have** in bracket-config.md).
Ask the user to confirm or change it. Do not dispatch anything until all of it is settled.

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
out the disagreement as a decision with the options. Wait for their decision before proceeding.
Do not dispatch the builder.

### Step 6 — Spike gate (only if spec.md exists and is a resolved spec)
Read spec.md first. If spec.md does not exist: STOP and tell the user. If spec.md appears to be a
partial or conflict-halted output (e.g. ends with an unresolved conflict summary): STOP and flag
it to the user. Do not improvise.

If the spec's `## Blocking Unknowns` section is anything other than "None.":
1. If any Blocking Unknown is `Needs: you`, or `Needs: device` without device access, ask the user
   whether they've already observed its answer (see **Answers the user already has** in
   bracket-config.md), and wait for the reply. Otherwise don't stop: go straight on.
2. Write pre-build.md per **Pre-build snapshot** in bracket-config.md.
3. Dispatch the builder in **spike-only mode** with spec.md, the device access answer, and any
   answers the user gave, using the `spike` role's model and effort. It writes:
- .claude/bracket-runs/<run-name>/spike-results.md

Read the report yourself. Do not rely only on its overall verdict line.
- **Every Blocking Unknown CONFIRMED** with observed evidence: continue to Step 7.
- **Any REFUTED or NOT RUN: STOP.** Do not dispatch the full build. End the invocation (per
  bracket-config.md) with state `GATE BLOCKED`. "Why" names each BU that isn't confirmed and what
  was observed. The next steps cover how to run each NOT RUN spike, the spec's "If it fails"
  fallback for each REFUTED one, and the decision: amend the spec, run the spike themselves, or
  abandon the run.

If the spikes are run later (by the user, or by you with them, for example on a device), or the
spec is amended in response, follow **Keeping the record current** in bracket-config.md before
continuing. spike-results.md's summary, its BU statuses, and the spike row in run-stats.md must
show the new results, not the original NOT RUN.

A spike is not confirmed by a template, a placeholder findings doc, or an assumption written into
code. If a "CONFIRMED" entry doesn't show what was actually run and observed, treat it as NOT RUN.

### Step 7 — Dispatch the builder
If pre-build.md doesn't exist yet (no spikes ran), write it first per **Pre-build snapshot** in
bracket-config.md.

Call builder in **build mode** with spec.md and, if it exists, spike-results.md, using the
`builder` role's model and effort. The builder does not stage, commit, or push anything — it only
writes a change summary, to:
- .claude/bracket-runs/<run-name>/change-summary.md

Then follow **Build gate** in bracket-config.md. Read the `## Spec Issues` section of
change-summary.md. If the builder is BLOCKED or errored, STOP: don't dispatch the verifier. End the
invocation with state `BUILD BLOCKED` (or `BUILD ERROR`), with each BLOCKED issue as a decision in
the next steps. Otherwise, carry any ASSUMED issues through to the ending.

### Step 8 — Dispatch the verifier (only if the build gate passed)
Call the verifier subagent in **verify mode** with spec.md, change-summary.md, spike-results.md (if it exists),
pre-build.md, and the device access answer. Point it at spike-results.md's `## Reusable setup`.
It writes:
- .claude/bracket-runs/<run-name>/verification.md

Then follow **Fix round** in bracket-config.md. If verification.md has any fix requests, it runs
one builder fix pass and one re-verify, saved to reverification.md, then stops. Never more than one
fix round. Everything else goes to the user in Step 9.

### Step 9 — End with the summary and next steps
Do NOT commit or push anything. End the invocation per **Ending an invocation** in
bracket-config.md, using the last verifier report (reverification.md if a fix round ran). Don't
let a verdict label carry the meaning. "Why" says plainly what is unconfirmed or broken, e.g.
"Structurally complete, but the core premise (BU-1: Photos lists the extension in its share sheet)
is still untested. Until it is, we don't know the feature works at all."

If the user later reports results (for example a `## Manual Verification` or a device session),
update the record per **Keeping the record current** in bracket-config.md, then reply in the
same ending shape, with the verdict and next steps as they stand now.

If the user later reports the feature isn't working, reach for the available automation
(simulator tooling, browser automation, curl, logs) to reproduce and debug it yourself before
asking them to run manual test rounds.
