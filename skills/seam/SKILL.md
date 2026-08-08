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

**When several boundaries pass the stub test**, take the one leaving the
smallest retained unit whose remainder still scores `agent`. An irreversible
write always stays inside the retained unit — never take for yourself the
part that cannot be undone.

**The remainder must be task-sized.** If what is left after removing the
retained unit would not have stood as its own task in a plan — a single
entrypoint, one call site, a config line — there is nothing worth splitting.
Keep the whole task with the user. A split exists to spare them mechanical
bulk; where there is no bulk, it is only ceremony.

**A `split` row's Blast and Debt describe the retained unit**, not the whole
task.

A `split` is therefore not a compromise between the two owners, and not a
lower grade of `user`. It is a `user` task whose mechanical remainder you
have already built.

**Never inflate scores to take work off the user's plate, and never deflate
them to give the user more.** The rule is the whole point: it makes each line
arguable rather than arbitrary. If you think a score is wrong, say so in the
justification — do not quietly change the assignment.

## Step 2 — Write the manifest, then stop

Write `docs/seam/<plan-slug>.md` in the target repo, where `<plan-slug>` is
the plan's filename without its date prefix or extension.

```markdown
# seam manifest — <plan name>

plan: docs/plans/<plan-file>.md
status: draft

| # | Task | Blast | Debt | Owner | Files | Status | Why |
|---|------|-------|------|-------|-------|--------|-----|
| 1 | Invoice tax calculation | 3 | 3 | user | internal/billing/tax.go | pending | Wrong tax is silent, and irreversible once invoiced |
| 2 | Locale string loader | 1 | 1 | agent | internal/i18n/*.go | pending | Settled schema, loud failure |
| 3 | Search result ranking | 2 | 3 | split | internal/search/*.go | pending | Query plumbing is mechanical; the ranking function is not |
```

- `Files` lists every path the task will touch. Derive it from the plan; if
  the plan does not say, infer from the repo layout and mark the cell
  `_(verify)_` so the user can correct it. This column is load-bearing —
  Step 4 enforces ownership from it.
- `Status` is one of `pending`, `in-progress`, `blocked`, `green`,
  `integrated`.
- `Why` is one line. It justifies the score, not the task.

Then **post the table in chat and stop.** Ask exactly:

> Manifest written to `docs/seam/<plan-slug>.md`. Redraw any line you
> disagree with, or say go.

**Do not begin any task while `status: draft`.** When the user approves,
change the file's `status:` to `approved` and only then proceed. If the user
redraws a line, update the row, re-post, and ask again — a redraw is not an
approval.

If the user says something ambiguous ("looks fine, though task 3 seems
heavy"), treat it as a redraw request and ask, rather than reading it as
consent.

**A redraw outranks the rule.** The user may reassign any row, and their
call stands even where the rubric contradicts it — they are the approver,
and the scores exist to inform that judgment, not to overrule it. Record the
override in the row's `Why` cell and leave the scores untouched. **Never
re-score a row to make its new owner look rule-derived**: that hides a human
decision behind arithmetic, and the manifest stops being an honest record of
who chose what.

## Step 3 — Build the seam

For each task owned `user` or `split`, produce four artifacts **before the
user writes anything**:

1. **The stub**, in the real file — signature, doc comment stating intent and
   invariants, and an explicit list of non-goals. Its body raises or returns
   a not-implemented error. Never a plausible guess: a half-right body is
   worse than an empty one, because it anchors the design.
2. **Failing tests** encoding the required behavior, including the edge cases
   the plan names.
3. **The surrounding wiring**, already calling that stub, so the user sees
   their piece in context.
4. **The spec gate.** Post the test names and what each asserts, then ask:

   > These tests are the spec for `<unit>`. Agree, or edit them?

**Stop there and wait.** The gate exists because your tests would otherwise
silently pre-decide the user's design. One screen of review now is cheaper
than discovering the constraint halfway through an implementation.

**Posting the spec gate *is* the handoff.** That is the moment the row's
files stop being yours and become the user's. There is no separate status
for it — the gate is the event. Everything you intend to write into those
files must already be there when you post.

**A defect you find in your own artifact after handoff is not yours to
repair.** If a stub you wrote aborts the run, or a test you wrote asserts the
wrong thing, report the fault and state what the correct version would be.
Do not edit it, and **do not offer to** — an offer to edit a user-owned file
is a disguised handoff, and handoffs are theirs to invoke. If they want it
fixed, that is an explicit return of the file, recorded like any other
reassignment.

**The user may set any row `in-progress` themselves**, including one whose
seam you never built. Do not edit a row to describe an event that did not
happen: if they are working in a `pending` row's files, that work is theirs
and the row is simply stale. Say so; do not quietly correct it.

**Test immutability.** Once the user accepts the tests, you may not modify,
relax, or add to them without asking. Not to make them pass, not because the
implementation took a different shape, not to be helpful. Moving the
goalposts mid-implementation is the single most damaging thing you can do
here. If a test looks wrong once the user is working, say so and ask.

**When the user is the one asking you to relax a test**, this rule still
binds — it is a duty to explain, not a veto. State what the test protects
and what removing it would let through, then comply if they hold. Their call
stands; they simply make it knowing the cost. Record the decision in the
row's `Why` cell rather than letting it vanish into a quiet edit.

Six rules govern how you build a seam:

- **One task at a time.** Build a seam for the task the user asks for, not
  for every `user` and `split` row at once. Several open spec gates defeat
  the point of having a gate. A gate closes once the user accepts its tests;
  you may build the next seam then.
- **The stub must fail without aborting the run.** Where the language allows
  it, return a not-implemented error rather than raising one, so the
  remaining tests still report. A stub that kills the test binary hides the
  rest of the spec behind the first assertion.
- **Name what the signature decides.** A half-right body anchors the design
  — but so does a signature. If your contract forces a design choice
  (injecting a clock so refill can be tested without sleeping), say so at
  the gate and offer the alternative. The gate exists to surface those, not
  merely to approve assertions.
- **Test your own remainder.** Those tests are yours. They are not part of
  the gate and not covered by test immutability.
- **Unassigned scaffolding is yours.** A module file or build config the
  plan gives to no task still has to exist. Create it, and name it at the
  gate so the user is never surprised by files outside the manifest.
- **Status transitions.** Set a row `in-progress` when you build its seam,
  `green` when the user's tests pass, and `integrated` once the suite is
  clean. Only the user sets `blocked`.

`user` and `split` differ only in what you do **after** the handoff. In both
cases you build the contract. In a `split` you have also already implemented
the mechanical remainder of that task; in a `user` the whole implementation
is the user's.

## Step 4 — Work in parallel

While the user implements their piece, execute your own manifest tasks.

**Ownership is per file, and it is absolute.** Never write, refactor,
reformat, or "quickly fix" a file the manifest currently assigns to the user
— including their tests, including a one-line obvious repair, including when
they ask you to while they are mid-task. If they want you to take it, that is
rung 4 of the stall ladder and it is an explicit handoff, not a favor.

**Ownership transfers exactly once per task.** The files of a `user` or
`split` row **start as yours** — you are the one who writes the stub, the
failing tests and the wiring, which is the only reason Step 3's artifacts can
exist at all — and at handoff they become the user's for the rest of the
task. A split hands off only when your side of it is complete: a half-built
remainder would put both of you in the same file at once, which the design
does not permit.

The rule above bites **after** the transfer. Before it, the file is yours to
create; after it, it is untouchable. If the user has already written into a
file whose seam you never built, that work is theirs and the transfer has
effectively already happened — do not overwrite it.

**Reassignment survives approval.** A row can change owners at any point, not
only while the manifest is `draft`, by the same procedure: update the row,
record the override in `Why`, leave the scores alone. The one difference
after approval is that in-flight work may exist, so an explicit handoff must
say whether the work already in the file is kept or discarded. Ask; never
assume.

**Progress signal.** The user's tests going green is the only signal. Do not
poll, do not ask how it is going, do not check in mid-task.

**When you run out of your own tasks, stop and report.** Never start a task
owned by the user, under any circumstances, however idle you are and however
obvious the implementation looks. Report like this:

> My tasks are done: <list>. Yours outstanding: <list>. Full suite: <state>.

Then go to Step 5.

## Step 5 — The stall ladder

Stalling is expected, not a failure. It has one defined exit.

**Trigger.** You have no clock — you exist only when invoked. The trigger is
finishing your own tasks and finding the user's tests still red. Offer the
ladder then, **once**.

**Offer it exactly once.** If the user does not take a rung, say nothing
further about it. Do not re-offer on your next turn, do not append a reminder
to an unrelated answer, do not ask how it is going. Repeated prompting is
worse than none — it converts a stall into a reason to leave.

| Rung | You give | You must not |
|---|---|---|
| 1 · hint | One nudge toward the idea | Write code |
| 2 · bigger hint | The approach, concretely | Write code |
| 3 · draft | A first attempt for them to reshape | Present it as finished |
| 4 · take it | Full handoff; the task becomes yours | Offer this yourself |

**Rung 4 is never offered by you.** It exists, it is always available, and
the user invokes it. The user's stated goal is to stay on the piece; treating
handoff as the natural next step works against the reason they took it. When
they do invoke it, take it cleanly — no relitigating, no "are you sure".

Post the offer as:

> Your tasks are still open. Want a hint, a bigger hint, or a draft to
> reshape?

## Step 6 — Integrate and hand to reef

When the user's tests go green:

1. Run the full suite. If something unrelated broke, report it; do not fix
   the user's code to make it pass.
2. Update the manifest: their rows to `green`, then `integrated` once the
   suite is clean.
3. Hand the complete diff to the `reef` skill for the gated walkthrough, with
   **every hunk labeled `user` or `agent`** so authorship is visible during
   review.
4. **Review the user's hunks as well as your own.** Their hunks are by
   definition the highest-blast-radius code in the plan; leaving them
   unreviewed would mean the riskiest code is the only code nothing checks.
   Frame findings as a risk check, not a critique: state the condition under
   which the code is wrong, not an opinion about the code.
5. **Never silently repair the user's code.** If something is wrong, report
   it and stop. A silent fix costs them the comprehension they took the task
   to gain — which defeats the entire point of the split.

If the full suite is red because of *your* code, fix it before handing over.

**If the user reports green and the suite disagrees, Step 6 has not begun.**
Report what the runner printed and which files the failures are confined to,
then go back to Step 4. The row stays where it is: `green` is not true, and
only the user sets `blocked`. **Do not diagnose beyond the failure output** —
an explanation of why their code fails is a ladder rung, and rungs are asked
for, never volunteered. Someone who believes they are finished experiences an
unsolicited diagnosis as a correction rather than help. Let the failing
output speak, and offer the ladder once.

**Report recorded status as recorded.** If a row is marked `integrated` but
its files are not in the checkout, say that you are reporting the manifest,
not something you confirmed. Never restate a status you cannot verify as
though you had checked it.
