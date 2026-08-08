# Expected triage — sample-plan.md

Assignments the skill must produce. A run whose Owner column differs on any row is a failure.

| # | Task | Blast | Debt | Owner | Why |
|---|------|-------|------|-------|-----|
| 1 | JWT refresh-token rotation | 3 | 2 | user | Auth failure is silent; max == 3 |
| 2 | TOML config loader | 1 | 1 | agent | Settled schema, loud failure |
| 3 | Token-bucket rate limiter | 3 | 3 | split | Algorithm scores user; the middleware wiring is mechanical |
| 4 | Structured logging middleware | 1 | 1 | agent | Mechanical, fails loudly |
| 5 | Migration adding users.locked_at | 2 | 1 | agent | Migration, but a nullable add drops cleanly and fails loudly |
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
  score blast 3 merely because "migration" matches. At blast 2 / debt 1 the
  rule yields **agent**, not user. This row's owner is `agent`.
- **Row 8 — judgment without vocabulary.** Real silent-corruption risk,
  described using none of the rubric's anchor words. A run that scores this
  low is keyword-matching instead of judging.
- **Row 9 — precedence.** Matches `money movement` (3) and `retry and
  idempotency logic` (2) at once. Must resolve to 3.
- **Row 10 — reversibility, opposite direction.** "Migration" fires the
  blast-3 anchor by name; the instance is reversible and loud, so it lands
  with the agent.
