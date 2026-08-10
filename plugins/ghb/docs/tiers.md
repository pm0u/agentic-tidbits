# Tiers — where your understanding budget goes

The rubric `coop` uses to decide, per slice of a change, who builds it and how much
you need to understand it afterward. It's the single source of truth: `coop.md`
references it, and the coordinator injects it into every builder and plan-reviewer
prompt. The rubric lives _here_ and nowhere else, so it can't drift.

> Tier is a property of the needed code, and specifically a divisible unit of the code, that is needed to accomplish the task. The goal is to create a shared understanding between agents and the human and the framework serves as a guidance for how to determine what is important to force onto the human.

## The unit: slices and shells

Tiering operates on **slices**, and slices are separated by **shells**. They're the
same structure seen from two sides — slices are the regions, shells are the boundaries
between them.

A **slice** is the smallest unit of a change that (a) carries a single tier and (b) can
be built by one builder against the shells of its neighbors. Its defining property is
**tier-homogeneity**: if a candidate slice is half Own and half Race, it isn't one slice,
it's two — and the split happens _at a seam_.

A **shell** is that seam made explicit: a signature plus a one-line behavior contract
that both sides agree on before either builds.

```
function fn(ctx: Context, action: Action): FunctionReturnType { /* body filled by owner */ }
```

Whoever owns the higher-tier side of a seam writes the shell; the other side fills the
body. A good shell states its inputs, its output, and the one guarantee a caller can rely
on — enough that the other side can build against it without asking. A stable shell is what
lets two builders work adjacent slices in parallel without blocking on each other — and an
unstable one (the contract keeps changing as you build) is the signal to stop parallelizing
and sequence instead.

## The three tiers

Every slice is one of these. Tier is set in Phase 0 and can be challenged by the plan
reviewer. Read the table down a column to get a tier's profile; read across a row to
compare the three on one thread. *Qualifies when* is a one-line pointer — the canonical
qualifier list is the **Concrete anchors** under [Classifying a slice](#classifying-a-slice).

| Thread                   | Own                                                                        | Co                                               | Race                                                                                |
| ------------------------ | -------------------------------------------------------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------------- |
| **Qualifies when**       | high blast radius; core or hard to verify | real logic you'll revisit; moderate stakes | throwaway or a known pattern |
| **Understanding target** | write or re-derive it; extend, debug, explain it cold                      | the contract and the seams, not every line       | none — behavior only                                                                |
| **Default builder**      | you                                                                        | agent; you review actively (predict-before-peek) | agent                                                                               |
| **Verification bar**     | human comments that show understanding or discussion with agent to prove   | predict-before-peek review, then it merges       | tests pass; behavior exercised                                                      |
| **Your review**          | re-derive it — reconstruct the reasoning, don't just read it               | predict-before-peek, then accept                 | confirm it works, move on                                                           |

_Your review_ is what _you_ do; it only bites when an agent built the slice — for an Own
slice you built yourself, the re-deriving was the building. Agent (adversarial) review is
not in this table: it runs at full depth on every tier, and that constant lives in `coop.md`.

## Classifying a slice

The **Qualifies when** row above is the quick profile; these are the signals you weigh to
place a slice there. Fast, not a checklist to grind through:

The signals, roughly in weight order — when they conflict, blast radius wins:

- **Blast radius** — if your model of it is wrong, is the cost a lint fix or a 2am page? The dominant signal; it's the cost of being wrong.
- **Task fit** — how hard is it for a human or agent to get feedback and iterate? Code an agent can't verify itself leans Own.
- **Core vs. periphery** — domain logic, invariants, shared state? Or glue and plumbing? Usually tracks blast radius, but not always: a config file is periphery with high blast radius.
- **Return frequency** — will you touch this region again, or is it write-once?
- **Novelty** — new to you, or a pattern you've written many times? Scales how much *you specifically* gain from owning it.

**The test** (one line, so it's not endless reasoning):

> Blast radius is the spine. High blast radius + (hard to verify, core, novel, or you'll
> revisit) → **Own**. Low blast radius + periphery + known pattern → **Race**. Everything
> between → **Co**. Genuinely unsure → default **Co** — the bubble-up rule backstops an
> under-tiered slice, so you don't need to over-tier defensively.

**Concrete anchors** (worked examples of each tier):

> Own: Difficult for agent to verify, core logic or coordination, large blast radius
>
> Co: Testing around _Own_ code, logic central to overall application function but not complex, specific asks/edits from operator
>
> Race: boilerplate, setup, copy/paste, refactor, reorganize, small bug fixes, text edits

## Who builds a tier vs. what tier it is

Classification (above) is a property of the code. _Who builds each tier_ is a separate
knob set per run by `coop`'s `--escalate` flag — you can take on more than your default
(Own only) up to building everything (codebase-familiarization mode). Classification
still runs regardless; it drives review intensity, shell authorship, and ordering even
when you're not the one building.

## Bubble-up — catching a mis-tiered slice mid-build

Phase 0 tiering is a guess and will sometimes be wrong. A build agent that starts a
slice tagged Co or Race and finds it's actually Own (touches an invariant, higher blast
radius than it looked) must flag it rather than race through — the coordinator surfaces
it to you.

A slice tagged Co or Race is actually Own the moment building it means any of:

- **You're touching a seam, not an interior** — changing a shared invariant, shared state, or a contract other slices or callers depend on. The slice turned out to be a shell.
- **The blast radius is bigger than the tag** — a wrong result here fails silently or corrupts data rather than throwing loudly, or many callers depend on the exact behavior.
- **It hinges on a 2am-class decision** — correctness turns on a subtle, non-obvious call: concurrency, ordering, auth, money/units, a data migration.
- **You'd be guessing at intent** — the spec doesn't pin a decision that materially changes behavior, so building it means choosing for the human.

What to do when one fires:

- **If you can still build it correctly** — build it, but flag it in the Build Report's Own-tier section so the human re-derives it in review. A Co/Race tag downgrades their review, and an unflagged Own slice is one that silently skipped the model the human needed.
- **If building it requires an Own-class decision you'd only be guessing at** — stop and report blocked. Blocked beats fabricated; don't pick for the human on a slice that was supposed to be theirs.
