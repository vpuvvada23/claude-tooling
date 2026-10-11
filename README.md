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

## Install

```
/plugin marketplace add vinnie/claude-tooling     # or the local path ~/dev/claude-tooling
/plugin install dev-pipeline@vinnie-tools
```
