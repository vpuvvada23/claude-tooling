---
name: build-verify
description: Builds and verifies a feature from an existing bracket run's spec. Takes a run name, loads the judge spec from .claude/bracket-runs/<run-name>/spec.md, runs any Blocking Unknown spikes as a gate, dispatches builder then verifier, and presents the final summary. Use after /bracket:plan has produced a spec you're happy with.
argument-hint: "<run-name> [--agents role=model[:effort],...]"
---
> Agent names in this skill (plan-speed, plan-extensible, plan-judge, builder, verifier) are provided by the bracket plugin and may appear namespaced in the available-agents list (e.g. bracket:plan-speed). Dispatch using the name exactly as it appears in that list.


# Bracket — Build and Verify

## Purpose
Pick up an existing bracket run after planning is done. Reads the reconciled spec produced
by /bracket:plan (or /bracket:full), runs its Blocking Unknowns as spikes, runs the
builder against it, then the verifier (with at most one fix round), and presents the final
summary for the user's review.

## Instructions

Read `${CLAUDE_PLUGIN_ROOT}/bracket-config.md` first. It defines the default agent config,
the `--agents` override syntax, the pre-flight confirmation, how to pass model and effort on
each dispatch, the run-stats format, how to keep the terminal quiet, and how to end the invocation.
Follow it throughout this skill.

### Step 1 — Get the run name and overrides
Parse the skill's arguments: the run name and any `--agents ...` overrides. If the run name is
missing, ask for it. It must correspond to an existing folder at .claude/bracket-runs/<run-name>/ (see "Run
folder" in bracket-config.md for runs still under the old .claude/pipeline-runs/).

### Step 2 — Load and validate the spec
Read .claude/bracket-runs/<run-name>/spec.md.

**If spec.md does not exist: STOP.** Tell the user clearly:
> "No spec found at .claude/bracket-runs/<run-name>/spec.md. Run /bracket:plan first
> to generate a spec, or check that the run name is correct."
Do not proceed, do not improvise a plan, do not look for alternatives.

**If spec.md exists but appears to be a partial or conflict-halted output** (e.g. ends with
an unresolved conflict summary rather than a resolved spec): STOP and flag it to the user.

**If spec.md has no `## Blocking Unknowns` section** (e.g. it predates the spike gate): STOP and
tell the user. Offer to re-run planning, or to proceed with no gate if they confirm the spec
rests on no unverified external assumptions.

### Step 3 — Pre-flight
Resolve the config per bracket-config.md (defaults, then `--agents`). Don't read settings from
an existing run-stats.md, because it's a record only. Show the pre-flight agent config table with
only the roles this skill will dispatch: spike (only if the spec has Blocking Unknowns), builder,
and verifier. Ask the user to confirm or change it. Do not dispatch anything until they confirm.
Then start this invocation's section in the run stats record at
.claude/bracket-runs/<run-name>/run-stats.md.

### Step 4 — Spike gate (only if Blocking Unknowns is not "None.")
First write pre-build.md per **Pre-build snapshot** in bracket-config.md, so it records the code
before any builder pass.

Dispatch the builder in **spike-only mode** with spec.md, using the `spike` role's model and
effort. It writes:
- .claude/bracket-runs/<run-name>/spike-results.md

Read the report yourself. Do not rely only on its overall verdict line.
- **Every Blocking Unknown CONFIRMED** with observed evidence: continue to Step 5.
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

### Step 5 — Dispatch the builder
If pre-build.md doesn't exist yet (no spikes ran), write it first per **Pre-build snapshot** in
bracket-config.md.

Call the builder subagent in **build mode** with spec.md and, if it exists, spike-results.md,
using the `builder` role's model and effort. It writes:
- .claude/bracket-runs/<run-name>/change-summary.md

The builder does not stage, commit, or push anything — it only writes a change summary.

Then follow **Build gate** in bracket-config.md. Read the `## Spec Issues` section of
change-summary.md. If the builder is BLOCKED or errored, STOP: don't dispatch the verifier. End the
invocation with state `BUILD BLOCKED` (or `BUILD ERROR`), with each BLOCKED issue as a decision in
the next steps. Otherwise, carry any ASSUMED issues through to the ending.

### Step 6 — Dispatch the verifier (only if the build gate passed)
Call the verifier subagent in **verify mode** with spec.md, change-summary.md, spike-results.md (if it exists),
and pre-build.md.
It writes:
- .claude/bracket-runs/<run-name>/verification.md

Then follow **Fix round** in bracket-config.md. If verification.md has any fix requests, it runs
one builder fix pass and one re-verify, saved to reverification.md, then stops. Never more than one
fix round. Everything else goes to the user in Step 7.

### Step 7 — End with the summary and next steps
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
