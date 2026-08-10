# edc

Defensive tools for working with AI, distributed as a [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace named `loadout`. It carries two plugins, by weight:

- **`edc`** — *everyday carry.* One-shot prompt utilities, no agents.
- **`ghb`** — *get-home bag.* The heavier machinery: the engineering loops, adversarial review, and the agents they spawn.

## Install

```
/plugin marketplace add pm0u/edc
/plugin install edc@loadout      # everyday-carry utilities
/plugin install ghb@loadout      # engineering loops + reviews
```

CLI equivalents: `claude plugin marketplace add pm0u/edc`, `claude plugin install ghb@loadout`.

Commands are prefixed by their plugin: `/edc:un-ai`, `/ghb:sloop`.

## edc — everyday carry

Lightweight one-shot tools; install if you just want the utilities.

| Command | What it does |
|---------|--------------|
| `/edc:derive-spec` · `/edc:reverse-spec` | Derive a spec from a task, or reverse one out of existing code. |
| `/edc:trace-flow` · `/edc:mermaid` | Trace a data/execution flow; generate Mermaid diagrams. |
| `/edc:implementation-summary` · `/edc:ton-of-bricks` | Summarize the intent behind a change; reality-check validation. |
| `/edc:un-ai` · `/edc:resolve-pr-feedback` | Strip AI slop from a doc; address unresolved PR comments. |

## ghb — get-home bag

The frameworks and their kit. Everything here spawns agents (bundled with the plugin).

| Command | What it does |
|---------|--------------|
| [`/ghb:sloop`](plugins/ghb/docs/sloop.md) | The engineering loop. A dev agent builds against a spec, two adversarial reviewers attack in parallel, and a coordinator iterates with fresh dev agents until the code passes review. |
| [`/ghb:coop`](plugins/ghb/docs/coop.md) | The cooperative loop. A fork of sloop where the work is split into tiered slices ([Own/Co/Race](plugins/ghb/docs/tiers.md)) and you build the ones you need to understand while agents build the rest. |
| `/ghb:review-code` · `/ghb:review-plan` | Adversarial review of recent changes, or of a plan/design doc. |
| `/ghb:build-n-break` · `/ghb:review-summary` | Execute a task then adversarially review it; morning PR review summary. |

The agents (`loop-*` builders, `adversarial-*` reviewers) live in this one plugin, so nothing is shared or duplicated across plugins.

## Philosophy

AI tools are useful for bounded, verifiable tasks. They are bad at judgment, taste, and knowing when they're wrong. These tools are built around that reality:

- **Adversarial by default.** Assume AI output is wrong until proven otherwise.
- **Verification over generation.** The hard part isn't writing code -- it's knowing the code is correct.
- **Human judgment where it matters.** Route human attention to disputes and decisions, not scanning diffs.

Stay safe out there, meatbags.

## License

MIT
