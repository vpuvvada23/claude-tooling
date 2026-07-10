# claude-tooling

Personal Claude Code plugin marketplace (`vinnie-tools`). Project-agnostic
dev tooling shared across projects ([origin project], [another project], …).

## Plugins

### dev-pipeline

Plan-build-verify agent pipeline:

- **Agents**: `plan-speed` and `plan-extensible` draft competing implementation
  plans, `plan-judge` reconciles them into a spec, `builder` implements the
  spec, `verifier` smoke-tests the result.
- **Skills**: `dev-pipeline-full` (plan → build → verify), `dev-pipeline-plan`
  (stop after spec), `dev-pipeline-build-verify` (build from an existing spec).
- Run artifacts are written to `.claude/pipeline-runs/<run-name>/` in the
  project the skill runs in (gitignore that path per project).

## Install

```
/plugin marketplace add vinnie/claude-tooling     # or the local path ~/dev/claude-tooling
/plugin install dev-pipeline@vinnie-tools
```

Extracted from the [origin project] repo (origin commit 23426f7).
