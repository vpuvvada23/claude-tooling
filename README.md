# claude-tooling

My personal Claude Code plugin marketplace, `vinnie-tools`. A marketplace is
just a catalog repo that Claude Code can install plugins from.

## Plugins

| Plugin | What it does |
|---|---|
| [**Bracket Dev Kit**](plugins/bracket/README.md) (`bracket`) | Competing planners, a judge, a builder, and a verifier for Claude Code. Unverified assumptions must pass a spike before the build, and every run ends with an honest PASS / UNCONFIRMED / FAIL verdict. |

See the [Bracket README](plugins/bracket/README.md) for usage, what it runs,
models and cost, and limitations.

## Install

In a terminal, one command:

```
claude plugin install bracket --marketplace vpuvvada23/claude-tooling
```

Or inside Claude Code:

```
/plugin marketplace add vpuvvada23/claude-tooling
/plugin install bracket@vinnie-tools
```

`vinnie-tools` is the marketplace name from `.claude-plugin/marketplace.json`,
not a GitHub user. Restart Claude Code after installing.

**Updating:** run `/plugin marketplace update vinnie-tools`, then update the
plugin from `/plugin` (or `claude plugin update bracket@vinnie-tools` in a
shell), and restart.

## License

[MIT](LICENSE). An independent project, not affiliated with or endorsed by
Anthropic.
