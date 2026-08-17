# Freelivo — Product Requirements Document

**Version:** 1.0
**Status:** Draft
**Type:** Learning project (backend API only)
**Stack:** Go 1.22+, Gin, PostgreSQL 16, JWT, Docker

---

## 1. Overview

Freelivo is a REST API for a freelance marketplace where **clients** post jobs and **freelancers** submit proposals. When a proposal is accepted, a **contract** is created, tracked through milestones, and closed with mutual reviews.

The product is deliberately scoped as an **API-only** project. There is no frontend. Every feature exists to exercise one of four learning targets:

| Learning target    | Where it shows up                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| CRUD               | Jobs, proposals, profiles, skills, categories, reviews                                                                         |
| SQL relationships  | 1:1 (user↔profile), 1:N (client→jobs), M:N (freelancer↔skills), self-referencing (category tree), composite keys, soft deletes |
| JWT authentication | Access/refresh token pair, rotation, revocation, middleware                                                                    |
| RBAC               | Three roles, permission matrix, ownership-based authorization                                                                  |

### 1.1 Non-goals

- No real payment processing (payments are simulated state transitions).
- No real-time chat / WebSockets.
- No frontend, mobile app, or admin dashboard UI.
- No email/SMS delivery (log to stdout instead).
- No file storage service — attachments store URLs only.

---

## 2. Personas & Roles

| Role         | Description                                                                                                                  |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `admin`      | Platform operator. Manages categories and skills, suspends users, moderates jobs and reviews. Cannot post jobs or proposals. |
| `client`     | Posts jobs, reviews proposals, awards contracts, releases milestone payments, reviews freelancers.                           |
| `freelancer` | Builds a profile, submits proposals, delivers milestones, reviews clients.                                                   |

A user has exactly **one** role, assigned at registration (`client` or `freelancer`). `admin` is seeded or promoted by another admin only.

---

## 3. Data Model

### 3.1 Entity relationship summary

```
users 1───1 profiles
users 1───N jobs                    (as client)
users 1───N proposals               (as freelancer)
users N───M skills                  (via freelancer_skills)
users 1───N refresh_tokens

categories 1───N categories         (self-referencing parent_id)
categories 1───N jobs

jobs 1───N proposals
jobs N───M skills                   (via job_skills)
jobs 1───1 contracts

proposals 1───1 contracts

contracts 1───N milestones
contracts 1───N reviews             (max 2: one per party)
contracts 1───N payments
```

### 3.2 Tables

#### `users`

| Column                  | Type        | Notes                                         |
| ----------------------- | ----------- | --------------------------------------------- |
| id                      | UUID        | PK, `gen_random_uuid()`                       |
| email                   | CITEXT      | UNIQUE, NOT NULL                              |
| password_hash           | TEXT        | NOT NULL, bcrypt cost 12                      |
| role                    | user_role   | ENUM(`admin`,`client`,`freelancer`), NOT NULL |
| status                  | user_status | ENUM(`active`,`suspended`), DEFAULT `active`  |
| email_verified          | BOOLEAN     | DEFAULT false                                 |
| created_at / updated_at | TIMESTAMPTZ |                                               |
| deleted_at              | TIMESTAMPTZ | NULL — soft delete                            |

#### `profiles` (1:1 with users)

| Column                  | Type          | Notes                                 |
| ----------------------- | ------------- | ------------------------------------- |
| user_id                 | UUID          | PK + FK → users(id) ON DELETE CASCADE |
| full_name               | VARCHAR(120)  | NOT NULL                              |
| headline                | VARCHAR(160)  | freelancer only                       |
| bio                     | TEXT          |                                       |
| country                 | CHAR(2)       | ISO 3166-1 alpha-2                    |
| hourly_rate             | NUMERIC(10,2) | freelancer only, ≥ 0                  |
| company_name            | VARCHAR(120)  | client only                           |
| avatar_url              | TEXT          |                                       |
| created_at / updated_at | TIMESTAMPTZ   |                                       |

#### `categories` (self-referencing)

| Column    | Type        | Notes                              |
| --------- | ----------- | ---------------------------------- |
| id        | SERIAL      | PK                                 |
| parent_id | INT         | FK → categories(id), NULL for root |
| name      | VARCHAR(80) | NOT NULL                           |
| slug      | VARCHAR(80) | UNIQUE, NOT NULL                   |
| is_active | BOOLEAN     | DEFAULT true                       |

> Constraint: max depth of 2 (root → child), enforced in service layer. Deleting a category with children or jobs returns `409`.

#### `skills`

| Column | Type        | Notes            |
| ------ | ----------- | ---------------- |
| id     | SERIAL      | PK               |
| name   | VARCHAR(60) | UNIQUE, NOT NULL |
| slug   | VARCHAR(60) | UNIQUE, NOT NULL |

#### `freelancer_skills` (M:N join, composite PK)

| Column           | Type                | Notes                             |
| ---------------- | ------------------- | --------------------------------- |
| user_id          | UUID                | FK → users(id) ON DELETE CASCADE  |
| skill_id         | INT                 | FK → skills(id) ON DELETE CASCADE |
| years_experience | SMALLINT            | 0–50                              |
| PK               | (user_id, skill_id) |                                   |

#### `jobs`

| Column                               | Type          | Notes                                                                       |
| ------------------------------------ | ------------- | --------------------------------------------------------------------------- |
| id                                   | UUID          | PK                                                                          |
| client_id                            | UUID          | FK → users(id), NOT NULL                                                    |
| category_id                          | INT           | FK → categories(id), NOT NULL                                               |
| title                                | VARCHAR(140)  | NOT NULL                                                                    |
| description                          | TEXT          | NOT NULL, min 50 chars                                                      |
| budget_type                          | budget_type   | ENUM(`fixed`,`hourly`)                                                      |
| budget_min / budget_max              | NUMERIC(12,2) | CHECK budget_max ≥ budget_min                                               |
| experience_level                     | exp_level     | ENUM(`entry`,`intermediate`,`expert`)                                       |
| status                               | job_status    | ENUM(`draft`,`open`,`in_progress`,`completed`,`cancelled`), DEFAULT `draft` |
| deadline                             | DATE          | must be future on create                                                    |
| proposals_count                      | INT           | denormalized counter, DEFAULT 0                                             |
| created_at / updated_at / deleted_at | TIMESTAMPTZ   |                                                                             |

Indexes: `(status, created_at DESC)`, `(category_id)`, `(client_id)`, GIN full-text on `title || description`.

#### `job_skills` (M:N join)

`(job_id UUID, skill_id INT)` — composite PK, both FKs cascade.

#### `proposals`

| Column                  | Type            | Notes                                                                |
| ----------------------- | --------------- | -------------------------------------------------------------------- |
| id                      | UUID            | PK                                                                   |
| job_id                  | UUID            | FK → jobs(id) ON DELETE CASCADE                                      |
| freelancer_id           | UUID            | FK → users(id)                                                       |
| cover_letter            | TEXT            | NOT NULL, 100–5000 chars                                             |
| bid_amount              | NUMERIC(12,2)   | > 0                                                                  |
| estimated_days          | SMALLINT        | 1–365                                                                |
| status                  | proposal_status | ENUM(`pending`,`withdrawn`,`accepted`,`rejected`), DEFAULT `pending` |
| created_at / updated_at | TIMESTAMPTZ     |                                                                      |

Constraint: `UNIQUE (job_id, freelancer_id)` — one proposal per freelancer per job.

#### `contracts`

| Column                    | Type            | Notes                                                    |
| ------------------------- | --------------- | -------------------------------------------------------- |
| id                        | UUID            | PK                                                       |
| job_id                    | UUID            | UNIQUE, FK → jobs(id)                                    |
| proposal_id               | UUID            | UNIQUE, FK → proposals(id)                               |
| client_id / freelancer_id | UUID            | FK → users(id)                                           |
| agreed_amount             | NUMERIC(12,2)   | NOT NULL                                                 |
| status                    | contract_status | ENUM(`active`,`completed`,`cancelled`), DEFAULT `active` |
| started_at / ended_at     | TIMESTAMPTZ     |                                                          |

#### `milestones`

| Column      | Type             | Notes                                             |
| ----------- | ---------------- | ------------------------------------------------- |
| id          | UUID             | PK                                                |
| contract_id | UUID             | FK → contracts(id) ON DELETE CASCADE              |
| title       | VARCHAR(140)     | NOT NULL                                          |
| amount      | NUMERIC(12,2)    | > 0                                               |
| due_date    | DATE             |                                                   |
| status      | milestone_status | ENUM(`pending`,`submitted`,`approved`,`rejected`) |
| sequence    | SMALLINT         | ordering within contract                          |

Business rule: `SUM(milestones.amount) = contracts.agreed_amount` — validated in a transaction.

#### `payments` (simulated)

| Column       | Type           | Notes                                                                       |
| ------------ | -------------- | --------------------------------------------------------------------------- |
| id           | UUID           | PK                                                                          |
| contract_id  | UUID           | FK → contracts(id)                                                          |
| milestone_id | UUID           | FK → milestones(id), nullable                                               |
| amount       | NUMERIC(12,2)  | gross amount, escrowed at milestone creation                                |
| platform_fee | NUMERIC(12,2)  | NOT NULL, DEFAULT 0 — `amount * commission_rate`, computed at release time  |
| net_amount   | NUMERIC(12,2)  | NOT NULL, DEFAULT 0 — `amount - platform_fee`, what the freelancer receives |
| status       | payment_status | ENUM(`escrowed`,`released`,`refunded`)                                      |
| created_at   | TIMESTAMPTZ    |                                                                             |

`CHECK (net_amount = amount - platform_fee)`. Both fee columns stay `0` while `status = escrowed` and are only populated when a payment transitions to `released` (a refund leaves them at `0` too, since no commission is taken on money that never left escrow). The rate itself is not stored per-payment — `platform_fee` is computed at release time from whatever `PLATFORM_COMMISSION_RATE` config value is active, so the ledger stays historically accurate even if the platform's rate changes later.

#### `reviews`

| Column                    | Type        | Notes              |
| ------------------------- | ----------- | ------------------ |
| id                        | UUID        | PK                 |
| contract_id               | UUID        | FK → contracts(id) |
| reviewer_id / reviewee_id | UUID        | FK → users(id)     |
| rating                    | SMALLINT    | CHECK 1–5          |
| comment                   | TEXT        | max 1000 chars     |
| created_at                | TIMESTAMPTZ |                    |

Constraint: `UNIQUE (contract_id, reviewer_id)`; `CHECK (reviewer_id <> reviewee_id)`. Only creatable when contract status is `completed`.

#### `refresh_tokens`

| Column          | Type        | Notes                            |
| --------------- | ----------- | -------------------------------- |
| id              | UUID        | PK                               |
| user_id         | UUID        | FK → users(id) ON DELETE CASCADE |
| token_hash      | TEXT        | SHA-256 of the token, UNIQUE     |
| expires_at      | TIMESTAMPTZ | NOT NULL                         |
| revoked_at      | TIMESTAMPTZ | NULL                             |
| user_agent / ip | TEXT        | audit                            |

---

## 4. Authentication

### 4.1 Token strategy

- **Access token:** JWT, HS256, TTL **15 minutes**, sent as `Authorization: Bearer <token>`.
- **Refresh token:** opaque random 256-bit string, TTL **7 days**, stored hashed in `refresh_tokens`.
- **Rotation:** every `/auth/refresh` call revokes the presented token and issues a new pair. Reuse of a revoked token revokes the **entire token family** for that user (reuse detection).

### 4.2 Access token claims

```json
{
  "sub": "9f1c...",
  "role": "freelancer",
  "email": "dev@example.com",
  "jti": "b21e...",
  "iss": "freelivo",
  "iat": 1735689600,
  "exp": 1735690500
}
```

### 4.3 Middleware chain

```
RequestID → Logger → Recovery → CORS → RateLimit → AuthRequired → RoleGuard → OwnershipGuard → Handler
```

`AuthRequired` validates signature, expiry, and issuer, then loads `user_id`/`role` into the Gin context. It rejects tokens belonging to `suspended` or soft-deleted users.

---

## 5. Authorization (RBAC)

Two layers:

1. **Role guard** — coarse, route-level. `RequireRole("client")`.
2. **Ownership guard** — fine, resource-level. A client may only edit _their own_ job; a freelancer may only withdraw _their own_ proposal.

### 5.1 Permission matrix

| Action                              |  admin   |      client       |       freelancer       |
| ----------------------------------- | :------: | :---------------: | :--------------------: |
| Register / login                    |    —     |        ✅         |           ✅           |
| View own profile                    |    ✅    |        ✅         |           ✅           |
| Edit own profile                    |    ✅    |        ✅         |           ✅           |
| Manage own skills                   |    ❌    |        ❌         |           ✅           |
| CRUD categories & skills            |    ✅    |        ❌         |           ❌           |
| List / view jobs                    |    ✅    |        ✅         |           ✅           |
| Create job                          |    ❌    |        ✅         |           ❌           |
| Edit / delete own job               | ✅ (any) |     ✅ (own)      |           ❌           |
| Submit proposal                     |    ❌    |        ❌         |           ✅           |
| View proposals on a job             |    ✅    |   ✅ (own job)    | ✅ (own proposal only) |
| Withdraw proposal                   |    ❌    |        ❌         |        ✅ (own)        |
| Accept / reject proposal            |    ❌    |   ✅ (own job)    |           ❌           |
| Create milestones                   |    ❌    | ✅ (own contract) |           ❌           |
| Submit milestone delivery           |    ❌    |        ❌         |   ✅ (own contract)    |
| Approve milestone / release payment |    ❌    | ✅ (own contract) |           ❌           |
| Leave review                        |    ❌    |        ✅         |           ✅           |
| Delete any review                   |    ✅    |        ❌         |           ❌           |
| Suspend / reactivate user           |    ✅    |        ❌         |           ❌           |
| Promote user to admin               |    ✅    |        ❌         |           ❌           |

Authorization failures return `403`; missing/invalid tokens return `401`. A resource the caller may not even know exists (another user's `draft` job) returns `404`, not `403`.

---

## 6. API Specification

Base path: `/api/v1`. All bodies are JSON.

### 6.1 Auth

| Method | Path               | Auth   | Description                           |
| ------ | ------------------ | ------ | ------------------------------------- |
| POST   | `/auth/register`   | public | Create user + profile, role in body   |
| POST   | `/auth/login`      | public | Returns access + refresh pair         |
| POST   | `/auth/refresh`    | public | Rotate token pair                     |
| POST   | `/auth/logout`     | user   | Revoke current refresh token          |
| POST   | `/auth/logout-all` | user   | Revoke all sessions                   |
| GET    | `/auth/me`         | user   | Current user + profile                |
| PATCH  | `/auth/password`   | user   | Change password, revokes all sessions |

### 6.2 Users & profiles

| Method | Path                                         | Auth                          |
| ------ | -------------------------------------------- | ----------------------------- |
| GET    | `/users/:id`                                 | public — public profile view  |
| PATCH  | `/users/me/profile`                          | owner                         |
| GET    | `/users?role=freelancer&skill=go&country=TR` | public — searchable directory |
| PATCH  | `/admin/users/:id/status`                    | admin                         |
| PATCH  | `/admin/users/:id/role`                      | admin                         |
| DELETE | `/users/me`                                  | owner — soft delete           |

### 6.3 Skills (freelancer's own set)

| Method | Path                        | Auth                          |
| ------ | --------------------------- | ----------------------------- |
| GET    | `/users/:id/skills`         | public                        |
| PUT    | `/users/me/skills`          | freelancer — replace full set |
| DELETE | `/users/me/skills/:skillId` | freelancer                    |

### 6.4 Taxonomy (admin-managed)

| Method                | Path                    | Auth   |
| --------------------- | ----------------------- | ------ |
| GET                   | `/categories?tree=true` | public |
| POST / PATCH / DELETE | `/categories/:id`       | admin  |
| GET                   | `/skills?q=go`          | public |
| POST / PATCH / DELETE | `/skills/:id`           | admin  |

### 6.5 Jobs

| Method | Path                | Auth                                |
| ------ | ------------------- | ----------------------------------- |
| GET    | `/jobs`             | public — filter, sort, paginate     |
| GET    | `/jobs/:id`         | public (drafts: owner/admin only)   |
| POST   | `/jobs`             | client                              |
| PATCH  | `/jobs/:id`         | owner client / admin                |
| DELETE | `/jobs/:id`         | owner client / admin — soft delete  |
| POST   | `/jobs/:id/publish` | owner client — `draft` → `open`     |
| POST   | `/jobs/:id/close`   | owner client — `open` → `cancelled` |
| GET    | `/jobs/me`          | client — own jobs incl. drafts      |

**Query params for `GET /jobs`:** `q`, `category_id`, `skill_id[]`, `budget_type`, `budget_min`, `budget_max`, `experience_level`, `status`, `sort` (`newest`|`budget_desc`|`proposals_asc`), `page`, `per_page` (default 20, max 100).

### 6.6 Proposals

| Method | Path                      | Auth                          |
| ------ | ------------------------- | ----------------------------- |
| POST   | `/jobs/:id/proposals`     | freelancer                    |
| GET    | `/jobs/:id/proposals`     | job owner / admin             |
| GET    | `/proposals/me`           | freelancer                    |
| GET    | `/proposals/:id`          | author / job owner / admin    |
| PATCH  | `/proposals/:id`          | author — only while `pending` |
| POST   | `/proposals/:id/withdraw` | author                        |
| POST   | `/proposals/:id/accept`   | job owner — creates contract  |
| POST   | `/proposals/:id/reject`   | job owner                     |

### 6.7 Contracts, milestones, payments

| Method | Path                        | Auth                                   |
| ------ | --------------------------- | -------------------------------------- |
| GET    | `/contracts`                | party to contract — own only           |
| GET    | `/contracts/:id`            | party / admin                          |
| POST   | `/contracts/:id/milestones` | client party                           |
| GET    | `/contracts/:id/milestones` | party                                  |
| POST   | `/milestones/:id/submit`    | freelancer party                       |
| POST   | `/milestones/:id/approve`   | client party — releases payment        |
| POST   | `/milestones/:id/reject`    | client party                           |
| POST   | `/contracts/:id/complete`   | client party — all milestones approved |
| POST   | `/contracts/:id/cancel`     | either party — refunds escrow          |
| GET    | `/contracts/:id/payments`   | party                                  |

### 6.8 Reviews

| Method | Path                     | Auth                        |
| ------ | ------------------------ | --------------------------- |
| POST   | `/contracts/:id/reviews` | party, contract `completed` |
| GET    | `/users/:id/reviews`     | public                      |
| DELETE | `/reviews/:id`           | admin                       |

---

## 7. Business Rules & State Machines

### 7.1 Job lifecycle

```
draft ──publish──> open ──accept proposal──> in_progress ──complete──> completed
  │                  │                            │
  └────delete────┐   └────close────> cancelled <──┘ (cancel contract)
```

- A job can only be edited while `draft` or `open` with **zero** proposals; after the first proposal, only `description` and `deadline` may change.
- Publishing requires ≥ 1 linked skill and a non-null category.

### 7.2 Proposal rules

- Only on jobs with status `open`.
- Freelancer cannot bid on their own job (impossible by role, but assert anyway).
- `bid_amount` must fall within `[budget_min * 0.5, budget_max * 2]`.
- Accepting a proposal runs in **one transaction**: set proposal `accepted`, set all sibling proposals `rejected`, set job `in_progress`, insert contract.
- **Visibility:** `jobs.proposals_count` is public and included on every job payload (§3.2), so anyone can see how much competition a job has. The proposals themselves are never listed to competing freelancers — `GET /jobs/:id/proposals` (§6.6) is restricted to the job's owning client and admins; a freelancer can only ever see their own proposal via `GET /proposals/me` or `GET /proposals/:id`.

### 7.3 Milestone rules

- Milestone amounts must sum exactly to `agreed_amount`.
- Flow: `pending` → (freelancer) `submitted` → (client) `approved` | `rejected` → back to `pending` on rejection.
- Creating a milestone inserts a matching `payments` row with `status = escrowed`, `amount = milestone.amount`, `platform_fee = 0`, `net_amount = 0` — this simulates funds being held for that milestone.
- Approving a milestone updates that payment row: `status → released`, `platform_fee = amount * PLATFORM_COMMISSION_RATE`, `net_amount = amount - platform_fee`. The freelancer's payable total is the sum of `net_amount` across their released payments, not the raw milestone amount.
- Cancelling a contract with any `escrowed` payments sets those rows to `status = refunded`; `platform_fee` and `net_amount` remain `0` since no commission applies to a refund.

### 7.4 Review rules

- Only after `contract.status = completed`.
- One review per party per contract; 30-day window after completion.
- A user's aggregate rating is computed on read: `AVG(rating)` + count, cached in profile via trigger or recomputed query.

---

## 8. API Conventions

### 8.1 Success envelope

```json
{
  "data": {},
  "meta": { "page": 1, "per_page": 20, "total": 143, "total_pages": 8 }
}
```

### 8.2 Error envelope

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body is invalid",
    "details": [{ "field": "bid_amount", "issue": "must be greater than 0" }],
    "request_id": "01HQ8..."
  }
}
```

### 8.3 Error codes

| HTTP | Code                                                         |
| ---- | ------------------------------------------------------------ |
| 400  | `VALIDATION_ERROR`, `MALFORMED_JSON`                         |
| 401  | `UNAUTHENTICATED`, `TOKEN_EXPIRED`, `TOKEN_INVALID`          |
| 403  | `FORBIDDEN`, `ACCOUNT_SUSPENDED`                             |
| 404  | `NOT_FOUND`                                                  |
| 409  | `CONFLICT`, `DUPLICATE_RESOURCE`, `INVALID_STATE_TRANSITION` |
| 422  | `BUSINESS_RULE_VIOLATION`                                    |
| 429  | `RATE_LIMITED`                                               |
| 500  | `INTERNAL_ERROR`                                             |

### 8.4 Other conventions

- Pagination: `?page=1&per_page=20`, offset-based (keyset pagination is a stretch goal).
- Timestamps: RFC 3339 UTC.
- IDs: UUID v4 in responses, never sequential integers for user-facing resources.
- `X-Request-ID` echoed on every response.

---

## 9. Non-Functional Requirements

| Area          | Requirement                                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Passwords     | bcrypt cost 12; never returned in any response                                                                                                                       |
| Rate limiting | 5 req/min on `/auth/login` and `/auth/register` per IP; 100 req/min globally per token                                                                               |
| Validation    | All input validated at the handler boundary (`go-playground/validator`)                                                                                              |
| SQL injection | Parameterized queries only — no string concatenation                                                                                                                 |
| Transactions  | Multi-table writes (proposal accept, milestone create, contract complete) wrapped in `BEGIN/COMMIT`                                                                  |
| N+1           | List endpoints must not issue per-row queries; use JOINs or batched `IN` lookups                                                                                     |
| Logging       | Structured JSON (`log/slog`), request ID on every line, no secrets or tokens logged                                                                                  |
| Config        | Environment variables only, `.env.example` committed, secrets never in git. Includes `PLATFORM_COMMISSION_RATE` (decimal, e.g. `0.10` for 10%), read once at startup |
| Migrations    | Versioned up/down files via `golang-migrate`                                                                                                                         |
| Testing       | ≥ 70% coverage on service layer; integration tests against a real Postgres in Docker (`testcontainers-go` or a CI service container)                                 |
| Docs          | OpenAPI 3 spec, served at `/swagger`; Postman collection in `/docs`                                                                                                  |
| Health        | `GET /health` (liveness) and `GET /ready` (DB ping)                                                                                                                  |

---

## 10. Suggested Project Structure

Package-by-feature (domain-driven): each business capability owns a single folder containing its own handler, service, repository, and models. Nothing about jobs lives outside `internal/job`; nothing about proposals lives outside `internal/proposal`. Only true cross-cutting concerns (middleware, DB connection, JWT utilities) sit in shared packages.

```
freelivo/
├── cmd/api/main.go              # wiring: config, db pool, router, graceful shutdown
├── internal/
│   ├── auth/                    # register, login, refresh rotation, logout, password change
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go        # users + refresh_tokens SQL
│   │   ├── model.go
│   │   ├── dto.go
│   │   ├── routes.go
│   │   └── auth_test.go
│   ├── user/                    # profiles, public directory, freelancer skills, admin user mgmt
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go        # profiles + freelancer_skills SQL
│   │   ├── model.go
│   │   ├── dto.go
│   │   └── routes.go
│   ├── taxonomy/                # categories (self-referencing) + skills catalog
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── model.go
│   │   └── routes.go
│   ├── job/                     # jobs + job_skills
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── model.go
│   │   ├── dto.go
│   │   └── routes.go
│   ├── proposal/                # proposals, incl. the accept-proposal transaction
│   │   ├── handler.go
│   │   ├── service.go           # calls into job + contract repos within one tx
│   │   ├── repository.go
│   │   ├── model.go
│   │   ├── dto.go
│   │   └── routes.go
│   ├── contract/                # contracts, milestones, simulated payments
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── model.go
│   │   ├── dto.go
│   │   └── routes.go
│   ├── review/                  # reviews + aggregate rating lookup
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   ├── model.go
│   │   └── routes.go
│   ├── middleware/               # auth guard, rbac, rate limit, logger, recovery, request id
│   ├── platform/                 # shared infra, imported by every feature — no feature imports another
│   │   ├── config/                # env loading
│   │   ├── database/              # pgx pool, tx helper
│   │   ├── jwtutil/                # sign / parse / claims
│   │   ├── hash/                  # bcrypt wrapper
│   │   ├── validator/             # request binding + validation helpers
│   │   ├── response/              # success/error envelope (§8)
│   │   └── pagination/            # offset pagination helper
│   └── server/                    # router.go — imports each feature's routes.go and mounts it
├── migrations/                    # 000001_create_users.up.sql / .down.sql
├── docs/                          # openapi.yaml, postman collection, ERD
├── tests/                         # cross-feature integration tests
├── docker-compose.yml
├── Makefile
└── .env.example
```

**Rules that keep this from rotting back into layered code:**

- **A feature owns its table(s).** `job/repository.go` is the only place that writes SQL against `jobs` and `job_skills`. If another feature needs job data, it calls `job.Service`, not `job.Repository` and never queries the table directly.
- **No feature imports another feature's internals.** Cross-feature calls go through the other feature's exported `Service` interface (e.g. `proposal.Service` depends on `job.Service` and `contract.Service`, injected in `main.go`). This is what keeps `internal/job` importable and testable on its own.
- **Within a feature, still separate concerns by file**, not by folder: `handler.go` binds/validates and calls `service.go`; `service.go` holds business rules and owns transaction boundaries; `repository.go` is the only file with SQL. This gives you the same testability as layered structure, just scoped to one feature instead of spread across the whole app.
- **Multi-feature transactions** (e.g. accepting a proposal touches `proposals`, `jobs`, and `contracts`) live in the service of the feature that initiates the action — here, `proposal.Service.Accept` opens the transaction and calls repository methods on `job` and `contract` that accept a `pgx.Tx`, so the DB round-trip stays atomic without collapsing the feature boundary.
- **`internal/server/router.go`** is the one file allowed to know about every feature; it wires middleware and calls `RegisterRoutes(rg *gin.RouterGroup)` from each feature package.

**Library choices:** `gin-gonic/gin`, `jackc/pgx/v5` (raw SQL with `sqlc` or `sqlx` — avoid an ORM so the SQL practice is real), `golang-jwt/jwt/v5`, `golang-migrate/migrate`, `go-playground/validator/v10`, `golang.org/x/crypto/bcrypt`, `google/uuid`.

---

## 11. Milestones

| Phase                        | Scope                                                                                                                 | Exit criteria                                                           |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **0 — Setup**                | Repo, Docker Compose (API + Postgres), config loading, `/health`, migration tooling, Makefile                         | `make up` boots the stack; migrations run clean                         |
| **1 — Auth**                 | users + profiles + refresh_tokens tables, register, login, refresh with rotation, logout, `/auth/me`, auth middleware | Full token lifecycle passes integration tests, incl. reuse detection    |
| **2 — RBAC & taxonomy**      | Role guard, ownership guard, categories (self-referencing) and skills CRUD, admin user management                     | Permission matrix in §5.1 verified by a table-driven test               |
| **3 — Jobs & proposals**     | Jobs CRUD, filtering/search/pagination, job_skills M:N, proposals with the accept transaction                         | Accepting a proposal atomically rejects siblings and creates a contract |
| **4 — Contracts & payments** | Contracts, milestones, milestone state machine, simulated payments                                                    | Milestone sum constraint enforced; approval releases payment            |
| **5 — Reviews & polish**     | Reviews with aggregate ratings, OpenAPI docs, rate limiting, structured logging, coverage pass                        | Swagger UI reflects all endpoints; coverage ≥ 70%                       |

---

## 12. Stretch Goals

Once the core is done, each of these adds a distinct new skill:

- **Notifications table** with an outbox pattern and a background worker.
- **Full-text search** on jobs using Postgres `tsvector` + GIN, with ranking.
- **Keyset pagination** on the jobs feed to compare against offset pagination.
- **Saved jobs / bookmarks** — another M:N join to practice.
- **Redis** for token blocklist and rate limiting.
- **Audit log** table capturing every admin action.
- **CI pipeline**: lint (`golangci-lint`), test, build, push image.

---

## 13. Resolved Decisions

| #   | Question                                                         | Decision                                                                                                                                                                                                                                                                                | Reflected in                           |
| --- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| 1   | Can a freelancer switch to `client` later, or is role permanent? | **Permanent.** `users.role` is set once at registration and never changes via a self-service endpoint. Only an admin can change it (`PATCH /admin/users/:id/role`, §6.2), and that stays an operational override, not a user-facing feature.                                            | §3.2 `users`, §6.2                     |
| 2   | Are hourly contracts tracked with time logs, or milestones only? | **Milestones only for v1.** No time-tracking table. `budget_type = hourly` on a job only affects how the client and freelancer _negotiate_ the bid — once a contract exists, delivery and payment are always milestone-based. Time logging is a stretch-goal candidate if needed later. | §3.2 `jobs`, §3.2 `milestones`, §7.3   |
| 3   | Does the platform take a commission?                             | **Yes.** `payments` gains `platform_fee` and `net_amount` columns, computed at release time from a configurable `PLATFORM_COMMISSION_RATE`. Escrowed (not-yet-released) and refunded payments carry `0` in both fields, since commission is only earned on money actually paid out.     | §3.2 `payments`, §7.3, §9 (NFR config) |
| 4   | Are proposals visible to competing freelancers?                  | **Count only.** `jobs.proposals_count` is public on every job payload; the actual list of proposals is restricted to the job's owning client and admins. A freelancer sees only their own proposal, never a competitor's.                                                               | §3.2 `jobs`, §6.6, §7.2                |
