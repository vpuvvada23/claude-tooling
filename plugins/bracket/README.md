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
```

- **Two competing planners.** `plan-speed` drafts the minimal path to working
  code, and `plan-extensible` drafts the future-proof version. Each lists the
  external/platform assumptions its plan depends on and whether they're
  verified.
- **A judge** reconciles the two into one spec. It defaults to extensibility
  unless speed wins clearly. It promotes every unverified, load-bearing
  assumption to a **Blocking Unknown**, with a concrete test and pass criterion.
- **A spike gate.** Before the full build, the builder runs each Blocking
  Unknown as a real experiment. If any aren't confirmed with observed evidence,
  the run stops.
- **A builder** implements the spec exactly and writes a change summary. It
  never stages, commits, or pushes.
- **A verifier** checks git tracking, spec conformance, and runtime behavior.
  It must try automation (simulators, headless browsers, curl) before calling
  anything "manual only". Its verdict is **PASS**, **FAIL**, or
  **UNCONFIRMED**. UNCONFIRMED means the code is complete but its core premise
  is still untested.

## Requirements

- **Claude Code** (CLI, desktop app, or IDE extension). Bracket relies on
  Claude Code subagents, plugin path variables, and skill arguments. It isn't
  designed for claude.ai chat or Cowork.
  v2.1.292+ is required for per-run effort overrides.
- Add `.claude/bracket-runs/` to each project's `.gitignore`.

## Usage

```
/bracket:full share-sheet Add an iOS share extension that saves photos to the app's album
/bracket:plan share-sheet <idea>          # plan only; stop after spec.md
/bracket:build-verify share-sheet         # build + verify an existing spec
```

You can also just ask: "run bracket for share-sheet: …".

Before anything is dispatched, Bracket asks any clarifying questions and shows a
**pre-flight agent config** (role, model, effort) for the agents it's about to
run. Confirm it or change it there.

## What it runs, writes, and sends

Bracket is plain markdown prompts: no hooks, MCP servers, scripts, or
dependencies. What its agents do in your project:

- **Run shell commands** in your project to build and verify: `git status` and
  `git check-ignore` (read-only), your project's own dev/build/test commands,
  and, for spikes and verification, local automation such as iOS simulator
  tooling (`xcrun simctl`, `idb`), Android `adb`, Playwright or other headless
  browsers, `curl` against a local server, and throwaway scripts. Every command
  runs under your Claude Code permission settings.
- **Modify your code.** The builder implements the approved spec.
- **Write run files** to `.claude/bracket-runs/<run-name>/`: `idea.md`,
  `plan-speed.md`, `plan-extensible.md`, `spec.md`,
  `spike-results.md`, `change-summary.md`, `verification.md`, `run-stats.md`.
- **Never** stages, commits, or pushes, and never sends data to any service
  other than the Claude models your Claude Code session already uses. It reads
  no credentials or environment variables.

## Models, effort, and cost

Defaults (from [`bracket-config.md`](bracket-config.md)):

| Role | Model | Effort |
|---|---|---|
| plan-speed, plan-extensible | `opus` | high |
| plan-judge | `opus` | xhigh |
| spike (builder, spike-only pass) | `opus` | high |
| builder | `sonnet` | high |
| verifier | `opus` | high |

Models are family aliases, so they track the latest release in each family.
The defaults favor quality over cost. A full run dispatches five or six agents,
mostly on Opus, so it uses a significant share of your plan's usage limits, or
API spend if you pay per token.

Override per run with `--agents` (comma-separated `role=model[:effort]`, `all`
sets every role, and specific roles win):

```
# Everything on Sonnet
/bracket:full my-feature --agents all=sonnet <idea>

# Sonnet everywhere, but keep Opus for the judge and the spike gate
/bracket:full my-feature --agents all=sonnet,plan-judge=opus,spike=opus <idea>

# Keep the models, lower the effort
/bracket:full my-feature --agents all=:medium <idea>
```

The judge and the spike pass are where a weaker result costs the most. Keep
them strong before economizing anywhere else.

`run-stats.md` records, for each command, the settings you confirmed and each
dispatch's tokens, duration, and outcome. Each command prints the stats table with totals when it finishes.

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
  builder runs on Sonnet.
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
  do need physical hardware. In that case the run halts and asks you rather than
  guessing.
- **No automatic retry.** A FAIL or UNCONFIRMED verdict stops for your decision.
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
