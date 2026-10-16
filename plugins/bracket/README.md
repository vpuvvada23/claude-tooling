# Bracket Dev Kit

Competing planners, a judge, a builder, and a verifier for Claude Code. Bracket
runs a feature idea through a plan → spike → build → verify pipeline of
specialized subagents. Unverified assumptions must pass a real experiment before
anything gets built, every run ends with an honest verdict, and each stage leaves
a reviewable file in your project.

## What it does

```
idea ─┬─► plan-speed ──────┐
      └─► plan-extensible ─┴─► plan-judge ─► spec.md ─► spike gate ─► builder ─► verifier
                                                                         ▲           │
                                                                         └ fix round ┘
```

- **Two competing planners.** `plan-speed` drafts the minimal path to working
  code, and `plan-extensible` drafts the future-proof version. Each lists the
  external/platform assumptions its plan depends on and whether they're
  verified.
- **A judge** reconciles the two into one spec. Its bias follows the run's
  direction, which Bracket asks about up front when it isn't clear: speed for
  a prototype (except for hard-to-undo choices like the data model),
  extensibility along your stated expansion plans for a growing project, and
  extensibility unless speed wins clearly when you're not sure. It promotes
  every unverified, load-bearing assumption to a **Blocking Unknown**, with a
  concrete test and pass criterion.
- **A spike gate.** Before the full build, the builder runs each Blocking
  Unknown as a real experiment, and stops as soon as its pass criterion is
  observed. If any aren't confirmed with observed evidence, the run stops. The
  spike leaves its working setup (a scratch build, test helpers, the commands
  that worked) for the verifier to reuse.
- **A builder** implements the spec exactly and writes a change summary. It
  never stages, commits, or pushes. When the spec leaves a small detail open,
  it makes a minimal assumption and flags it. When the spec's approach won't
  work or needs a bigger decision, it stops. The verifier runs only on a
  finished build, and every assumption or blocker is a step in the summary at
  the end, not just a note in the change summary.
- **A verifier** checks git tracking, spec conformance, and runtime behavior.
  It must try automation (simulators, headless browsers, curl) before calling
  anything "manual only". Its verdict is **PASS**, **FAIL**, or
  **UNCONFIRMED**. UNCONFIRMED means the code is complete but its core premise
  is still untested. A Coverage line under the verdict says what passed where
  and what still needs testing, e.g. "Simulator: PASS · Device: NEEDS TESTING".
  It tests and never edits your code, and it sorts each failure by who owns the
  fix.
- **One fix round.** When the spec was clear and the build got it wrong, or
  the build introduced a bug that has an obvious fix (like breaking something
  that used to work), the verifier hands the builder numbered fix requests with
  repro steps. The builder
  fixes only those, and the verifier re-runs every check to catch regressions.
  Then the run stops. Spec gaps and unconfirmed assumptions go to you, never
  back to the builder.

## Requirements

- **Claude Code** (CLI, desktop app, or IDE extension). Bracket relies on
  Claude Code subagents, plugin path variables, and skill arguments. It isn't
  designed for claude.ai chat or Cowork.
  v2.1.292+ is required for per-run effort overrides.
- Add `.claude/bracket-runs/` to each project's `.gitignore`.

## Install

In a terminal:

```
claude plugin install bracket --marketplace vpuvvada23/claude-tooling
```

Or inside Claude Code:

```
/plugin marketplace add vpuvvada23/claude-tooling
/plugin install bracket@vinnie-tools
```

Restart Claude Code after installing. To update later, run
`claude plugin marketplace update vinnie-tools`, then
`claude plugin update bracket@vinnie-tools`, and restart.

## Usage

```
/bracket:full share-sheet Add an iOS share extension that saves photos to the app's album
/bracket:plan share-sheet <idea>          # plan only; stop after spec.md
/bracket:build-verify share-sheet         # build + verify an existing spec
```

You can also just ask: "run bracket for share-sheet: …".

**When not to use it.** Bracket is built for features with real decisions and
real unknowns. For a bug fix, a copy change, or a small UI tweak, it says so
before starting and recommends building it directly with a quick smoke test.
The choice is yours: say you want bracket anyway and it carries on.

Before anything is dispatched, Bracket asks any clarifying questions and shows a
**pre-flight agent config** (role, model, effort) for the agents it's about to
run. Confirm it or change it there. Everything that needs you is asked up front,
so the run doesn't stop halfway to wait for you:

- **Device access.** If your project runs on physical hardware, Bracket asks
  once whether agents may install development builds on your device and drive
  the app there. Yes covers this app only: never system settings, other apps,
  or your content outside the app. With no, device checks become manual steps.
- **Answers you already have.** If a Blocking Unknown needs you or your device,
  Bracket asks whether you've already seen the answer, e.g. "you've toggled
  this before: did iOS show a dialog?" What you've observed counts as the
  result, and that spike doesn't run.

While it runs, the terminal stays quiet: one line per stage, with progress
inside it:

```
→ stage 5 of 5 · verifier running (opus, high)
  verifier · ~25% · build and unit tests passed
  verifier · ~50% · Simulator checks 1–8 of 16 done
  verifier · ~75% · all Simulator checks done, writing report
✓ UNCONFIRMED · sim PASS · device NEEDS TESTING
```

The percentages count each agent's own steps, not time. Each agent writes its
own report to the run folder and hands back a short status. Every command
ends with a short summary (verdict, coverage, and why) followed by a numbered
**Your next steps** list: what you need to do, in order, and why each step
matters. For example, the device checks that would turn UNCONFIRMED into PASS,
or a decision the spec left to you. The full reports stay in the run folder for
when you want the detail.

## What it runs, writes, and sends

Bracket is plain markdown prompts: no hooks, MCP servers, scripts, or
dependencies. What its agents do in your project:

- **Run shell commands** in your project to build and verify: `git status` and
  `git check-ignore` (read-only), `git stash create` before the build (it records
  your uncommitted changes as a commit object and changes nothing in your working
  tree, index, or stash list), a temporary `git worktree` outside your project
  for comparing a failing test against the code from before the build (removed
  afterward), your project's own dev/build/test commands,
  and, for spikes and verification, local automation such as iOS simulator
  tooling (`xcrun simctl`, `idb`), Android `adb`, Playwright or other headless
  browsers, `curl` against a local server, and throwaway scripts. Every command
  runs under your Claude Code permission settings.
- **Modify your code.** The builder implements the approved spec and, in the
  fix round, the verifier's fix requests. The verifier never modifies it.
- **Write run files** to `.claude/bracket-runs/<run-name>/`: `idea.md`,
  `plan-speed.md`, `plan-extensible.md`, `spec.md`, `pre-build.md`,
  `spike-results.md`, `change-summary.md`, `verification.md`,
  `reverification.md` (after a fix round), `conflict.md` (if the judge halts),
  `run-stats.md`.
- **Never** stages, commits, or pushes, and never sends data to any service
  other than the Claude models your Claude Code session already uses. It reads
  no credentials or environment variables.

## Models, effort, and cost

Defaults (from [`bracket-config.md`](bracket-config.md)):

| Role | Model | Effort |
|---|---|---|
| plan-speed, plan-extensible | `opus` | medium |
| plan-judge | `opus` | high |
| spike (builder, spike-only pass) | `sonnet` | high |
| builder | `sonnet` | high |
| verifier | `opus` | high |

Models are family aliases, so they track the latest release in each family.
The defaults put Opus where judgment matters (planning, the judge, and the
verifier deciding who owns each failure) and Sonnet where the work is hands-on
(the spike and the build). A full run dispatches five or six agents, so it still
uses a significant share of your plan's usage limits, or API spend if you pay
per token.

Override per run with `--agents` (comma-separated `role=model[:effort]`, `all`
sets every role, and specific roles win):

```
# Everything on Sonnet
/bracket:full my-feature --agents all=sonnet <idea>

# Sonnet everywhere, but keep Opus for the judge and the verifier
/bracket:full my-feature --agents all=sonnet,plan-judge=opus,verifier=opus <idea>

# More thinking for a hard feature
/bracket:full my-feature --agents plan-judge=:xhigh,spike=opus <idea>
```

The judge and the verifier are where a weaker result costs the most. Keep
them strong before economizing anywhere else. Most of a run's time goes to
builds, simulators, and tests rather than thinking, so the spec's checklist
(what the verifier must drive versus what a code read covers) matters more for
run time than effort does.

`run-stats.md` records, for each command, the settings you confirmed and each
dispatch's tokens, duration, and outcome. When a command finishes, it prints a
one-line run cost total. Open `run-stats.md` for the per-dispatch breakdown.
A Current state block at its top shows where the run stands now.

Results that arrive after an agent finishes, like a device session you run
by hand, update the run's files in place. The summaries at the top of
`spike-results.md` and `verification.md`, the outcome in `run-stats.md`, and its
Current state block all change to match, so nothing still says NOT RUN once
it has run.

## Why it's built this way

- **Competing planners, not one.** A single planner tends to anchor on its
  first idea. Two planners with opposite goals surface the real tradeoffs, and
  the judge has to resolve them explicitly.
- **Agreement isn't evidence.** In an early run, both planners chose the same
  approach without checking that the platform even supported it, and the build
  went ahead on an unverified premise. The spike gate exists because of that
  run.
- **An honest verdict.** "PASS with manual steps" hid a feature whose core
  premise had never run. UNCONFIRMED gives that state its own name, and the
  final summary leads with it.
- **Cheap where it's safe, strong where judgment matters.** Once the spec has
  made the decisions, implementation is the most mechanical step, so the
  builder runs on Sonnet. So does the spike, which runs experiments the judge
  already designed.
- **Ask early, not halfway.** Questions only you can answer, like device access
  or something you've already seen on your phone, come at the start. A run that
  stops halfway to wait for you costs your time as well as the agents'.
- **The human owns git.** You review the spec, change summary, and
  verification report, then commit yourself.

## Known limitations

- **Model and effort are as requested.** Environment variables such as
  `CLAUDE_CODE_EFFORT_LEVEL` or `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`, org model
  allowlists, and effort clamping can change what actually runs. On Bedrock,
  Vertex, or Foundry, aliases may resolve to older models.
- **Stats are approximate.** Token counts exclude the orchestrating session.
  Durations are summed per agent, so parallel planners count twice.
- **Spikes are only as good as the available automation.** Some checks really
  do need physical hardware. Without device access, the run halts and asks you
  rather than guessing. Even with it, Claude Code's own permission settings and
  safety checks can block a device step. Agents don't work around them: the step
  becomes a manual one.
- **Progress lines need background agents.** Agents send them as messages to
  your session. An agent run in the foreground can't, so its stage shows only
  the start and end lines.
- **The spike's setup lasts one session.** It lives in the session's scratchpad.
  If you continue a run in a new session, the verifier rebuilds it.
- **One fix round only.** Build defects get one automatic fix round. Anything
  still failing after it, and every spec issue or unconfirmed assumption,
  stops for your decision.
- **Older specs halt.** Specs written before v0.2.0 have no
  `## Blocking Unknowns` section, so build-verify stops on them. Re-plan, or
  confirm the spec has no unverified external assumptions.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Commands or agents don't appear after install | Restart Claude Code. |
| `--agents` effort overrides seem ignored | Check `claude --version` is v2.1.292+. Check for `CLAUDE_CODE_EFFORT_LEVEL` or `CLAUDE_CODE_SUBAGENT_MODEL_FORCE` in your environment. |
| Run files show up in `git status` | Add `.claude/bracket-runs/` to the project's `.gitignore`. |
| build-verify stops: no Blocking Unknowns section | The spec predates v0.2.0. Re-run `/bracket:plan`, or confirm when prompted. |

## Disclaimer

Bracket is an independent project. It is not affiliated with, endorsed by, or
supported by Anthropic. Its agents run commands and modify files in your
project, so review their output before committing. Provided as-is under the
[MIT License](LICENSE), with no warranty.
