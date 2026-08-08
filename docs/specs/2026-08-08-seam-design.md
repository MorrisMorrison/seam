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

Triage runs only when there is no manifest yet. If `docs/seam/<plan-slug>.md` already exists, the run is a resumption: the agent reads that manifest, resumes from it, and reports where things stand. It does not re-score, does not rewrite the `Why` column, and does not reset `status:` — see *State and resumability*.

Score the task as written; do not invent requirements it does not state. **Inherent properties of where the code runs count as written** — an HTTP handler serves concurrent requests, a nightly job runs unattended — because those are facts about the work rather than added requirements. Unstated *requirements* do not count: scale, deployment topology, SLAs, and threat models the plan never mentions are out of bounds.

### Blast radius — how bad is *wrong*?

| Score | Anchor |
|---|---|
| 3 | Irreversible or silent. Data loss or corruption, money movement, auth/authz, schema migrations, concurrency and races, cryptography — anything that can fail without producing an error. |
| 2 | Recoverable but costly. Wrong output reaching users, performance cliffs, retry and idempotency logic, external API contracts. |
| 1 | Fails loudly, cheap to fix. Rendering, formatting, config plumbing, logging, CRUD over a settled schema. |

**The blast-3 catch-all is a fallback, not an anchor.** "Anything that can fail without producing an error" exists to catch work no named category reaches. Where a task matches a **named** anchor in a lower row — `retry and idempotency logic`, `performance cliffs`, `external API contracts`, and the row-1 names — that named match governs and the catch-all does not promote it. A specifically named lower anchor beats the catch-all; without this, nearly every named blast-2 item is promoted to 3 and the assignment rule's second clause never fires.

**Two tie-breakers, both required.**

**Apply precedence first, then reversibility.** They pull in opposite directions by design: precedence prevents under-scoring, reversibility prevents scoring a category name instead of the work.

- **Precedence on multiple matches.** Tasks routinely match anchors in two rows — evicting entries from a shared session cache matches both `concurrency and races` (3) and `performance cliffs` (2). **The highest matching named anchor wins.** Name both matches in the justification so the call is visible and arguable rather than silent. Precedence settles conflicts between two *named* anchors; it never fires the catch-all against a name.
- **Reversibility.** Category names are a starting point, not a verdict — score the *instance*. Ask whether this specific change is reversible and whether it fails loudly. Deleting a generated thumbnail matches `data loss` by name, but it is rebuilt from the original on next request and its absence shows immediately: score it 1. Deleting the original upload it was derived from cannot be undone: score it 3. **When reversibility clears a category match, drop to the score the work would carry if the category name did not apply at all** — do not split the difference. Without this, every derived artifact, migration, and additive change scores identically to a destructive rewrite, and the human is handed safe, boring work.

### Comprehension debt — can you own this later if you did not write it?

| Score | Anchor |
|---|---|
| 3 | Novel algorithm or load-bearing domain logic. Cannot be re-derived by reading; central to the system's model. |
| 2 | Nontrivial but conventional. Re-following it later costs real time. |
| 1 | Mechanical or generated. Reading it is understanding it. |

**3 versus 2.** "Conventional" means the work has a **canonical published form the implementer would follow** — a documented algorithm, a standard protocol exchange, the ordinary shape of a well-known library. That scores 2 however fiddly it is to re-follow. 3 is reserved for logic whose shape is specific to this system: it cannot be looked up, only re-derived from the domain. Work that is both central and canonical scores 2 — being load-bearing does not make a published algorithm unlearnable.

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
| 1 | Offline sync conflict resolution | 3 | 3 | user | internal/sync/merge.go | pending | Diverged replicas drop edits without complaining; the merge rules are ours alone |
| 2 | Static asset fingerprinting | 1 | 1 | agent | internal/assets/*.go | pending | Deterministic rewrite, and a wrong hash 404s on sight |
| 3 | Search result ranking | 2 | 3 | split | internal/search/*.go | pending | Query plumbing is mechanical; the ranking function is not |
```

`Status` is one of `pending`, `in-progress`, `blocked`, `green`, `integrated`.

Two rules govern the manifest, and neither is negotiable:

- **Nothing starts until the manifest is approved.** A bad split is the failure mode with no recovery, so the one gate that cannot be automated is the human reading the table and redrawing any line they disagree with. Scores exist to make each line arguable rather than arbitrary.
- **The manifest records file ownership per task.** The agent is structurally forbidden from writing to any file the manifest currently assigns to the human. Disjointness is not an accident of construction but a requirement on the table: **the `Files` sets of differently-owned rows must be disjoint.** Rows sharing an owner may share a path; rows with different owners may not, and a wildcard must not straddle an ownership boundary. An overlap means the split is drawn in the wrong place and must be redrawn — or the paths narrowed — before approval. Because the column is disjoint, no merge protocol is needed and no worktree is required.

**A redraw outranks the rule.** The human may reassign any row, and their call stands even where the rubric contradicts it — they are the approver, and the scores exist to inform that judgment rather than to overrule it. Record the override in the row's `Why` cell and leave the scores untouched. Never re-score a row to make its new owner look rule-derived: that hides a human decision behind arithmetic, and the manifest stops being an honest record of who chose what.

**Reassignment survives approval.** The redraw mechanics above are not confined to the draft stage: a row can move owners at any point. What changes after approval is only that in-flight work may exist, so the handoff must say explicitly whether existing work in the file is kept or discarded.

**Ownership changes only by a recorded manifest edit, never implicitly, and one procedure covers every such change.** A redraw before approval, a reassignment after it, a rung 4 handoff, and an explicit return of a file to the agent are the same event under different names: update the row's `Owner`, record the reason in `Why`, leave `Blast` and `Debt` untouched. Rung 4 is not an exception — it edits the manifest like any other owner change. There is no cap on how often a row changes hands; the requirement is that each change is written down, so the row always names its current owner.

**The files of a `user` or `split` row start as the agent's** — it is the agent that writes the stub, the failing tests, and the wiring, which is why Part 2's artifacts can exist at all — and **they transfer to the human when the human accepts the spec gate**. A `split` task's files start as the agent's for the same reason, with the mechanical remainder built before handoff. From that moment the agent treats them like any other human-owned file. This is why a split task hands off only when the agent's side of it is finished — a partially built remainder would require both parties in the same file at the same time, which the design does not permit.

---

## Part 2 — The seam (task level)

`user` and `split` differ only in what the agent does *after* the handoff. In both cases the agent builds the contract; in a `user` task the entire implementation is then the human's, while in a `split` task the agent has also implemented the mechanical remainder of that same task before handing over.

For both, the agent produces four artifacts before the human writes anything:

1. **The stub**, in the real file, with signature, doc comment, stated invariants, and explicit non-goals.
2. **Failing tests** encoding the required behavior.
3. **The surrounding wiring**, already calling that stub.
4. **The spec gate** — the tests are presented as the specification, with a single question: *agree?* The human may edit or reject them before starting.

The spec gate exists to counter the one real risk of a test-based contract: the agent's tests silently pre-deciding the human's design. Reviewing them costs one screen; discovering the constraint halfway through an implementation costs the task.

**The handoff is the human's acceptance of the gate, not the agent's posting of it.** Everything the agent intends to write into those files must already be there when it posts — the gate is its last write, not a pause in the middle of a series of them. But the files remain the agent's until acceptance, which is what makes the gate's question answerable: an edit the human asks for *at the gate* is made by the agent, then re-posted for a fresh answer. Acceptance transfers the files to the human for the remainder of the task, and needs no separate status value — acceptance is the event.

**A defective artifact discovered after handoff cannot be repaired by the agent.** If the agent finds a fault in a stub, test, or wiring it built — a stub that aborts the run, a test asserting the wrong thing — it reports the fault and states what the correct version would be. It does not edit, and it does not offer to: an offer to edit a human-owned file is a disguised handoff, and handoffs are the human's to invoke. If they want it fixed, that is an explicit return of the file, recorded like any other reassignment.

**Test immutability.** Once the human accepts the tests, the agent may not modify them without asking. Otherwise it can move the goalposts mid-implementation.

**When the human is the one asking to relax a test**, the rule still binds — it is a duty to explain, not a veto. The agent states what the test protects and what its removal would let through, then complies if the human holds. Their call stands; they simply make it knowing the cost, and the decision is recorded in the row's `Why` cell rather than disappearing into a quiet edit.

Six rules govern how a seam is built:

- **One task at a time.** Build a seam for the task the human asks for, not for every `user` and `split` row at once. Several open spec gates defeat the point of having a gate. A gate closes once the user accepts its tests; you may build the next seam then.
- **The stub must fail without aborting the run.** Where the language allows it, return a not-implemented error rather than raising one, so the remaining tests still report. A stub that kills the test binary hides the rest of the spec behind the first assertion.
- **Name what the signature decides.** A half-right body anchors the design — but so does a signature. If the contract forces a design choice (taking a seeded random source as a parameter, so shuffle order can be asserted rather than merely sampled), say so at the gate and offer the alternative. The gate exists to surface those, not merely to approve assertions.
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

**Trigger.** A chat-resident skill has no clock; the agent exists only when invoked. The trigger is therefore the agent completing its own manifest tasks: it returns, sees the human's tests still red, and offers the ladder once per stall episode. A true wall-clock timebox would require a Claude Code hook and is deliberately out of scope for v1.

**One offer per stall episode.** An episode begins with an offer on a row and ends when the human next acts on that row — taking a rung, pushing work into it, reporting it green, or reassigning it. While an offer is outstanding the agent says nothing further about it. Once the episode ends, a later stall on the same row is a new episode and may be offered once again. Never two live offers for one row.

**Rungs**, offered once per episode and then silent — repeated prompting is worse than none:

1. **Hint** — one nudge, no code.
2. **Bigger hint** — the approach, still no code.
3. **Draft** — the agent writes a first attempt for the human to reshape.
4. **Take it** — full handoff. Manual invocation only, never offered automatically, always available. Like any owner change it is recorded in the manifest: new `Owner`, reason in `Why`, scores untouched.

Rung 4 is never suggested by the agent because the human's stated goal is to stay on the piece. It exists because a stall with no exit is precisely how work parks at 90%.

## Integration and review

When the human's tests go green:

1. The agent integrates and runs the full suite.
2. The complete diff goes to reef for the gated hunk walkthrough, with every hunk labeled `user` or `agent`.
3. The agent reviews the human's hunks as well as its own, framed as a risk check rather than a critique. The human's hunks are by definition the highest-blast-radius code in the plan; leaving them unreviewed would mean the riskiest code is the only code nothing checks.
4. The agent never silently repairs the human's code. If something is wrong it reports and stops.
5. A red suite is fixed by the agent only where the failing code sits in a file the agent **still owns**. Ownership decides this, not authorship: a fault in a stub, test, or wiring the agent wrote that now lives in a human-owned file is reported under the post-handoff rule, never edited.

## State and resumability

The manifest is the state. Because it lives in the repo and records owner, files, and status per task, the workflow is resumable in the same way reef is: leave mid-task, return later, and seam reports which piece was yours and where it stopped.

This only holds if the manifest is never rebuilt from scratch. On any invocation the agent looks for `docs/seam/<plan-slug>.md` **before** it triages: if the file exists it is the state, and the agent resumes from it and reports where things stand. It does not re-score, does not overwrite recorded statuses, and does not reset an approved manifest to `draft` — the `Why` column is the record of every human decision made so far, and re-triage would erase it. A fresh manifest is written only when no manifest exists.

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
