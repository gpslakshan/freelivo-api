# Freelivo — Implementation Plan

Companion to [PRD.md](PRD.md) and [tech-stack.md](tech-stack.md). The PRD says *what*; this says *in what order, in what size chunks, and how you know a chunk is done*.

Phases map to PRD §11, with PRD phase 3 split (jobs, then proposals) and phase 5 split (reviews, then polish), because each half is a full day's work on its own.

| Phase | Theme | Tasks | PRD §11 |
| --- | --- | --- | --- |
| 0 | Foundation & platform packages | 12 | Phase 0 |
| 1 | Auth & token lifecycle | 11 | Phase 1 |
| 2 | RBAC, taxonomy, user/profile surface | 12 | Phase 2 |
| 3 | Jobs | 10 | Phase 3a |
| 4 | Proposals & the accept transaction | 9 | Phase 3b |
| 5 | Contracts, milestones, payments | 11 | Phase 4 |
| 6 | Reviews & ratings | 6 | Phase 5a |
| 7 | Docs, hardening, CI | 9 | Phase 5b |

**80 tasks.** Each is sized to be finishable in one sitting and to leave the build green.

---

## Working rules (apply to every task)

- **Green at every commit.** `make lint test` passes before a task is called done. A task that can't compile on its own is too big — split it.
- **Migration first, then sqlc, then repo, then service, then handler, then routes.** Same order inside every feature; it means the compiler tells you what's missing next.
- **`sqlc generate` is a build step, not a manual chore.** Wire it into the Makefile from task 0.6 onward and re-run it whenever a `.sql` file changes.
- **No feature imports another feature's package internals.** Cross-feature calls go through the other feature's exported `Service` interface, injected in `main.go` (PRD §10).
- **Tests land with the code, not after.** Service-layer table tests in the same task; cross-feature integration tests at the end of each phase.
- **Every endpoint returns the §8 envelope** — no bare JSON, no bare `gin.H` errors.

---

## Phase 0 — Foundation & platform packages

Goal: `make up` boots API + Postgres, migrations run clean, `/health` and `/ready` answer. Nothing domain-specific yet.

| # | Task | Done when |
| --- | --- | --- |
| 0.1 | `git init`, `go mod init`, `.gitignore`, `README` skeleton, directory tree from PRD §10 (empty packages with a doc comment each) | `go build ./...` succeeds on an empty tree |
| 0.2 | `docker-compose.yml`: Postgres 16 with a named volume, healthcheck, and the API service | `docker compose up` gives a Postgres accepting connections |
| 0.3 | `platform/config`: `caarlos0/env/v11` + `godotenv`, typed `Config` struct, `.env.example` committed | Missing required env var fails fast at startup with a named error |
| 0.4 | `platform/database`: pgx v5 pool, `shopspring/decimal` type registration, pool tuning, `Ping` | Pool connects; a smoke test round-trips a `NUMERIC` as `decimal.Decimal` |
| 0.5 | `platform/database`: `WithTx(ctx, fn)` helper — begin, rollback-on-error/panic, commit | Unit test proves rollback on error and on panic |
| 0.6 | `golang-migrate` wiring + `sqlc.yaml` (pgx/v5 engine, decimal override) + Makefile targets: `up`, `down`, `migrate-up`, `migrate-down`, `migrate-new`, `sqlc`, `lint`, `test`, `cover`, `run` | `make migrate-new name=x` produces an up/down pair; `make sqlc` runs clean |
| 0.7 | Migration `000001`: `CREATE EXTENSION citext, pgcrypto`; shared `updated_at` trigger function | Up and down both run clean, twice in a row |
| 0.8 | `platform/response`: success + error envelopes (§8.1/§8.2), the `ErrorCode` constants from §8.3, and one `AbortWithError` helper | Table test covers every code → HTTP status mapping |
| 0.9 | `platform/validator`: bind-and-validate helper turning `validator.ValidationErrors` into `details[]` field/issue pairs; custom tags (`future_date`, ISO-3166 `country`) | Malformed JSON → `MALFORMED_JSON`; bad field → `VALIDATION_ERROR` with populated details |
| 0.10 | `middleware`: RequestID (ULID, echoed as `X-Request-ID`), slog structured logger, Recovery mapping panics to `INTERNAL_ERROR`, CORS | A panicking test route returns a 500 envelope carrying the request ID; the log line has it too |
| 0.11 | `internal/server/router.go` + `cmd/api/main.go`: middleware chain, graceful shutdown on SIGINT/SIGTERM, `GET /health` (liveness) and `GET /ready` (DB ping) | `make up` → both endpoints 200; `/ready` returns 503 with Postgres stopped |
| 0.12 | `platform/pagination`: parse `page`/`per_page` (default 20, max 100), build the `meta` block | Unit test covers clamping, negative input, and `total_pages` rounding |

**Exit criteria:** `make up` boots the stack; migrations run clean up and down; `/health` and `/ready` behave; a deliberately panicking route returns a well-formed envelope.

---

## Phase 1 — Auth & token lifecycle

Goal: the full token lifecycle from PRD §4, including reuse detection. This is the phase that unblocks everything else.

| # | Task | Done when |
| --- | --- | --- |
| 1.1 | Migration `000002`: `user_role`/`user_status` enums, `users` table, unique index on `email` where `deleted_at IS NULL` | Duplicate active email rejected at DB level; a soft-deleted email can be reused |
| 1.2 | Migration `000003`: `profiles` (PK = FK to users, cascade) | Deleting a user row cascades the profile |
| 1.3 | Migration `000004`: `refresh_tokens` with unique `token_hash`, index on `(user_id, revoked_at)` | Runs clean; index shows in `\d refresh_tokens` |
| 1.4 | `platform/hash`: bcrypt cost 12 wrapper; `platform/jwtutil`: sign/parse HS256 with the §4.2 claim set, issuer + expiry validation | Tests: wrong secret, expired token, wrong issuer, tampered payload all rejected distinctly |
| 1.5 | `auth/repository.go`: user queries (create with profile, by email, by ID) + refresh-token queries (insert, find by hash, revoke one, revoke all for user) via sqlc | Repo integration test against testcontainers Postgres passes |
| 1.6 | `POST /auth/register`: create user + profile in one transaction, role from body restricted to `client`/`freelancer` | 201 with user payload and no `password_hash` anywhere; `admin` in body → 400 |
| 1.7 | `POST /auth/login`: verify password, reject `suspended`/soft-deleted, issue access + refresh pair, record user agent + IP | Wrong password and unknown email return the same 401 in the same time envelope |
| 1.8 | `POST /auth/refresh` with **rotation**: revoke presented token, issue a new pair, all in one transaction | Old token fails on second use |
| 1.9 | **Reuse detection**: presenting an already-revoked token revokes the entire family for that user | Integration test: rotate → replay the old token → both the replayed and the current token are dead |
| 1.10 | `AuthRequired` middleware: parse, validate, load `user_id`/`role` into the Gin context, reject suspended/deleted users | Table test covering missing header, malformed bearer, expired (`TOKEN_EXPIRED`), invalid (`TOKEN_INVALID`), suspended (`ACCOUNT_SUSPENDED`) |
| 1.11 | `POST /auth/logout`, `POST /auth/logout-all`, `GET /auth/me`, `PATCH /auth/password` (revokes all sessions) | Password change invalidates every refresh token; `/auth/me` returns user + profile |

**Exit criteria:** full lifecycle passes integration tests including reuse detection; no endpoint anywhere leaks `password_hash`.

---

## Phase 2 — RBAC, taxonomy, user & profile surface

Goal: the permission machinery plus the two admin-managed catalogs and the public user directory.

| # | Task | Done when |
| --- | --- | --- |
| 2.1 | `RequireRole(roles ...string)` middleware | Table test: each role against each guard, plus anonymous |
| 2.2 | Ownership-guard pattern: a small helper that services use to distinguish 403 from 404 per PRD §5 (invisible resource → 404) | Documented in code with the rule stated; unit test on the decision function |
| 2.3 | Migration `000005`: `categories` (self-referencing `parent_id`), unique `slug` | Runs clean; a two-level tree inserts |
| 2.4 | Migration `000006`: `skills`, unique `name`/`slug` | Runs clean |
| 2.5 | Migration `000007`: `freelancer_skills` composite PK, `years_experience` CHECK 0–50 | Duplicate `(user_id, skill_id)` rejected |
| 2.6 | `taxonomy`: categories CRUD (admin), depth ≤ 2 enforced in service, delete blocked with children or jobs → 409 | Creating a grandchild → 422; deleting a parent with children → 409 |
| 2.7 | `GET /categories?tree=true`: single query, assembled into a tree in Go (no per-node query) | Query count asserted at 1 in the test |
| 2.8 | `taxonomy`: skills CRUD (admin) + `GET /skills?q=` public search | Non-admin write → 403 |
| 2.9 | `user`: `GET /users/:id` public profile + `PATCH /users/me/profile`, with role-appropriate field rules (`headline`/`hourly_rate` freelancer-only, `company_name` client-only) | A client setting `hourly_rate` → 422 |
| 2.10 | `user`: `PUT /users/me/skills` (replace whole set, one transaction), `DELETE /users/me/skills/:skillId`, `GET /users/:id/skills` | Replace is atomic; unknown `skill_id` → 422, set unchanged |
| 2.11 | `user`: `GET /users?role=&skill=&country=` directory with pagination — one query with joins, no N+1 | Filter combinations covered; query count asserted |
| 2.12 | `admin`: `PATCH /admin/users/:id/status`, `PATCH /admin/users/:id/role`, `DELETE /users/me` (soft delete) | Suspending a user makes their existing access token fail with `ACCOUNT_SUSPENDED` |

**Exit criteria:** a single table-driven test walks the entire PRD §5.1 permission matrix — every row × every role — and passes.

---

## Phase 3 — Jobs

Goal: jobs CRUD, the M:N skill links, and the filtering/search/pagination surface.

| # | Task | Done when |
| --- | --- | --- |
| 3.1 | Migration `000008`: `budget_type`/`exp_level`/`job_status` enums, `jobs` table with the budget CHECK, `proposals_count`, soft delete | `budget_max < budget_min` rejected at DB level |
| 3.2 | Migration `000009`: `job_skills` composite PK, both FKs cascade | Runs clean |
| 3.3 | Migration `000010`: indexes — `(status, created_at DESC)`, `(category_id)`, `(client_id)`, GIN full-text on `title \|\| description` | `EXPLAIN` on the feed query shows an index scan |
| 3.4 | `job`: create (client only, status `draft`, deadline must be future, description ≥ 50 chars) with skill links in one transaction | Past deadline → 422; skills attached atomically |
| 3.5 | `job`: `GET /jobs/:id` — drafts visible only to owner/admin, others get 404 | Non-owner reading a draft gets 404, not 403 |
| 3.6 | `job`: `PATCH /jobs/:id` with the §7.1 edit rule — full edit while `draft`/`open` with zero proposals, thereafter only `description` and `deadline` | Editing `budget_min` on a job with proposals → 409 `INVALID_STATE_TRANSITION` |
| 3.7 | `job`: `DELETE /jobs/:id` soft delete (owner/admin), excluded from all reads afterwards | Deleted job 404s on every read path |
| 3.8 | `job`: `POST /jobs/:id/publish` (requires ≥ 1 skill + category) and `POST /jobs/:id/close` | Publishing a skill-less job → 422 |
| 3.9 | `job`: `GET /jobs` — `q`, `category_id`, `skill_id[]`, budget range, `experience_level`, `status`, three sorts, pagination | Dynamic filters built with parameterized args only; skill filter doesn't duplicate rows |
| 3.10 | `job`: `GET /jobs/me` (client's own, drafts included) + batched skill loading on every list endpoint | N+1 test: listing 20 jobs with skills issues ≤ 2 queries |

**Exit criteria:** the jobs feed answers every filter/sort combination in the PRD with no N+1 and no string-concatenated SQL.

---

## Phase 4 — Proposals & the accept transaction

Goal: the phase's centerpiece is task 4.6 — one atomic multi-table state change across three features.

| # | Task | Done when |
| --- | --- | --- |
| 4.1 | Migration `000011`: `proposal_status` enum, `proposals` table, `UNIQUE (job_id, freelancer_id)` | Second proposal from the same freelancer rejected at DB level |
| 4.2 | Migration `000012`: `contracts` with unique `job_id` and unique `proposal_id` | Runs clean (table needed now so the accept transaction can write it) |
| 4.3 | `proposal`: `POST /jobs/:id/proposals` — job must be `open`, bid within `[budget_min*0.5, budget_max*2]`, self-bid asserted against, `proposals_count` incremented in the same transaction | Out-of-range bid → 422; counter matches row count under concurrent inserts |
| 4.4 | `proposal`: `GET /jobs/:id/proposals` (job owner/admin only), `GET /proposals/me`, `GET /proposals/:id` (author / job owner / admin) | A competing freelancer gets 404 on someone else's proposal |
| 4.5 | `proposal`: `PATCH /proposals/:id` (author, `pending` only) and `POST /proposals/:id/withdraw` | Editing an accepted proposal → 409 |
| 4.6 | **`proposal.Service.Accept`** — one transaction: proposal → `accepted`, siblings → `rejected`, job → `in_progress`, contract inserted. Calls `job` and `contract` repo methods that accept a `pgx.Tx` | Integration test asserts all four effects; an injected failure at the last step leaves zero changes |
| 4.7 | Define the cross-feature `Service` interfaces (`job.Service`, `contract.Service`) that `proposal` depends on, wire them in `main.go`, generate mocks with `uber-go/mock` | `proposal` package tests run with mocks and no DB |
| 4.8 | `proposal`: `POST /proposals/:id/reject` (job owner) | Rejecting an already-accepted proposal → 409 |
| 4.9 | Concurrency test: two clients accepting two proposals on the same job simultaneously | Exactly one succeeds; the other gets 409, and the DB shows one contract |

**Exit criteria:** accepting a proposal atomically rejects siblings and creates a contract; the failure path leaves no partial writes.

---

## Phase 5 — Contracts, milestones, payments

Goal: the milestone state machine and the simulated escrow ledger, with the money arithmetic done in `decimal`.

| # | Task | Done when |
| --- | --- | --- |
| 5.1 | Migration `000013`: `milestone_status` enum, `milestones` with `amount > 0` and `sequence` | Runs clean |
| 5.2 | Migration `000014`: `payment_status` enum, `payments` with `CHECK (net_amount = amount - platform_fee)` | A row violating the identity is rejected |
| 5.3 | `contract`: `GET /contracts` (own only) and `GET /contracts/:id` (party/admin) | A non-party gets 404 |
| 5.4 | `contract`: `POST /contracts/:id/milestones` — client party, all milestones created in one transaction, `SUM(amount)` must equal `agreed_amount` | Sum mismatch → 422, nothing inserted |
| 5.5 | Same transaction inserts one `escrowed` payment per milestone with `platform_fee = 0`, `net_amount = 0` | Payment rows match milestone rows 1:1 with matching amounts |
| 5.6 | `contract`: `GET /contracts/:id/milestones` (party), batched | One query for the set |
| 5.7 | `POST /milestones/:id/submit` — freelancer party, `pending` → `submitted` | Submitting an approved milestone → 409 |
| 5.8 | `POST /milestones/:id/approve` — client party; in one transaction: milestone → `approved`, payment → `released`, `platform_fee = amount * PLATFORM_COMMISSION_RATE`, `net_amount = amount - platform_fee`, all in `decimal` | Fee arithmetic exact at the cent for rates like 0.10 and 0.075; CHECK never trips |
| 5.9 | `POST /milestones/:id/reject` — `submitted` → `pending` | Round-trip submit → reject → submit works |
| 5.10 | `POST /contracts/:id/complete` — client party, requires all milestones `approved`, sets contract `completed` and job `completed` | Completing with a pending milestone → 422 |
| 5.11 | `POST /contracts/:id/cancel` — either party; all `escrowed` payments → `refunded` with fees left at 0, contract and job → `cancelled` | Already-released payments untouched by a later cancel |

**Exit criteria:** the milestone sum constraint holds; approval releases exactly one payment with correct commission; cancellation refunds only what's still escrowed.

---

## Phase 6 — Reviews & ratings

| # | Task | Done when |
| --- | --- | --- |
| 6.1 | Migration `000015`: `reviews` with rating CHECK 1–5, `UNIQUE (contract_id, reviewer_id)`, `CHECK (reviewer_id <> reviewee_id)` | Self-review rejected at DB level |
| 6.2 | `POST /contracts/:id/reviews` — party only, contract must be `completed`, 30-day window, reviewee derived from the contract (never from the body) | Review on an active contract → 422; day-31 review → 422 |
| 6.3 | `GET /users/:id/reviews` — paginated, with reviewer summary joined | No N+1 across the page |
| 6.4 | Aggregate rating (`AVG` + count) exposed on the public profile, computed in the profile query | A user with no reviews returns null/0, not an error |
| 6.5 | `DELETE /reviews/:id` — admin only | Non-admin → 403 |
| 6.6 | End-to-end integration test: register both parties → post job → publish → propose → accept → milestones → submit/approve → complete → mutual reviews | The whole happy path passes in one test against a real Postgres |

**Exit criteria:** the full marketplace lifecycle passes as a single end-to-end test.

---

## Phase 7 — Docs, hardening, CI

| # | Task | Done when |
| --- | --- | --- |
| 7.1 | Rate limiting: `golang.org/x/time/rate` — 5/min per IP on login and register, 100/min per token globally, `429` + `RATE_LIMITED` | Test trips both limiters and confirms the envelope |
| 7.2 | Logging pass: request ID on every line, structured fields, and an explicit check that no token, password, or hash is ever logged | Grep-based test over a captured log buffer during the e2e run |
| 7.3 | `swaggo/swag` annotations across all handlers; Swagger UI at `/swagger` | Every endpoint in PRD §6 appears with request/response schemas |
| 7.4 | Export `docs/openapi.yaml` + a Postman collection; ERD diagram in `docs/` | Collection runs the happy path against a local stack |
| 7.5 | Seed command: admin user, category tree, skill catalog, a few demo users | `make seed` gives a stack you can explore immediately |
| 7.6 | Coverage pass on the service layer to ≥ 70% | `make cover` reports ≥ 70% on `internal/*/service.go` |
| 7.7 | `golangci-lint` config + fix everything it finds | `make lint` is clean |
| 7.8 | Multi-stage Dockerfile on `distroless/static`; `air` for local live reload | Image builds and runs; container is non-root and under ~20 MB |
| 7.9 | GitHub Actions: lint → test (Postgres service container) → build | Green on a pull request |

**Exit criteria:** Swagger reflects every endpoint; coverage ≥ 70%; CI green.

---

## Sequencing notes

- **Phase 0 and 1 are strictly serial** — everything downstream needs the envelope, the tx helper, and `AuthRequired`. Don't parallelize here.
- **From phase 2 on, taxonomy (2.3–2.8) and the user surface (2.9–2.12) are independent** and can be done in either order.
- **Phase 4 depends on the `contracts` table** (task 4.2) even though contracts aren't exposed until phase 5. That's why the migration lands early.
- **Task 4.7 is the architectural fork.** Define the cross-feature `Service` interfaces the first time you actually need one, not before — but do define them there, or `proposal` will reach into `job`'s repository and the package boundary from PRD §10 is gone for good.
- **`decimal` from the start** (task 0.4). Retrofitting money types after phase 5 means rewriting every payment path.

## Deferred to stretch goals (PRD §12)

Notifications/outbox, keyset pagination, saved jobs, Redis, audit log, and ranked full-text search are out of scope for these 80 tasks. The GIN index in task 3.3 leaves the door open for ranked search without a migration rewrite.
