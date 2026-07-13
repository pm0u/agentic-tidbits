# /sloop — the engineering loop

A coordinator-driven loop for building a change under adversarial review. A dev agent builds against a spec, two reviewers attack the result in parallel, and a coordinator iterates with fresh dev agents until the code passes review or the loop stalls. You stay out of it until the end — the one exception is a single approval gate before any code is written.

The bet is simple: AI is decent at building bounded things and bad at knowing when it's wrong. So sloop never trusts a single pass. Whatever the builder produces gets stress-tested by agents whose only job is to find what's broken, and a coordinator — which never writes code itself — decides what's a real finding and what's noise.

## When to use it

Reach for sloop when the task is big enough that a single build-and-hope pass isn't trustworthy, and concrete enough to pin down acceptance criteria. It earns its overhead on changes where the risk is in the details — the kind of thing you'd want a careful reviewer on anyway.

Skip it for trivial changes, where the coordination cost outweighs the work, and for open-ended exploration where you don't yet know what "done" means. sloop needs a contract to build and review against — if you can't state 2-6 acceptance criteria, you're not ready for the loop yet, and `/derive-spec` is the better first stop.

## Usage

```
/sloop [task description]
/sloop --yolo [task description]
```

By default the loop pauses once, after the plan review, to show you the final spec and get your go-ahead before any code is written. `--yolo` skips that gate and runs autonomously — the coordinator resolves open questions with its own judgment and records those calls in the final report. Everything after Phase 0 is autonomous either way.

## Roles

| Role | Who | Job |
|------|-----|-----|
| Coordinator | primary session | Writes the spec, spawns the agents, adjudicates findings, decides pass/iterate. Never writes code. |
| Researcher | `sloop-researcher` | Explores the codebase and returns a brief so the coordinator authors the spec from fact. Phase 0, only when the task needs grounding the coordinator lacks. |
| Plan reviewer | `adversarial-plan-reviewer` | Stress-tests the spec at handoff, before any code exists. |
| Builder | `sloop-dev` | Implements the spec (or fixes findings) in phases, verifies for real, reports honestly what it didn't verify. Fresh instance every iteration. |
| Reviewer A | `adversarial-code-reviewer` | Line level: bugs, fabrications, reckless completion. |
| Reviewer B | `adversarial-architecture-reviewer` | System level: placement, patterns, complexity, direction. |
| Verifier | `sloop-verifier` | Reproduces the build report's verification claims and attacks its "not verified" list. Spawned alongside the reviewers when the report claims behavioral verification; skipped when re-running the test suite covers it. |

Subagents can't spawn subagents — all spawning happens in the coordinator.

## How it flows

**Phase 0 — spec & plan review.** The coordinator turns your task into a short spec: goal, constraints, 2-6 acceptance criteria. When the task needs codebase grounding it doesn't already have, it hands off to the researcher first, which explores and returns a brief so the spec is authored from fact rather than guesswork. The plan reviewer then stress-tests the spec — the cheapest place to catch a bad assumption is before any code exists. You approve the frozen spec (unless `--yolo`), the work goes onto a feature branch, and the coordinator records whether the test suite passes at the baseline so pre-existing failures can't be pinned on the build.

**Phase 1 — build.** A fresh dev agent implements the spec in phases (orient, plan, implement, verify), commits in small units, and reports what it did and — just as important — what it couldn't verify. It gets the research brief so it orients from the map instead of re-exploring.

**Phase 2 — review.** Both reviewers attack the diff in parallel. They get the spec, the research brief, and the build report, including its "not verified" and "assumptions" sections, so they know where to dig. When the build report claims behavioral verification, a verifier joins the same parallel batch and reproduces the claims — re-running what the dev says it ran, attempting what it says it couldn't. On fix rounds the reviewers get the round's delta as their focus, confirm the claimed fixes, and check for regressions rather than re-reviewing the whole range from scratch.

**Phase 3 — adjudicate.** The coordinator merges the reviews (and the verification report, if a verifier ran) and triages every finding: actionable ones go to the next round, noise gets dropped with a reason, disputes get resolved with evidence or parked for you. A build-report claim the verifier contradicted is automatically actionable and discredits the report's other claims. Then it decides — pass, iterate with a fresh dev agent, or stall.

**Phase 4 — report.** You get a summary: the branch, a diff link for the whole range, the review history, which findings were resolved and how, what was dismissed as noise (you may disagree), the disputes left for you, and a short "what to check" list of specific things to verify by hand.

## What you get back

The work lands on a feature branch, with the loop's commits kept off main. The final report routes your attention to where it matters — disputes and unverified claims — instead of asking you to scan the whole diff. The dismissed-as-noise section is deliberate: the coordinator shows its work so you can catch a finding it dropped too eagerly.

## Limits worth knowing

- Passing the loop means the code survived adversarial review, not that it's correct. You're still the final reviewer — that's what the "what to check" list is for.
- The loop stalls on purpose. A wrong-direction architecture verdict, the same finding surviving two rounds, or three completed iterations stops it and escalates to you. More loops won't fix a disagreement about direction, and pretending otherwise just burns iterations.
- The spec freezes after the Phase 0 gate. Mid-loop scope changes mean starting a new loop — the contract can't move under the agents building and reviewing against it.
- `--yolo` trades the safety of the approval gate for autonomy. The coordinator still runs the plan review; it just resolves what it finds with its own judgment. Good for low-stakes or well-understood tasks, riskier for anything where a wrong assumption is expensive.
