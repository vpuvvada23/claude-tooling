# claude-tooling

Personal Claude Code plugin marketplace (`vinnie-tools`). Project-agnostic
dev tooling shared across projects.

## Plugins

### dev-pipeline

Plan-build-verify agent pipeline:

- **Agents**: `plan-speed` and `plan-extensible` draft competing implementation
  plans, `plan-judge` reconciles them into a spec (including any Blocking
  Unknowns), `builder` runs those as spikes and then implements the spec,
  `verifier` smoke-tests the result (PASS / UNCONFIRMED / FAIL).
- **Spike gate**: unverified, load-bearing platform assumptions in the spec must
  be confirmed by a spike before the full build is dispatched.
- **Skills**: `dev-pipeline-full` (plan → spike → build → verify), `dev-pipeline-plan`
  (stop after spec), `dev-pipeline-build-verify` (build from an existing spec).
- Run artifacts are written to `.claude/pipeline-runs/<run-name>/` in the
  project the skill runs in (gitignore that path per project).

#### Agent models and effort

Defaults live in [`plugins/dev-pipeline/pipeline-config.md`](plugins/dev-pipeline/pipeline-config.md):

| Role | Model | Effort |
|---|---|---|
| plan-speed, plan-extensible | `opus` | high |
| plan-judge | `opus` | xhigh |
| spike (builder, spike-only pass) | `opus` | high |
| builder | `sonnet` | high |
| verifier | `opus` | high |

Models are family aliases, so they track the latest release in each family.
The skills pass model and effort explicitly on every dispatch; each agent's
frontmatter mirrors these defaults only as a fallback for direct use (keep the
two in sync).

**Per-run overrides.** Pass `--agents` when starting a run:

```
/dev-pipeline-full photo-share --agents builder=opus,verifier=sonnet:xhigh <idea>
/dev-pipeline-build-verify photo-share --agents all=opus
```

Format is comma-separated `role=model[:effort]` (or `role=:effort`); `all` sets
every role, and specific roles win over it. Before dispatching anything, each
skill shows the resolved config for you to confirm or edit. The confirmed config
is saved to `run-config.md` in the run folder and reused by later invocations
for the same run.

**Run stats.** Every dispatch appends a row to `run-stats.md` (requested model
and effort, tokens, tool uses, duration, outcome), and each skill prints the
full table with totals when it finishes, including runs that halt early.
Model and effort are as requested; environment overrides or effort clamping can
change what actually ran. Token counts exclude the orchestrating session.

## Install

Run these inside Claude Code (not a shell):

```
/plugin marketplace add vpuvvada23/claude-tooling     # or the local path ~/dev/claude-tooling
/plugin install dev-pipeline@vinnie-tools
```

`vinnie-tools` is the marketplace name from `.claude-plugin/marketplace.json`,
not a GitHub user.

**Private repo access.** The repository is private. Claude Code clones it with
your normal git credentials (SSH key, `gh auth login`, or a credential helper),
so anyone installing it needs read access (be added as a collaborator) and
working GitHub credentials. Background auto-updates may need a token such as
`GITHUB_TOKEN` in the environment; if updates don't arrive, run
`/plugin marketplace update vinnie-tools` manually.

**Requirements**

- Claude Code v2.1.292 or later (for per-run `--agents` overrides).
- Add `.claude/pipeline-runs/` to each project's `.gitignore`.
