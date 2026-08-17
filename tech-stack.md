# Freelivo — Tech Stack

Companion to [PRD.md](PRD.md) §9–10. This pins down the exact libraries behind the PRD's high-level choices (Go, Gin, PostgreSQL, JWT, Docker) and records the reasoning for the few decisions the PRD leaves open.

---

## Core

| Concern     | Choice                                           | Why                                                                                                                                    |
| ----------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Language    | Go, current stable release                       | PRD's floor is 1.22; nothing here needs an old toolchain, and newer versions give better `net/http` routing, `slog` maturity, tooling  |
| HTTP router | `gin-gonic/gin`                                  | As specced in PRD §10. Largest ecosystem for the middleware chain in PRD §4.3                                                          |
| DB driver   | `jackc/pgx/v5` (native pool, not `database/sql`) | Native mode gives real Postgres types — `NUMERIC`, `CITEXT`, `TIMESTAMPTZ`, arrays — and batching to satisfy the N+1 rule in PRD §9    |
| Query layer | `sqlc`                                           | Generates typed Go from real `.sql` files — SQL practice stays real, compiler catches schema drift. Configure with the `pgx/v5` engine |
| Migrations  | `golang-migrate`                                 | As specced in PRD §9                                                                                                                   |
| JWT         | `golang-jwt/jwt/v5`                              | As specced in PRD §10                                                                                                                  |
| Validation  | `go-playground/validator/v10`                    | As specced in PRD §9/§10, wired into Gin's binding                                                                                     |
| Passwords   | `golang.org/x/crypto/bcrypt`, cost 12            | As specced in PRD §9                                                                                                                   |
| Money       | `shopspring/decimal` + `pgx-shopspring-decimal`  | `float64` for `NUMERIC(12,2)` will silently break the `net_amount = amount - platform_fee` CHECK (PRD §3.2 `payments`)                 |
| Config      | `caarlos0/env/v11` + `joho/godotenv`             | Struct-tag env loading; matches the env-only rule in PRD §9                                                                            |
| Logging     | stdlib `log/slog`                                | As specced in PRD §9                                                                                                                   |
| UUIDs       | `google/uuid`                                    | As specced in PRD §10                                                                                                                  |

## Testing & ops

| Concern        | Choice                                                                      |
| -------------- | --------------------------------------------------------------------------- |
| Assertions     | `stretchr/testify`                                                          |
| Integration DB | `testcontainers-go` locally, Postgres service container in CI               |
| Mocks          | `uber-go/mock` for cross-feature `Service` interfaces (PRD §10)             |
| Rate limiting  | `golang.org/x/time/rate` in-memory for v1; Redis in stretch goals (PRD §12) |
| API docs       | `swaggo/swag` annotations → Swagger UI at `/swagger` (PRD §9)               |
| Lint           | `golangci-lint` (PRD §12 CI pipeline)                                       |
| Live reload    | `air`                                                                       |
| Container      | Multi-stage build on `distroless/static`                                    |
| CI             | GitHub Actions: lint → test (Postgres service) → build                      |

---
