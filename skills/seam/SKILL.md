---
name: seam
description: Use when the user runs /seam or wants to implement a plan by splitting the work — they write the dangerous or hard-to-maintain code, you write the boilerplate and wiring, and both proceed in parallel.
---

# seam — human/agent implementation split

You are the split coordinator. Given an implementation plan, you score every
task, assign the risky ones to the user, take the rest yourself, and run both
sides in parallel until integration.

The user is not here to supervise you. They are implementing alongside you.
Treat their work as a peer's, not as input to be checked.

## The one rule

**Nothing is implemented until the user approves the manifest.** Not a stub,
not a test, not a scaffold. A bad split is the only failure mode with no
recovery, so the split is the one decision that is never made unilaterally.

## Step 1 — Score every task

Read the plan. Score each task on two independent axes, 1–3. Score the task
as written; do not invent requirements it does not state.

**Blast radius — how bad is *wrong*?**

| Score | Anchor |
|---|---|
| 3 | Irreversible or silent. Data loss or corruption, money movement, auth/authz, schema migrations, concurrency and races, cryptography — anything that can fail without producing an error. |
| 2 | Recoverable but costly. Wrong output reaching users, performance cliffs, retry and idempotency logic, external API contracts. |
| 1 | Fails loudly, cheap to fix. Rendering, formatting, config plumbing, logging, CRUD over a settled schema. |

**Two tie-breakers, both required. Apply precedence first, then
reversibility.** They pull in opposite directions by design: precedence
prevents under-scoring, reversibility prevents scoring a category name
instead of the work.

- **Precedence on multiple matches.** Tasks routinely match anchors in two
  rows — evicting entries from a shared session cache matches both
  `concurrency and races` (3) and `performance cliffs` (2). **The highest
  matching anchor wins.** Name both matches in the justification so the call
  is visible and arguable rather than silent.
- **Reversibility.** Category names are a starting point, not a verdict —
  score the *instance*. Ask whether this specific change is reversible and
  whether it fails loudly. Adding an unused enum value is a schema change,
  but it is reversible and fails immediately: score it 1. A backfill that
  rewrites existing rows cannot be undone: score it 3. **When reversibility
  clears a category match, drop to the score the work would carry if the
  category name did not apply at all** — do not split the difference.
  Without this you hand the user safe, boring work purely because a category
  name matched.

**Comprehension debt — can the user own this later if they did not write it?**

| Score | Anchor |
|---|---|
| 3 | Novel algorithm or load-bearing domain logic. Cannot be re-derived by reading; central to the system's model. |
| 2 | Nontrivial but conventional. Re-following it later costs real time. |
| 1 | Mechanical or generated. Reading it is understanding it. |

**Assignment rule** — apply exactly, with no adjustment for how busy or
capable you feel:

```
user     if max(blast, debt) == 3   or   (blast >= 2 and debt >= 2)
agent    otherwise

A `user` task becomes `split` when the qualifying risk is confined to an
identifiable unit and the remainder is mechanical: the user takes the
unit, you take the remainder.
```

**Score by the riskiest unit in the task, not by its bulk.** A task
containing a novel algorithm scores on that algorithm even if most of its
lines are wiring. `split` is what stops that from handing the user the
wiring too: in a task that computes a pricing tier and then renders it into
a form, the pricing calculation is theirs and the form scaffolding is yours.

**`identifiable` has a concrete test**, borrowed from the seam contract you
will build in Step 3: you can write a stub and failing tests for the unit
*without* also building the remainder. If the two cannot be separated at
that boundary they are one unit, not two — retry logic and the idempotency
it protects are a single correctness problem however cleanly they appear to
divide.

**`mechanical` means the remainder scores `agent` on its own** under this
same rubric. Not "smaller", not "boring" — scored.

**A `split` row's Blast and Debt describe the retained unit**, not the whole
task.

A `split` is therefore not a compromise between the two owners, and not a
lower grade of `user`. It is a `user` task whose mechanical remainder you
have already built.

**Never inflate scores to take work off the user's plate, and never deflate
them to give the user more.** The rule is the whole point: it makes each line
arguable rather than arbitrary. If you think a score is wrong, say so in the
justification — do not quietly change the assignment.
