# edc

Defensive tools for working with AI. Designed for [OpenPackage](https://openpackage.dev).

## Install

```bash
opkg install gh@pm0u/edc
```

## Agents

| Agent | Description |
|-------|-------------|
| `adversarial-plan-reviewer` | Stress-tests implementation plans. Pokes holes in assumptions, identifies risks, and challenges feasibility before work begins. |
| `adversarial-code-reviewer` | Hunts for AI-generated failure modes in code changes. Targets reckless completion patterns, fabricated assumptions, scope violations, and engineering judgment issues. |
| `adversarial-architecture-reviewer` | Reviews code changes at the system level — placement, pattern consistency, complexity budget, and direction. Pairs with the code reviewer for big-picture coverage. |
| `loop-researcher` | Codebase researcher for the engineering loops (`/sloop`, `/coop`). Explores the code and returns a distilled brief so the coordinator writes the spec from fact, not guesswork. |
| `loop-dev` | Implementation agent for the engineering loops (`/sloop`, `/coop`). Builds against a spec in phases, verifies for real, and reports honestly what wasn't verified. |
| `loop-verifier` | Verification agent for the engineering loops (`/sloop`, `/coop`). Reproduces the dev agent's Build Report claims — re-runs what it says was verified, attempts what it says wasn't. |
| `distill-learnings` | Extracts reusable lessons from completed work. |

## Commands

| Command | Description |
|---------|-------------|
| `/sloop` | Engineering loop: a plan reviewer stress-tests the spec at handoff, a dev agent builds, two adversarial reviewers attack in parallel, and a coordinator iterates with fresh dev agents until the code passes review. See [docs/sloop.md](docs/sloop.md). |
| `/coop` | Cooperative engineering loop: a fork of `/sloop` where the work is split into tiered slices and you build the ones you need to understand while agents build the rest, all under the same adversarial review. See [docs/coop.md](docs/coop.md). |
| `/build-n-break` | Execute a task then adversarially review the result. You review disputes, not the full diff. |
| `/review-plan` | Run an adversarial review against a plan or design doc. |
| `/review-code` | Run an adversarial code review against recent changes. |
| `/implementation-summary` | Generate a summary of what was built and why -- intent and decisions, not a diff listing. |
| `/ton-of-bricks` | Reality-check validation. Forces real-world verification instead of trusting tests and types. |
| `/resolve-pr-feedback` | Fetch and address unresolved PR review comments. |
| `/derive-spec` | Derive a specification from a task description. |
| `/reverse-spec` | Reverse-engineer a specification from existing code. |
| `/trace-flow` | Trace a data or execution flow through the codebase. |
| `/mermaid` | Generate Mermaid diagrams from code or descriptions. |
| `/un-ai` | Strip AI slop from a document. Extract what it actually says in plain language. |

## Philosophy

AI tools are useful for bounded, verifiable tasks. They are bad at judgment, taste, and knowing when they're wrong. These tools are built around that reality:

- **Adversarial by default.** Assume AI output is wrong until proven otherwise.
- **Verification over generation.** The hard part isn't writing code -- it's knowing the code is correct.
- **Human judgment where it matters.** Route human attention to disputes and decisions, not scanning diffs.

Stay safe out there, meatbags.

## License

MIT
