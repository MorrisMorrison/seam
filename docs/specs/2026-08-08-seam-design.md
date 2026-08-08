# seam — design

**Date:** 2026-08-08
**Status:** approved design, not yet implemented

A Claude Code skill that splits implementation work between the human and the agent: the human writes the parts that are dangerous or that they would regret not understanding, the agent writes everything else, and both work at the same time.

---

## Problem

AI-assisted development produces two failures at once.

**Comprehension rot.** Code arrives fully formed. It works, it passes, and six months later nobody can debug it because nobody wrote it. The riskiest code in a project is often the code its author has read once.

**Attention collapse.** Dispatching an agent creates a wait. The wait has no shape, so attention leaks out of it — phone, browser, another task, another room. Notification-based recovery does not work: the ping is missed, or seen and deferred, or seen and refused because returning means reloading context.

The naive fix for the second problem — filling the wait with an engaging companion activity — is wrong. It ties the developer to a single session and makes serialization permanent, destroying the parallelism that made agents worth using.

seam resolves both by moving the human's attention onto **real work on the critical path**, in parallel with the agent rather than in the gaps around it.

## Design constraints

Derived from the ADHD constraint filter (see the landscape doc in Synapse). Every part of this design is checked against them:

1. **Attention-regulating, not informing.** A tool that must be read has already failed.
2. **No attention switch required.** No mechanism may depend on a notification being noticed and acted upon.
3. **Novelty-proof by construction.** Content must derive from live work, or the tool becomes wallpaper within two weeks.
4. **Unavoidable and self-starting.** If it must be remembered, it is dead.

seam satisfies these because the work *is* the content: every task, score, and seam is new, and the workflow is how implementation happens rather than an optional layer beside it.

## Non-goals

- Not a wall-clock timer or focus tool.
- Not a distraction blocker. Reduction does not work here; engagement does.
- No gamification — no XP, scores, or streaks.
- Does not replace `writing-plans`. It consumes a plan; it does not produce one.
- Single repository, single plan at a time.

---

## Part 1 — Triage (plan level)

Input is an implementation plan produced by `writing-plans`. Every task is scored on two independent axes, 1–3.

### Blast radius — how bad is *wrong*?

| Score | Anchor |
|---|---|
| 3 | Irreversible or silent. Data loss or corruption, money movement, auth/authz, schema migrations, concurrency and races, cryptography — anything that can fail without producing an error. |
| 2 | Recoverable but costly. Wrong output reaching users, performance cliffs, retry and idempotency logic, external API contracts. |
| 1 | Fails loudly, cheap to fix. Rendering, formatting, config plumbing, logging, CRUD over a settled schema. |

### Comprehension debt — can you own this later if you did not write it?

| Score | Anchor |
|---|---|
| 3 | Novel algorithm or load-bearing domain logic. Cannot be re-derived by reading; central to the system's model. |
| 2 | Nontrivial but conventional. Re-following it later costs real time. |
| 1 | Mechanical or generated. Reading it is understanding it. |

### Assignment rule

```
yours    if max(blast, debt) == 3   or   (blast >= 2 and debt >= 2)
agent    otherwise

A `yours` task becomes `split` when the qualifying risk is confined to an
identifiable unit and the remainder is mechanical: the human takes the
unit, the agent takes the remainder.
```

Score by the riskiest unit in the task, not by its bulk. A task containing a novel algorithm scores on that algorithm even if most of its lines are wiring — `split` is what stops that from handing the human the wiring too.

This encodes the chosen "dangerous only" threshold: generous enough to catch everything genuinely risky or opaque, tight enough to leave most of the plan running in parallel.

### The manifest

Triage produces `docs/seam/<plan-slug>.md` in the target repo:

```markdown
# seam manifest — <plan name>

plan: docs/plans/<plan-file>.md
status: draft | approved | complete

| # | Task | Blast | Debt | Owner | Files | Status | Why |
|---|------|-------|------|-------|-------|--------|-----|
| 1 | Session token rotation | 3 | 2 | yours | internal/auth/rotate.go | pending | Auth failure is silent |
| 2 | Config loader | 1 | 1 | agent | internal/config/*.go | pending | Settled schema, loud failure |
| 3 | Rate limiter | 3 | 3 | split | internal/limit/*.go | pending | Wiring is mechanical; the bucket algorithm is not |
```

`Status` is one of `pending`, `in-progress`, `blocked`, `green`, `integrated`.

Two rules govern the manifest, and neither is negotiable:

- **Nothing starts until the manifest is approved.** A bad split is the failure mode with no recovery, so the one gate that cannot be automated is the human reading the table and redrawing any line they disagree with. Scores exist to make each line arguable rather than arbitrary.
- **The manifest records file ownership per task.** The agent is structurally forbidden from writing to any file the manifest currently assigns to the human. Disjointness is guaranteed by construction, so no merge protocol is needed and no worktree is required.

Ownership is per file and can transfer exactly once per task. A `split` task's files start as the agent's: it writes the mechanical remainder and the seam artifacts, and **at handoff those files transfer to the human for the remainder of the task**. From that moment the agent treats them like any other human-owned file. This is why a split task hands off only when the agent's side of it is finished — a partially built remainder would require both parties in the same file at the same time, which the design does not permit.

---

## Part 2 — The seam (task level)

`yours` and `split` differ only in what the agent does *after* the handoff. In both cases the agent builds the contract; in a `yours` task the entire implementation is then the human's, while in a `split` task the agent has also implemented the mechanical remainder of that same task before handing over.

For both, the agent produces four artifacts before the human writes anything:

1. **The stub**, in the real file, with signature, doc comment, stated invariants, and explicit non-goals.
2. **Failing tests** encoding the required behavior.
3. **The surrounding wiring**, already calling that stub.
4. **The spec gate** — the tests are presented as the specification, with a single question: *agree?* The human may edit or reject them before starting.

The spec gate exists to counter the one real risk of a test-based contract: the agent's tests silently pre-deciding the human's design. Reviewing them costs one screen; discovering the constraint halfway through an implementation costs the task.

**Test immutability.** Once the human accepts the tests, the agent may not modify them without asking. Otherwise it can move the goalposts mid-implementation.

## Parallel execution

While the human implements their piece, the agent executes its own manifest tasks.

- The agent never writes to a file owned by a `yours` task.
- Green tests on the human's piece are the progress signal. Nothing polls, watches, or interrupts.
- If the agent exhausts its own tasks it **stops and reports**. It never starts a task owned by the human, under any circumstances.
- If the human changes the interface mid-implementation, they say so and the agent re-wires its side. This is expected traffic, not an error path.

## The stall ladder

Stalling is the expected state, not an exception. It has a defined exit.

**Trigger.** A chat-resident skill has no clock; the agent exists only when invoked. The trigger is therefore the agent completing its own manifest tasks: it returns, sees the human's tests still red, and offers the ladder once. A true wall-clock timebox would require a Claude Code hook and is deliberately out of scope for v1.

**Rungs**, offered once and then silent — repeated prompting is worse than none:

1. **Hint** — one nudge, no code.
2. **Bigger hint** — the approach, still no code.
3. **Draft** — the agent writes a first attempt for the human to reshape.
4. **Take it** — full handoff. Manual invocation only, never offered automatically, always available.

Rung 4 is never suggested by the agent because the human's stated goal is to stay on the piece. It exists because a stall with no exit is precisely how work parks at 90%.

## Integration and review

When the human's tests go green:

1. The agent integrates and runs the full suite.
2. The complete diff goes to reef for the gated hunk walkthrough, with every hunk labeled `yours` or `agent`.
3. The agent reviews the human's hunks as well as its own, framed as a risk check rather than a critique. The human's hunks are by definition the highest-blast-radius code in the plan; leaving them unreviewed would mean the riskiest code is the only code nothing checks.
4. The agent never silently repairs the human's code. If something is wrong it reports and stops.

## State and resumability

The manifest is the state. Because it lives in the repo and records owner, files, and status per task, the workflow is resumable in the same way reef is: leave mid-task, return later, and seam reports which piece was yours and where it stopped.

## Failure modes

| Situation | Behavior |
|---|---|
| Human disagrees with the split | Redraw any line in the manifest; contracts re-derive from the new assignment |
| Agent's tests encode the wrong behavior | Caught at the spec gate; if found later, the human amends and the agent adapts |
| Human changes the interface mid-implementation | Human says so; agent re-wires its side |
| Agent finishes all its tasks first | Stops and reports; offers the stall ladder if the human's tests are red |
| Human abandons the piece entirely | Rung 4, manual handoff |
| File collision | Cannot occur by construction; if it does, the manifest is wrong and the human's copy wins |

## Validation

The skill is validated by an application test on one real task in a live repository, the way `/reef` was validated. Per the lesson recorded in the reef build notes, skill-following tests are not delegated to weak models — they skim skills and report false passes.

## Relationship to the existing chain

```
brainstorming → writing-plans → seam (triage) → parallel execution → reef
                                                 ├── human: core units
                                                 └── agent: everything else
```

seam sits between planning and execution. It consumes a plan, assigns it, and hands the finished diff to reef.

## Naming

`seam` — a geological seam, and the seam in code where two pieces meet. Alternates considered: `cleave`, `fault`.
