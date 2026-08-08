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
