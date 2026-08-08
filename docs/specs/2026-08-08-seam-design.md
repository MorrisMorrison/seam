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

**Two tie-breakers, both required.**

**Apply precedence first, then reversibility.** They pull in opposite directions by design: precedence prevents under-scoring, reversibility prevents scoring a category name instead of the work.

- **Precedence on multiple matches.** Tasks routinely match anchors in two rows — evicting entries from a shared session cache matches both `concurrency and races` (3) and `performance cliffs` (2). **The highest matching anchor wins.** Name both matches in the justification so the call is visible and arguable rather than silent.
- **Reversibility.** Category names are a starting point, not a verdict — score the *instance*. Ask whether this specific change is reversible and whether it fails loudly. Adding an unused enum value is a schema change, but it is reversible and fails immediately: score it 1. A backfill that rewrites existing rows cannot be undone: score it 3. **When reversibility clears a category match, drop to the score the work would carry if the category name did not apply at all** — do not split the difference. Without this, every migration, index, and column add scores identically to a destructive rewrite, and the human is handed safe, boring work.

### Comprehension debt — can you own this later if you did not write it?

| Score | Anchor |
|---|---|
| 3 | Novel algorithm or load-bearing domain logic. Cannot be re-derived by reading; central to the system's model. |
| 2 | Nontrivial but conventional. Re-following it later costs real time. |
| 1 | Mechanical or generated. Reading it is understanding it. |

### Assignment rule

```
user     if max(blast, debt) == 3   or   (blast >= 2 and debt >= 2)
agent    otherwise

A `user` task becomes `split` when the qualifying risk is confined to an
identifiable unit and the remainder is mechanical: the human takes the
unit, the agent takes the remainder.
```

**Identifiable** has a concrete test, borrowed from the seam contract itself: you can write a stub and failing tests for the unit *without* also building the remainder. If the two cannot be separated at that boundary they are one unit, not two — retry logic and the idempotency it protects are a single correctness problem however cleanly they appear to divide.

**Mechanical** means the remainder scores `agent` on its own under this same rubric. Not "smaller", not "boring" — scored.

**When several boundaries pass the stub test**, take the one leaving the smallest retained unit whose remainder still scores `agent`. An irreversible write always stays inside the retained unit — never hand the agent the part that cannot be undone.

**The remainder must be task-sized.** If what is left after removing the retained unit would not have stood as its own task in a plan — a single entrypoint, one call site, a config line — there is nothing worth splitting. Keep the whole task with the human. A split exists to spare them mechanical bulk; where there is no bulk, it is only ceremony.

A `split` row's Blast and Debt describe **the retained unit**, not the whole task.

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
| 1 | Invoice tax calculation | 3 | 3 | user | internal/billing/tax.go | pending | Wrong tax is silent, and irreversible once invoiced |
| 2 | Locale string loader | 1 | 1 | agent | internal/i18n/*.go | pending | Settled schema, loud failure |
| 3 | Search result ranking | 2 | 3 | split | internal/search/*.go | pending | Query plumbing is mechanical; the ranking function is not |
```

`Status` is one of `pending`, `in-progress`, `blocked`, `green`, `integrated`.

Two rules govern the manifest, and neither is negotiable:

- **Nothing starts until the manifest is approved.** A bad split is the failure mode with no recovery, so the one gate that cannot be automated is the human reading the table and redrawing any line they disagree with. Scores exist to make each line arguable rather than arbitrary.
- **The manifest records file ownership per task.** The agent is structurally forbidden from writing to any file the manifest currently assigns to the human. Disjointness is guaranteed by construction, so no merge protocol is needed and no worktree is required.

**A redraw outranks the rule.** The human may reassign any row, and their call stands even where the rubric contradicts it — they are the approver, and the scores exist to inform that judgment rather than to overrule it. Record the override in the row's `Why` cell and leave the scores untouched. Never re-score a row to make its new owner look rule-derived: that hides a human decision behind arithmetic, and the manifest stops being an honest record of who chose what.

**Reassignment survives approval.** The redraw mechanics above are not confined to the draft stage: a row can move owners at any point, by the same procedure — update the row, record the override in `Why`, leave the scores alone. What changes after approval is only that in-flight work may exist, so the handoff must say explicitly whether existing work in the file is kept or discarded.

Ownership is per file and can transfer exactly once per task. **The files of a `user` or `split` row start as the agent's** — it is the agent that writes the stub, the failing tests, and the wiring, which is why Part 2's artifacts can exist at all — and **at handoff they transfer to the human for the remainder of the task**. A `split` task's files start as the agent's for the same reason, with the mechanical remainder built before handoff. From that moment the agent treats them like any other human-owned file. This is why a split task hands off only when the agent's side of it is finished — a partially built remainder would require both parties in the same file at the same time, which the design does not permit.

---

## Part 2 — The seam (task level)

`user` and `split` differ only in what the agent does *after* the handoff. In both cases the agent builds the contract; in a `user` task the entire implementation is then the human's, while in a `split` task the agent has also implemented the mechanical remainder of that same task before handing over.

For both, the agent produces four artifacts before the human writes anything:

1. **The stub**, in the real file, with signature, doc comment, stated invariants, and explicit non-goals.
2. **Failing tests** encoding the required behavior.
3. **The surrounding wiring**, already calling that stub.
4. **The spec gate** — the tests are presented as the specification, with a single question: *agree?* The human may edit or reject them before starting.

The spec gate exists to counter the one real risk of a test-based contract: the agent's tests silently pre-deciding the human's design. Reviewing them costs one screen; discovering the constraint halfway through an implementation costs the task.

**Test immutability.** Once the human accepts the tests, the agent may not modify them without asking. Otherwise it can move the goalposts mid-implementation.

**When the human is the one asking to relax a test**, the rule still binds — it is a duty to explain, not a veto. The agent states what the test protects and what its removal would let through, then complies if the human holds. Their call stands; they simply make it knowing the cost, and the decision is recorded in the row's `Why` cell rather than disappearing into a quiet edit.

Six rules govern how a seam is built:

- **One task at a time.** Build a seam for the task the human asks for, not for every `user` and `split` row at once. Several open spec gates defeat the point of having a gate. A gate closes once the user accepts its tests; you may build the next seam then.
- **The stub must fail without aborting the run.** Where the language allows it, return a not-implemented error rather than raising one, so the remaining tests still report. A stub that kills the test binary hides the rest of the spec behind the first assertion.
- **Name what the signature decides.** A half-right body anchors the design — but so does a signature. If the contract forces a design choice (injecting a clock so refill can be tested without sleeping), say so at the gate and offer the alternative. The gate exists to surface those, not merely to approve assertions.
- **The agent tests its own remainder.** Those tests are the agent's own and are neither part of the gate nor covered by test immutability.
- **Unassigned scaffolding belongs to the agent.** A module file or build config that the plan gives to no task still has to exist. The agent creates it and names it at the gate, so the human is never surprised by files outside the manifest.
- **Status transitions.** The agent sets a row `in-progress` when it builds the seam, `green` when the human's tests pass, and `integrated` once the suite is clean. Only the human sets `blocked`.

## Parallel execution

While the human implements their piece, the agent executes its own manifest tasks.

- The agent never writes to a file owned by a `user` task.
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
2. The complete diff goes to reef for the gated hunk walkthrough, with every hunk labeled `user` or `agent`.
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
