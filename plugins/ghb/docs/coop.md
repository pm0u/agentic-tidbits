# /coop — the cooperative engineering loop

A fork of [`/sloop`](sloop.md) where you are a first-class builder, not just the final
reviewer. The change is split into slices, each slice is tiered, you build the slices you
need to understand and agents build the rest, adversarial reviewers attack everything, and
a coordinator iterates until the code passes review.

sloop's bet still holds — AI is decent at building bounded things and bad at knowing when
it's wrong, so nothing is trusted on a single pass. coop adds a second bet: for some slices
the *human's understanding* is a required output, not just correctness, and that
understanding only comes from building or re-deriving the code. So coop routes those slices
to you on purpose, instead of handing you a finished diff to rubber-stamp.

## When to use it

Reach for coop over sloop when you'll own or maintain the code and a wrong mental model is
expensive — or when the point is to *learn* the code, not just ship it. It's the answer to
"the agent could build this, but then I wouldn't understand it." If you genuinely don't need
the model — throwaway, boilerplate, a well-understood change — plain sloop is less overhead.

## The tier rubric

The rubric is two files. [`docs/tiers-core.md`](tiers-core.md) is the classification —
shared with every other tool that tiers work, and not coop-specific.
[`docs/tiers.md`](tiers.md) is coop's layer on it: slices as the unit, shells, bubble-up, and
the ledger. Each slice is **Own** (you build or re-derive it), **Co** (an agent builds, you
review predict-before-peek), or **Race** (an agent builds, you verify it works). Blast radius
is the dominant signal; unsure defaults to Co, because bubble-up catches an under-tiered slice
mid-build. Tier decides who *builds* a slice, not who *fixes* it — on a fix round,
[bubble-down](tiers-core.md#routing-a-fix--by-the-work-not-by-the-unit) routes each finding by
the kind of work the fix is, and the ledger keeps your model of an Own slice current when an
agent lands fixes inside it.

## Usage

```
/coop [task description]
/coop --escalate=co [task description]
/coop --escalate=co,race [task description]
/coop --yolo [task description]
```

`--escalate` sets *who builds* which tier — classification runs at every level regardless.
Default: you build Own, agents build Co + Race. `--escalate=co`: you also build Co.
`--escalate=co,race` (or `=all`): you build everything and agents only plan, review, and
verify — the codebase-familiarization mode. `--yolo` implies `--escalate=none` and skips the
approval gate; the plan review still runs.

## How it flows

**Phase 0 — spec, slice & tier.** The coordinator turns your task into a short spec, then
decomposes it into tier-homogeneous slices, classifies each, and drafts a shell (a signature
plus a one-line contract) for every seam between slices. The plan reviewer stress-tests the
spec *and the tiering* — a Race slice that mutates a shared invariant should be Own. You
approve the frozen spec, slice list, and shells before any code.

**Phase 1 — build.** Shells are agreed first. Test coverage for an Own slice is its own Co
slice, built by an agent and ordered foundation-first behind the core it pins — the rubric
already classifies testing around Own code as Co work, so it never lands on you. Then the build runs in the order set per seam:
foundation-first seams (where a slice needs its predecessor to actually exist, not just a
contract) go first; everything else fans out in parallel — each concurrent agent slice runs
in its own git worktree (via `loop-dev` with worktree isolation) while you build your slices
in the primary working directory, committing as yourself. The loop reconverges by merging each
agent's branch back into the feature branch (foundation-first); because shells keep parallel
slices file-disjoint, those merges stay clean. An agent that finds a Co/Race slice is actually
Own bubbles it up; the slice is re-tiered, you re-derive the flagged code in review, and it's
rebuilt only if that re-derivation finds the model wrong. When you signal done, you also say
what you verified — your slices produce no Build Report, so that's the claim set the verifier
reproduces for your work.

**Phase 2 — review.** Both adversarial reviewers attack every slice at full depth regardless
of tier or who built it — your Own slice gets the same scrutiny as an agent's Race slice. A
verifier reproduces behavioral claims when any Build Report — or your own done signal — makes
them. Reviewers get the shells (the contracts to check against) but not the tier labels, so a
tier can't argue them into going easier.

**Phase 3 — adjudicate.** The coordinator triages findings as sloop does — trajectory check,
premise findings escalated to you rather than dismissed, chain check on each actionable
finding — plus: a violated shell is automatically actionable, and a bubble-up flag re-tiers
its slice. Once the agent
verdicts are clean, you do your per-tier review before the loop can pass — predict-before-peek
on each Co slice, a behavior check on each Race slice, re-derivation of anything bubble-up
flagged. [`docs/re-derive.md`](re-derive.md) is the method for those reads. On iterate, findings route by the kind of work the fix is rather than by who built the slice:
an **Absorbed** fix — an added case, a guard, an optimization behind unchanged behavior, landing
at an existing extension point without moving an invariant or a contract — goes to a fresh
`loop-dev` even inside a slice you built, while a **Bend** comes back to you. An agent that
starts an Absorbed fix and finds it bends stops rather than reshaping your core. Every agent fix
inside an Own slice gets a ledger line — case, delta, model-touching yes/no — and at the tier
review you read the ledger and re-derive only the model-touching entries. That's the mechanism
that gets a dozen minor edge cases off your plate without costing you the model you built the
slice to get.

**Phase 4 — report.** The sloop report shape, plus who built what, the tiering decisions, the
per-slice ledger, and an **understanding check** — a pointed question or two per Own slice you built, to confirm you
can still explain it cold. A passed loop where you can't answer for your own slices is a failed
loop; this is where that gets caught, not at 2am. Where a slice has a ledger, at least one
question comes from a model-touching entry — the code you didn't write inside a slice you're
credited with owning. A missed answer doesn't close the loop — you re-derive the slice and the
check re-runs.

## What you get back

The work lands on a feature branch. You come away with the code *and* a real model of the
parts that mattered — because you built them — while the parts that didn't were handled for
you and still survived adversarial review.

## Limits worth knowing

- The coordination is only as parallel as the seams allow. Tightly coupled slices, or shells
  that won't stabilize, fall back to sequential — coop doesn't pretend otherwise.
- Passing the loop still means the code survived adversarial review, not that it's correct.
  You're the final reviewer.
- Agent review never softens by tier. Tier decides who *builds* and how deep *you* review;
  the adversarial agents attack everything at full strength.
- Shells are contracts. Once agreed, neither side changes one unilaterally — a contract that
  moves under a parallel builder is how the seam breaks.
- Bubble-down only moves *mechanical* fixes. Unsure is always a Bend, because an over-routed
  Absorbed fix costs you an edit you'd have made anyway while an under-routed Bend quietly
  rewrites the model you're supposed to be holding.
- A slice that keeps producing Bends or model-touching ledger entries has the wrong *shape*,
  not thin coverage. The loop stalls there on purpose — no number of absorbed cases fixes a
  shape.
