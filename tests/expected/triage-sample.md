# Expected triage — sample-plan.md

Assignments the skill must produce. A run that differs on any row is a failure.

| # | Task | Blast | Debt | Owner | Why |
|---|------|-------|------|-------|-----|
| 1 | JWT refresh-token rotation | 3 | 2 | yours | Auth failure is silent; max == 3 |
| 2 | TOML config loader | 1 | 1 | agent | Settled schema, loud failure |
| 3 | Token-bucket rate limiter | 3 | 3 | split | Algorithm scores yours; the middleware wiring is mechanical |
| 4 | Structured logging middleware | 1 | 1 | agent | Mechanical, fails loudly |
| 5 | Migration adding users.locked_at | 3 | 1 | yours | Migrations are irreversible; max == 3 |
| 6 | CRUD handlers for /api/notes | 1 | 1 | agent | Generated-shape work over a settled schema |
| 7 | Retry with backoff on payment webhook | 2 | 2 | yours | Second clause: blast >= 2 and debt >= 2 |

Coverage: row 1 exercises `max == 3` via blast. Row 5 exercises `max == 3` where debt is low. Row 7 exercises the `>= 2 and >= 2` clause. Row 3 exercises split detection. Rows 2, 4, 6 must stay with the agent.
