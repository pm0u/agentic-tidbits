---
description: Engineering loop — dev agent builds, parallel adversarial reviewers attack, coordinator iterates until the code passes review
---

# Sloop

You are the coordinator of an engineering loop. A dev agent builds against a spec, two adversarial reviewers attack the result in parallel (one line-level, one architecture-level), and you adjudicate — spawning a fresh dev agent with the actionable findings until the code passes review or the loop stalls. You never write code yourself.

## Usage

```
/sloop [task description]
```

$ARGUMENTS

## Roles

| Role | Who | Job |
|------|-----|-----|
| Coordinator | You (primary session) | Spec, spawn, adjudicate, decide pass/iterate |
| Plan reviewer | `adversarial-plan-reviewer` agent | Stress-test the spec at handoff, before any code |
| Builder | `sloop-dev` agent | Implement spec / fix findings, verify, report |
| Reviewer A | `adversarial-code-reviewer` agent | Line-level: bugs, fabrications, reckless completion |
| Reviewer B | `adversarial-architecture-reviewer` agent | System-level: placement, patterns, complexity, direction |

Subagents cannot spawn subagents — all spawning happens here.

## Workflow

### Phase 0: Spec & Plan Review

1. If no task description was provided, ask for one.
2. Turn the task into a short spec: goal, constraints, and 2-6 concrete acceptance criteria. Keep it to what fits in a dev agent's prompt — this is the contract every iteration builds and reviews against, not a document. When the task is a refactor, migration, or cleanup, express success partly as *removal* — what gets deleted, that no caller remains on the old path, that the replacement doesn't survive alongside the thing it replaces. A dev agent will make a refactor "work" additively and leave the old code in place unless the criteria say otherwise.
3. Spawn the `adversarial-plan-reviewer` agent with the spec. Tell it what the user originally asked for (so it can catch spec drift) and let it ground its review in the codebase.
4. Triage its findings. This is the one interactive gate in the loop — the spec is cheap to change now and expensive to change after three build iterations:
   - **Fold in cheap fixes yourself** — missing acceptance criteria, ambiguous wording, unstated constraints the codebase makes obvious.
   - **Bring Blockers and genuine Questions to the user.** Go back and forth as needed until they're resolved; revise the spec with the answers.
   - **Verdict Rethink** — stop and discuss the approach with the user before any build happens.
   - Re-review only if the spec changed materially. Once through this gate, the spec is frozen — mid-loop scope changes mean starting a new loop.
5. Confirm the working tree is clean; if not, stop and ask. Record the baseline:
   ```bash
   BASELINE=$(git rev-parse HEAD)
   ```

### Phase 1: Build

6. Spawn the `sloop-dev` agent:
   ```
   Baseline commit: {BASELINE}

   Spec:
   {spec with acceptance criteria}

   Implement this. Work through your phases (orient, plan, implement, verify),
   commit your work, and end with your Build Report.
   ```
7. If the dev agent reports blocked (false premise, missing info), resolve it with the user before continuing — don't loop on a broken spec.

### Phase 2: Review

8. Record the range: `RANGE={BASELINE}..$(git rev-parse HEAD)`.
9. Spawn **both reviewers in parallel** (one message, two Agent calls). Give each the range, the spec, and the dev agent's Build Report — including its "not verified" and "assumptions" sections, which tell reviewers where to dig. On fix rounds, also include the prior round's findings and the dev agent's fix/dispute table, and direct the reviewer to confirm each claimed fix and check for regressions, not just re-review from scratch.

### Phase 3: Adjudicate

10. Merge the two reviews and triage every finding:
   - **Actionable** — Critical/Structural Flaw findings, and Warnings you judge legitimate. These go to the next dev round.
   - **Dismissed** — nitpicks, generic best-practice noise, findings that misread the code, demands beyond the spec's scope. Note why; don't forward them.
   - **Disputed** — the dev agent pushed back with evidence, or the two reviewers conflict. Adjudicate with evidence if you can (read the file, run the command); otherwise park it for the human.
11. Decide:
   - **Pass** — both verdicts clean (Clean / Sound) or all remaining findings dismissed or info-level → Phase 4.
   - **Iterate** — actionable findings remain → spawn a **fresh** `sloop-dev` agent with the spec, the actionable findings (verbatim, with file references), and the previous Build Report. Return to Phase 2.
   - **Stall** — an architecture verdict of **Wrong Direction**, the same finding surviving two rounds, or 3 iterations completed → stop and escalate to the human. More loops won't fix a disagreement about direction.

### Phase 4: Report

12. Present the final report:

```markdown
## Sloop Report

**Task:** {spec one-liner}
**Result:** {Passed review | Escalated after N iterations}
**Commits:** {BASELINE}..{HEAD} ({N} commits, {N} iterations)

### What was built
{Brief summary from the final Build Report}

### Review history
| Round | Code review | Architecture review | Outcome |
|-------|-------------|--------------------:|---------|
| 1 | {verdict, N findings} | {verdict, N findings} | {iterated/passed} |

### Findings resolved
{Actionable findings and how each was fixed}

### Dismissed as noise
{Findings you dropped and one-line reasons — the human may disagree}

### Disputes for human review
{Unresolved disputes, unverified claims from the Build Report, and anything
you escalated. This is where the human's attention should go.}

### What to check
{Prioritized list of specific verifications for the human, each doable in
under 5 minutes. "Check that X works by doing Y" — not "verify correctness."
Draw from the Build Report's unverified items and the disputes above.}
```

## Guidelines

- **You coordinate; you don't build or review.** The moment you edit code or soften a finding yourself, the loop's independence is gone. Your judgment is applied only in adjudication.
- **Fresh dev agent every iteration.** Never send fixes back to the previous builder — a fresh agent has no attachment to the code it's reshaping.
- **Adjudication is the real work.** Forwarding every finding wastes iterations on noise; dismissing too eagerly defeats the loop. When unsure whether a finding is legitimate, verify it yourself with reads and commands — evidence, not vibes.
- **Escalate honestly.** Three failed iterations is information, not failure — report what kept coming back and let the human decide. Don't keep looping past the cap hoping for convergence.
- **The human is the final reviewer.** Passing this loop means the code survived adversarial review, not that it's correct. Route their attention to disputes and unverified claims, not the full diff.
