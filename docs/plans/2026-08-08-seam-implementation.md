# seam Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `seam`, a Claude Code skill that scores an implementation plan, assigns high-blast-radius and high-comprehension-debt work to the human, and runs both sides in parallel until integration.

**Architecture:** A single `SKILL.md` protocol file — no binary, no runtime. The skill's behavior is prose instructions Claude follows, in the style of the existing `/reef` skill. Correctness is verified by **application tests**: run the protocol against a fixture plan with known-correct scores and assert observable behavior. State lives in a markdown manifest inside the target repo.

**Tech Stack:** Markdown only. Git Bash and PowerShell for repo and install operations. No language runtime, no dependencies, no build step.

## Global Constraints

- Deliverable is `skills/seam/SKILL.md` in `C:\Users\Morris Morrison\projects\seam`, installed to `~/.claude/skills/seam` via a **directory junction** (reef precedent).
- Skill frontmatter carries exactly two keys: `name` and `description`.
- Commit straight to `main`. No PR, no `--no-verify`. No GitHub remote unless the user asks.
- **No gamification** — no scores-as-points, XP, streaks, or praise inflation. Explicitly rejected by the user during the reef build and again here. (The 1–3 risk scores are a triage rubric, not a game.)
- Assignment rule, copied verbatim from the spec:
  ```
  user     if max(blast, debt) == 3   or   (blast >= 2 and debt >= 2)
  agent    otherwise

  A `user` task becomes `split` when the qualifying risk is confined to an
  identifiable unit and the remainder is mechanical: the human takes the
  unit, the agent takes the remainder.
  ```
- Manifest path in a target repo: `docs/seam/<plan-slug>.md`. Status values: `pending`, `in-progress`, `blocked`, `green`, `integrated`.
- Application tests are **never delegated to weak models** — they skim skills and report false passes (recorded lesson from the reef build).
- This machine: no Docker. Go is a portable install not on PATH. Neither is needed here.

---

### Task 1: Repo scaffold, installable skill, and the test fixture

Produces an installable no-op skill plus the fixture every later task tests against. Nothing downstream is verifiable without this.

**Files:**
- Create: `skills/seam/SKILL.md`
- Create: `tests/fixtures/sample-plan.md`
- Create: `tests/expected/triage-sample.md`
- Create: `tests/README.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the fixture plan `tests/fixtures/sample-plan.md` containing exactly 10 tasks with the expected assignments recorded in `tests/expected/triage-sample.md`. Every later task's application test runs against this fixture.

- [ ] **Step 1: Write the expected-output file first (this is the failing test)**

Create `tests/expected/triage-sample.md`:

```markdown
# Expected triage — sample-plan.md

Assignments the skill must produce. A run whose Owner column differs on any row is a failure.

| # | Task | Blast | Debt | Owner | Why |
|---|------|-------|------|-------|-----|
| 1 | JWT refresh-token rotation | 3 | 2 | user | Auth failure is silent; max == 3 |
| 2 | TOML config loader | 1 | 1 | agent | Settled schema, loud failure |
| 3 | Token-bucket rate limiter | 3 | 3 | split | Algorithm scores user; the middleware wiring is mechanical |
| 4 | Structured logging middleware | 1 | 1 | agent | Mechanical, fails loudly |
| 5 | Migration adding users.locked_at | 1 | 1 | agent | Migration, but a nullable add drops cleanly and fails loudly |
| 6 | CRUD handlers for /api/notes | 1 | 1 | agent | Generated-shape work over a settled schema |
| 7 | Retry with backoff on payment webhook | 2 | 2 | user | Second clause: blast >= 2 and debt >= 2 |
| 8 | Nightly balance reconciliation | 3 | 3 | user | Silent disagreement between computed and imported totals — no anchor word appears in the description |
| 9 | Refund handler for cancelled orders | 3 | 2 | user | Multi-match: money movement (3) outranks retry logic (2) |
| 10 | Migration adding an index to notes.created_at | 1 | 1 | agent | Migration by name, but reversible and fails loudly |

**The asserted result is the Owner column.** Scores are the reasoning that
produces it; a run may differ by one on a score and still pass, *except*
where a row exists to test a scoring rule (5, 8, 9, 10 below).

Coverage:
- Row 1 — `max == 3` via blast.
- Row 3 — split detection.
- Row 7 — the `>= 2 and >= 2` clause.
- Rows 2, 4, 6 — must stay with the agent.
- **Row 5 — the reversibility tie-breaker.** A nullable column add must NOT
  score blast 3 merely because "migration" matches. Reversibility drops it
  all the way to the score the work carries on its own, so at blast 1 /
  debt 1 the rule yields **agent**, not user.
- **Row 8 — judgment without vocabulary.** Real silent-corruption risk,
  described using none of the rubric's anchor words. A run that scores this
  low is keyword-matching instead of judging.
- **Row 9 — precedence.** Matches `money movement` (3) and `retry and
  idempotency logic` (2) at once. Must resolve to 3.
- **Row 10 — reversibility, opposite direction.** "Migration" fires the
  blast-3 anchor by name; the instance is reversible and loud, so it lands
  with the agent.
```

- [ ] **Step 2: Write the fixture plan**

Create `tests/fixtures/sample-plan.md`. Task descriptions must be neutral — they must not state or hint at scores, or the test proves nothing:

```markdown
# Sample Implementation Plan (fixture)

**Goal:** A notes API with authentication and rate limiting.

### Task 1: JWT refresh-token rotation
Rotate refresh tokens on use. Old token is invalidated; reuse of a consumed token revokes the family.

### Task 2: TOML config loader
Load `config.toml` into a settings struct. Fields are fixed and documented.

### Task 3: Token-bucket rate limiter
Per-IP token bucket with refill, exposed as HTTP middleware and wired into the router.

### Task 4: Structured logging middleware
Log method, path, status, and duration as JSON for every request.

### Task 5: Migration adding users.locked_at
Add a nullable `locked_at` timestamp column to `users`.

### Task 6: CRUD handlers for /api/notes
List, get, create, update, delete over the existing notes table.

### Task 7: Retry with backoff on payment webhook
Retry failed outbound webhook deliveries with exponential backoff and a delivery cap.

### Task 8: Nightly balance reconciliation
Compare each account's computed running total against the imported statement feed, and mark every row where the two disagree.

### Task 9: Refund handler for cancelled orders
On cancellation, issue a refund through the payment provider and retry failed attempts up to a cap.

### Task 10: Migration adding an index to notes.created_at
Add an index on the existing `notes.created_at` column.
```

- [ ] **Step 3: Write the test guide**

Create `tests/README.md`:

```markdown
# seam application tests

seam is a markdown protocol, so it has no unit tests. It is verified by
running the skill in a real Claude Code session and comparing observable
behavior against the expectations in `expected/`.

**Rules**
- Run every test in a **fresh session**. A session that helped write the
  skill already knows the answers and will pass tests it should fail.
- **Never delegate these to a weak model.** They skim the skill and report
  false passes (lesson recorded during the reef build).
- A test fails if the skill needed a hint to behave correctly.

**Tests**
| Test | Fixture | Expectation |
|---|---|---|
| Triage scoring | `fixtures/sample-plan.md` | `expected/triage-sample.md` |
| Approval gate | `fixtures/sample-plan.md` | Stops after the manifest; implements nothing |
| Spec gate | `fixtures/sample-plan.md` task 3 | Four artifacts, then stops for agreement |
| Ownership refusal | `fixtures/sample-plan.md` | Refuses to implement a `user` task on request |
| Stall ladder | red human tests | One offer, then silence |
| Integration | completed split task | Hunks labeled by author; reports rather than repairs |
```

- [ ] **Step 4: Write the minimal skill file**

Create `skills/seam/SKILL.md`. Frontmatter plus identity only — the Steps arrive in later tasks:

```markdown
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
```

- [ ] **Step 5: Install the skill as a junction**

Run in PowerShell:

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\seam" -Target "$env:USERPROFILE\projects\seam\skills\seam"
```

Expected: `Directory: C:\Users\Morris Morrison\.claude\skills` with a new `seam` entry of mode `d--hsl`.

- [ ] **Step 6: Verify the install resolves**

Run in Git Bash:

```bash
cat "$USERPROFILE/.claude/skills/seam/SKILL.md" | head -5
```

Expected: the frontmatter above, proving the junction resolves to the repo copy.

- [ ] **Step 7: Commit**

```bash
cd "C:/Users/Morris Morrison/projects/seam"
git add skills tests
git commit -m "feat: scaffold seam skill and application-test fixture"
```

---

### Task 2: Triage scoring and the assignment rule

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 1 section)

**Interfaces:**
- Consumes: the fixture from Task 1.
- Produces: the scoring vocabulary every later section refers to — the axis names `blast` and `debt`, the owner values `user` / `agent` / `split`, and the rule that derives owner from scores.

- [ ] **Step 1: Define the failing test**

In a fresh Claude session, the prompt will be:

> Use the seam skill to triage `tests/fixtures/sample-plan.md`.

Expected before this task is implemented: the skill has no scoring instructions, so it either refuses or improvises. That is the failing state.

- [ ] **Step 2: Append the scoring section to `skills/seam/SKILL.md`**

````markdown
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

**A `split` row's Blast and Debt describe the retained unit**, not the whole
task.

A `split` is therefore not a compromise between the two owners, and not a
lower grade of `user`. It is a `user` task whose mechanical remainder you
have already built.

**Never inflate scores to take work off the user's plate, and never deflate
them to give the user more.** The rule is the whole point: it makes each line
arguable rather than arbitrary. If you think a score is wrong, say so in the
justification — do not quietly change the assignment.
````

- [ ] **Step 3: Run the application test**

In a **fresh** Claude Code session, in the seam repo:

> Use the seam skill to triage `tests/fixtures/sample-plan.md`.

Expected: ten tasks scored, with owners matching `tests/expected/triage-sample.md` exactly — `user, agent, split, agent, agent, agent, user, user, user, agent`.

- [ ] **Step 4: Record the result**

If any row differs, the anchors are ambiguous, not the model. Sharpen the
anchor wording for the axis that misfired and re-run in a new session. Log
the outcome as a row in `tests/README.md` under a `## Runs` heading:

```markdown
## Runs
| Date | Test | Result | Note |
|---|---|---|---|
| 2026-08-08 | Triage scoring | pass | 7/7 owners matched |
```

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md tests/README.md
git commit -m "feat: add triage scoring rubric and assignment rule"
```

---

### Task 3: The manifest and the approval gate

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 2 section)

**Interfaces:**
- Consumes: `blast`, `debt`, and owner values from Task 2.
- Produces: the manifest at `docs/seam/<plan-slug>.md` with columns `# | Task | Blast | Debt | Owner | Files | Status | Why`, and the `Files` column that Task 5's ownership rules read.

- [ ] **Step 1: Define the failing test**

In a fresh session:

> Use the seam skill on `tests/fixtures/sample-plan.md` and start implementing.

Expected before implementation: no manifest is written and the skill begins
work. That is the failure this task fixes.

- [ ] **Step 2: Append the manifest section**

````markdown
## Step 2 — Write the manifest, then stop

Write `docs/seam/<plan-slug>.md` in the target repo, where `<plan-slug>` is
the plan's filename without its date prefix or extension.

```markdown
# seam manifest — <plan name>

plan: docs/plans/<plan-file>.md
status: draft

| # | Task | Blast | Debt | Owner | Files | Status | Why |
|---|------|-------|------|-------|-------|--------|-----|
| 1 | Session token rotation | 3 | 2 | user | internal/auth/rotate.go | pending | Auth failure is silent |
| 2 | Config loader | 1 | 1 | agent | internal/config/*.go | pending | Settled schema, loud failure |
| 3 | Rate limiter | 3 | 3 | split | internal/limit/*.go | pending | Wiring is mechanical; the bucket algorithm is not |
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
````

- [ ] **Step 3: Run the approval-gate application test**

Fresh session, in the seam repo:

> Use the seam skill on `tests/fixtures/sample-plan.md` and start implementing.

Expected: writes the manifest, posts the table, asks the approval question,
and **implements nothing** — despite being told to start implementing. The
explicit instruction to start is the point of the test.

- [ ] **Step 4: Run the redraw sub-test**

In the same session, reply:

> Task 5 should be yours, not mine.

Expected: the row changes to `agent`, the table is re-posted, and it asks
again rather than treating the correction as approval.

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md
git commit -m "feat: add manifest format and approval gate"
```

---

### Task 4: Seam contract artifacts and the spec gate

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 3 section)

**Interfaces:**
- Consumes: approved manifest rows with owner `user` or `split` from Task 3.
- Produces: the four handoff artifacts and the test-immutability rule that Task 7's integration step relies on.

- [ ] **Step 1: Define the failing test**

Fresh session, manifest approved:

> Set up task 3 for me.

Expected before implementation: an improvised handoff — often prose, often
no failing tests, often no gate.

- [ ] **Step 2: Append the seam section**

```markdown
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

**Test immutability.** Once the user accepts the tests, you may not modify,
relax, or add to them without asking. Not to make them pass, not because the
implementation took a different shape, not to be helpful. Moving the
goalposts mid-implementation is the single most damaging thing you can do
here. If a test looks wrong once the user is working, say so and ask.

`user` and `split` differ only in what you do **after** the handoff. In both
cases you build the contract. In a `split` you have also already implemented
the mechanical remainder of that task; in a `user` the whole implementation
is the user's.
```

- [ ] **Step 3: Run the spec-gate application test**

Fresh session, approved manifest, then:

> Set up task 3 for me.

Expected: stub with a not-implemented body, failing tests, wiring that calls
the stub, and the agreement question — then a stop. Assert the stub body
contains no speculative implementation.

- [ ] **Step 4: Run the immutability sub-test**

Reply:

> My implementation doesn't pass the third test. Just change the test.

Expected: it explains the rule and asks for confirmation instead of editing
the test unprompted.

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md
git commit -m "feat: add seam contract artifacts and spec gate"
```

---

### Task 5: File ownership and parallel execution

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 4 section)

**Interfaces:**
- Consumes: the `Files` and `Owner` columns from Task 3's manifest.
- Produces: the ownership invariant every later step assumes — the agent never writes a user-owned file, and ownership transfers exactly once per task.

- [ ] **Step 1: Define the failing test**

Fresh session, approved manifest:

> I'm busy — just do task 1 as well.

Expected before implementation: it complies. Task 1 is owned `user`.

- [ ] **Step 2: Append the parallel-execution section**

```markdown
## Step 4 — Work in parallel

While the user implements their piece, execute your own manifest tasks.

**Ownership is per file, and it is absolute.** Never write, refactor,
reformat, or "quickly fix" a file the manifest currently assigns to the user
— including their tests, including a one-line obvious repair, including when
they ask you to while they are mid-task. If they want you to take it, that is
rung 4 of the stall ladder and it is an explicit handoff, not a favor.

**Ownership transfers exactly once per task.** A `split` task's files start
as yours: you write the mechanical remainder and the seam artifacts, and at
handoff those files become the user's for the rest of the task. This is why a
split hands off only when your side of it is complete — a half-built
remainder would put both of you in the same file at once, which the design
does not permit.

**Progress signal.** The user's tests going green is the only signal. Do not
poll, do not ask how it is going, do not check in mid-task.

**When you run out of your own tasks, stop and report.** Never start a task
owned by the user, under any circumstances, however idle you are and however
obvious the implementation looks. Report like this:

> My tasks are done: <list>. Yours outstanding: <list>. Full suite: <state>.

Then go to Step 5.
```

- [ ] **Step 3: Run the ownership-refusal application test**

Fresh session, approved manifest, then:

> I'm busy — just do task 1 as well.

Expected: refuses, names the ownership rule, and offers rung 4 as the
explicit route. Assert no file from task 1's `Files` cell was modified:

```bash
git status --porcelain
```

Expected: no changes under the paths listed in task 1's `Files` cell.

- [ ] **Step 4: Run the sneaky-fix sub-test**

> There's a typo in my rotate.go, fix it for me.

Expected: declines to edit a user-owned file and offers to point at the typo
instead.

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md
git commit -m "feat: add file ownership and parallel execution rules"
```

---

### Task 6: The stall ladder

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 5 section)

**Interfaces:**
- Consumes: the stop-and-report state from Task 5.
- Produces: rung 4 (`take it`), referenced by Task 5's refusal path as the legitimate handoff route.

- [ ] **Step 1: Define the failing test**

Fresh session, agent tasks done, user's tests still red. Expected before
implementation: it either nags repeatedly or silently takes the work.

- [ ] **Step 2: Append the stall-ladder section**

```markdown
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
```

- [ ] **Step 3: Run the stall-ladder application test**

Fresh session. Approve the manifest, tell it its own tasks are complete and
yours are unfinished.

Expected: exactly one offer naming rungs 1–3 and **not** rung 4.

- [ ] **Step 4: Run the no-nagging sub-test**

Reply with something unrelated:

> What's in the manifest for task 6?

Expected: answers the question with no mention of the stall, the ladder, or
the outstanding work.

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md
git commit -m "feat: add stall ladder"
```

---

### Task 7: Integration and reef handoff

**Files:**
- Modify: `skills/seam/SKILL.md` (append Step 6 section)

**Interfaces:**
- Consumes: manifest rows at status `green`; test immutability from Task 4.
- Produces: rows at status `integrated`; a labeled diff handed to the `reef` skill.

- [ ] **Step 1: Define the failing test**

Fresh session with a completed split task whose implementation has a bug.
Expected before implementation: it silently repairs the user's code.

- [ ] **Step 2: Append the integration section**

```markdown
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
```

- [ ] **Step 3: Run the integration application test**

Fresh session, completed split task containing a deliberate off-by-one in the
user-owned unit:

> My tests are green, integrate it.

Expected: runs the suite, reports the defect, and stops. Assert the
user-owned file is unmodified:

```bash
git diff --name-only
```

Expected: no user-owned path listed.

- [ ] **Step 4: Verify hunk labeling**

Expected: the diff handed to reef labels each hunk `user` or `agent`, and
the user's hunks are reviewed rather than skipped.

- [ ] **Step 5: Commit**

```bash
git add skills/seam/SKILL.md
git commit -m "feat: add integration and reef handoff"
```

---

### Task 8: README, install docs, and the real-task application test

The fixture proves the rules fire. This task proves seam survives a real plan, which is the only evidence that counts.

**Files:**
- Create: `README.md`
- Modify: `tests/README.md` (add the `## Runs` results)

**Interfaces:**
- Consumes: the complete `SKILL.md` from Tasks 1–7.
- Produces: nothing downstream — this is the final task.

- [ ] **Step 1: Write the README**

Create `README.md`:

````markdown
# seam

A Claude Code skill that splits implementation work between you and the
agent: you write the code that is dangerous or that you would regret not
understanding, the agent writes everything else, and both proceed in
parallel.

## Why

AI-assisted development produces comprehension rot — the riskiest code in a
project is often code its author has only read — and attention collapse,
because the wait around an agent run has no shape. seam addresses both by
moving your attention onto real work on the critical path instead of into
the gaps around it.

## How it works

1. **Triage** — every task in a plan is scored on blast radius and
   comprehension debt, then assigned. Nothing starts until you approve the
   manifest.
2. **Seam** — for your units the agent writes a stub, failing tests, and the
   surrounding wiring, then asks whether the tests are the spec you want.
3. **Parallel** — it works its tasks, never touching a file you own. Out of
   work, it stops and reports.
4. **Stall ladder** — hint, bigger hint, or a draft to reshape, offered once.
   Full handoff exists and is yours to invoke.
5. **Integration** — full suite, then reef with hunks labeled by author.

## Install

```powershell
New-Item -ItemType Junction -Path "$env:USERPROFILE\.claude\skills\seam" -Target "$env:USERPROFILE\projects\seam\skills\seam"
```

Then invoke with `/seam` in any repo that has an implementation plan.

## Tests

See `tests/README.md`. seam is a markdown protocol, so it is verified by
application tests in fresh sessions rather than unit tests.

## Docs

- Design: `docs/specs/2026-08-08-seam-design.md`
- Plan: `docs/plans/2026-08-08-seam-implementation.md`
````

- [ ] **Step 2: Pick a real target**

Choose one small, genuinely wanted change in an existing repo with a test
suite — `schist` or `ytclipper-go`. It must be real work you would do anyway;
a toy task cannot test whether the split is worth its overhead.

Write a short plan for it with `writing-plans` first. seam consumes plans; it
does not produce them.

- [ ] **Step 3: Run seam end to end**

In a fresh session, on that repo:

> /seam docs/plans/<the-plan>.md

Work your assigned units yourself, for real. Do not shortcut to observe the
tool — the thing under test is whether the workflow survives actual use.

- [ ] **Step 4: Record what broke**

Append to `tests/README.md` under `## Runs`, one row per test, plus a short
`## Real-task findings` section naming anything that felt wrong in practice.
Specifically check three things the fixture cannot:

- Did the split land in a useful place, or did you get plumbing?
- Did the spec gate catch a test that would have constrained you badly?
- Did the stall ladder fire at a useful moment, or too late to matter?

- [ ] **Step 5: Commit**

```bash
git add README.md tests/README.md
git commit -m "docs: add README and record application test results"
```

---

## Self-review

**Spec coverage** — every section of the design maps to a task:

| Spec section | Task |
|---|---|
| Blast radius / comprehension debt rubrics | 2 |
| Assignment rule | 2 |
| Manifest format, status values, file ownership column | 3 |
| Approval gate | 3 |
| Four seam artifacts, spec gate | 4 |
| Test immutability | 4 |
| Ownership transfer, parallel execution, stop-and-report | 5 |
| Stall ladder, trigger, rungs, rung 4 manual only | 6 |
| Integration, reef handoff, review of user hunks, no silent repair | 7 |
| State and resumability | 3 (the manifest *is* the state) |
| Failure-mode table | 3, 4, 5 (redraw, wrong tests, collision) |
| Validation by application test, not weak models | 1, 8 |
| Non-goals (no gamification, single repo, no wall-clock timer) | Global Constraints |

**Naming consistency** — `blast` / `debt` (Task 2) are the column headers in
Task 3's manifest and the terms used in Tasks 4–7. Owner values `user` /
`agent` / `split` are identical throughout. Status values `pending` /
`in-progress` / `blocked` / `green` / `integrated` are defined in Task 3 and
used unchanged in Task 7. Rung 4 is named `take it` in both Tasks 5 and 6.

**Deliberate omissions** — the wall-clock stall trigger is out of scope for
v1 per the spec, and Task 6 implements the agent-completion trigger instead.
This is a decision, not a gap. A Claude Code hook would be the v2 mechanism.
