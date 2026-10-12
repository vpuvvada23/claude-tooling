# claude-tooling

My personal Claude Code plugin marketplace, `vinnie-tools`. A marketplace is
just a catalog repo that Claude Code can install plugins from. It currently
holds one plugin, **dev-pipeline**: a plan → spike → build → verify workflow
that runs a feature idea through a panel of specialized subagents and leaves a
reviewable paper trail of every stage.

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

Every stage writes to `.claude/pipeline-runs/<run-name>/` in your project:
`idea.md`, `run-config.md`, `plan-speed.md`, `plan-extensible.md`, `spec.md`,
`spike-results.md`, `change-summary.md`, `verification.md`, `run-stats.md`.

## Quick start

**Requirements**
- [Claude Code](https://code.claude.com) (CLI, desktop app, or IDE extension).
  v2.1.292+ is required for per-run effort overrides.
- Add `.claude/pipeline-runs/` to each project's `.gitignore`.

**Install** (run inside Claude Code, not a shell):

```
/plugin marketplace add vpuvvada23/claude-tooling
/plugin install dev-pipeline@vinnie-tools
```

Restart Claude Code after installing so the agents and skills load.

**Use** (in any project):

```
/dev-pipeline-full share-sheet Add an iOS share extension that saves photos to the app's album
/dev-pipeline-plan share-sheet <idea>          # plan only; stop after spec.md
/dev-pipeline-build-verify share-sheet         # build + verify an existing spec
```

You can also just ask: "run the dev pipeline for share-sheet: …".

Before anything is dispatched, the skill asks any clarifying questions and shows
the agent config for you to confirm or change.

## Agent models and effort

Defaults live in [`plugins/dev-pipeline/pipeline-config.md`](plugins/dev-pipeline/pipeline-config.md):

| Role | Model | Effort |
|---|---|---|
| plan-speed, plan-extensible | `opus` | high |
| plan-judge | `opus` | xhigh |
| spike (builder, spike-only pass) | `opus` | high |
| builder | `sonnet` | high |
| verifier | `opus` | high |

Models are family aliases, so they track the latest release in each family.
The skills pass model and effort explicitly on every dispatch. Each agent's
frontmatter mirrors these defaults only as a fallback for direct use.

**Per-run overrides.** Pass `--agents` when starting a run:

```
/dev-pipeline-full share-sheet --agents builder=opus,verifier=sonnet:xhigh <idea>
/dev-pipeline-build-verify share-sheet --agents all=sonnet
```

The format is comma-separated `role=model[:effort]` (or `role=:effort`). `all`
sets every role, and specific roles win over it. The confirmed config is saved
to `run-config.md` and reused by later invocations for the same run.

**Run stats.** Every dispatch appends a row to `run-stats.md` with the requested
model and effort, tokens, tool uses, duration, and outcome. Each skill prints the
full table with totals when it finishes, including runs that halt early.

## Cost and usage

The defaults favor quality over cost. Five of the six roles run on **Opus**,
the judge at **xhigh** effort and the rest at **high**. Only the builder runs on
Sonnet. A full run dispatches five or six agents, so it uses a significant share
of your plan's usage limits, or API spend if you pay per token. Check
`run-stats.md` after a run to see where the tokens went.

To cut cost on a routine feature, override per run with `--agents` (syntax in
[`pipeline-config.md`](plugins/dev-pipeline/pipeline-config.md#per-run-overrides)):

```
# Everything on Sonnet
/dev-pipeline-full my-feature --agents all=sonnet <idea>

# Sonnet everywhere, but keep Opus for the judge and the spike gate
/dev-pipeline-full my-feature --agents all=sonnet,plan-judge=opus,spike=opus <idea>

# Keep the models, lower the effort
/dev-pipeline-full my-feature --agents all=:medium <idea>
```

The judge and the spike pass are where a weaker or rushed result costs the
most, because a missed risk there means building on an unverified premise. Keep
them strong before economizing anywhere else.

## Why I built it this way

- **Competing planners, not one.** A single planner tends to anchor on its
  first idea. Two planners with opposite goals surface the real tradeoffs, and
  the judge has to resolve them explicitly instead of inheriting one plan's
  blind spots.
- **Agreement isn't evidence.** In an early run, both planners chose the same
  approach without checking that the platform even supported it, and the build
  went ahead on an unverified premise. The spike gate exists because of that
  run: an assumption the whole feature depends on gets tested before the full
  build, not documented as a TODO.
- **An honest verdict.** "PASS with manual steps" hid a feature whose core
  premise had never run. UNCONFIRMED gives that state its own name, and the
  final summary leads with it.
- **Cheap where it's safe, strong where judgment matters.** Once the spec has
  made the decisions, implementation is the most mechanical step, so the
  builder runs on Sonnet. Planning, the judge, spikes, and verification run on
  Opus.
- **The human owns git.** Agents never stage, commit, or push. You review the
  spec, change summary, and verification report, then commit yourself.

## Known limitations

- **Claude Code only.** It relies on Claude Code subagents, plugin path
  variables, and skill arguments. It doesn't run in claude.ai chat or other AI
  coding tools.
- **Token-heavy by default.** See [Cost and usage](#cost-and-usage).
- **Model and effort are as requested.** Environment variables such as
  `CLAUDE_CODE_EFFORT_LEVEL` or `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`, org model
  allowlists, and effort clamping can change what actually runs. On Bedrock,
  Vertex, or Foundry, aliases may resolve to older models.
- **Stats are approximate.** Token counts exclude the orchestrating session.
  Durations are summed per agent, so parallel planners count twice. There's no
  cost estimate.
- **Spikes are only as good as the available automation.** Some checks really
  do need physical hardware. In that case the run halts and asks you rather than
  guessing.
- **No automatic retry.** A FAIL or UNCONFIRMED verdict stops for your decision.
  It never loops back to the builder on its own.
- **Older specs halt.** Specs written before v0.2.0 have no
  `## Blocking Unknowns` section, so build-verify stops on them. Re-plan, or
  confirm the spec has no unverified external assumptions.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/plugin marketplace add` fails | Check the repo name (`vpuvvada23/claude-tooling`) and your network. Update Claude Code (`claude update`) if `/plugin` isn't recognized. |
| Skills or agents don't appear after install | Restart Claude Code. |
| `--agents` effort overrides seem ignored | Check `claude --version` is v2.1.292+. Check for `CLAUDE_CODE_EFFORT_LEVEL` or `CLAUDE_CODE_SUBAGENT_MODEL_FORCE` in your environment. |
| Run files show up in `git status` | Add `.claude/pipeline-runs/` to the project's `.gitignore`. |
| New versions don't arrive | Run `/plugin marketplace update vinnie-tools`, then update the plugin from `/plugin` (or `claude plugin update dev-pipeline@vinnie-tools` in a shell), and restart. |
| build-verify stops: no Blocking Unknowns section | The spec predates v0.2.0. Re-run `/dev-pipeline-plan`, or confirm when prompted. |

## Disclaimer

This is an independent project. It is not affiliated with, endorsed by, or
supported by Anthropic. The pipeline's agents run commands and modify files in
your project, so review their output before committing. Provided as-is under
the [MIT License](LICENSE), with no warranty.

## License

[MIT](LICENSE)
