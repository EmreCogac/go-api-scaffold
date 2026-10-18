# go-api-scaffold

A Claude skill that turns a short project description into a working, secure-by-default Go REST API skeleton. It can also review an existing Go API against a security checklist and harden its CI, Docker and deploy setup.

## What it does

- **Scaffold mode**: generates a layered (clean architecture) Go API with env-based config, migrations, Docker setup, auth and security middleware already wired in.
- **Review mode**: reads an existing Go API and reports findings by severity (Critical / High / Medium / Low), with file and line, risk and suggested fix. It does not change code until you approve.
- **Production mode** (on request): CI pipeline, hardened Dockerfile and prod compose, deploy and backup scripts, and tests that keep the hardening from being removed.

It adapts auth and security to your client type: **mobile**, **web**, or **both**.

## When it triggers

Ask for things like:

- "Create a new API for my app"
- "Set up a backend in Go"
- "Go project skeleton"
- "Review / audit my Go API for security"
- "Harden my deploy"

## Questions it may ask

Only what is missing, in one message (at most 4 questions):

1. Client type: mobile, web, or both?
2. Auth: none, email + password (default), or Google/OIDC sign-in?
3. Project name and Go module path
4. Special requirements (file uploads, background jobs, websockets, multi-tenancy, production deploy, ...)

If you don't answer, it proceeds with defaults and lists every assumption at the end.

## Default stack

| Concern | Choice |
|---|---|
| Router | Gin |
| ORM | GORM (pgx driver) |
| Database | PostgreSQL 16 |
| Migrations | golang-migrate, embedded SQL, run by `cmd/migrate` (goose also fine) |
| Config | Environment variables only, via viper, validated at startup |
| Validation | go-playground/validator |
| Logging | `log/slog` (JSON) |
| Auth | JWT (golang-jwt/v5) + rotating opaque refresh tokens; bcrypt for passwords |
| Rate limiting | `x/time/rate` or ulule/limiter |
| API docs | Spec-first `api/openapi.yaml` with a conformance test (swag optional) |
| Background jobs | River (Postgres-backed), when needed |
| Tests | testing + testify + testcontainers (real Postgres) |

You can ask for something else (Fiber, sqlc, MySQL, ...). The security rules still apply.

## Generated project layout

```
<project>/
├── cmd/{api,migrate,worker}/main.go
├── internal/
│   ├── domain/            # entities, interfaces, Clock (no framework imports)
│   ├── usecase/           # business rules
│   ├── delivery/http/     # router, handler, middleware, dto
│   ├── infrastructure/    # postgres, jwt, clock, ...
│   ├── config/
│   └── pkg/               # apperror, logger
├── api/openapi.yaml
├── migrations/
├── deploy/                # on request
├── docs/DECISIONS.md
├── .env.example  .gitignore  .dockerignore  .golangci.yml
└── Dockerfile  docker-compose.yml  Makefile  README.md  AGENTS.md
```

Dependency direction: `delivery -> usecase -> domain`, with `infrastructure` implementing the interfaces declared in `domain`.

## Behavior by client type

| | Mobile | Web | Both |
|---|---|---|---|
| Token transport | `Authorization: Bearer` | Access token in memory, refresh token in httpOnly + Secure + SameSite cookie | Separate `/api/v1/mobile/...` and `/api/v1/web/...` auth endpoints |
| Refresh tokens | Rotating, SHA-256 hashed, reuse revokes the family | Same, delivered via cookie | Shared usecase layer |
| CSRF | Not used | Required for cookie-authenticated state-changing endpoints | Web routes only |
| CORS | Not needed | Allow-list from `CORS_ALLOWED_ORIGINS` | Web routes only |
| Rate limit key | User / device (IP when unauthenticated) | IP (with trusted proxies configured) | Per route group |
| Extras | App-version gate (`426`), `GET /v1/config`, cursor pagination | Security headers (HSTS, nosniff, X-Frame-Options, CSP, Referrer-Policy) | |

## Security checklist

Applied to every generated project and used as the baseline in review mode:

1. bcrypt (cost >= 12), 8 to 72 byte passwords, normalized emails
2. Input validation and request body size limits
3. Parameterized queries only
4. Object-level authorization; foreign IDs return the same 404 as nonexistent IDs
5. DTOs instead of binding to models (mass assignment)
6. Strict rate limits and lockout on auth endpoints
7. Generic login errors, dummy hash for unknown accounts
8. No stack traces or SQL errors to clients; one error format with `request_id`
9. No secrets or sensitive data in logs
10. Pinned JWT algorithm, required claims, UUID `sub`, user existence check
11. Hashed, rotated refresh tokens with family reuse detection and cleanup
12. TLS to the DB in production, non-superuser DB role
13. `govulncheck` (`make vuln`) and gitleaks in CI
14. Safe file uploads (bytes-based type check, server-chosen keys, re-encode)
15. Reusable role-check middleware
16. Docs and debug endpoints off in production
17. Constant-time secret comparison
18. Account deletion and data export

## Safety rules

- Only `.env.example` is generated, never a real `.env`, and the skill never reads your real `.env*` files.
- No real-looking secrets, passwords or API keys are written to any file. Generate your own with `openssl rand -base64 48`.
- In production, the app refuses to start on short or placeholder secrets, empty trusted proxies, or dev-only settings.
- The skeleton is verified with `go mod tidy`, `go build ./...` and `go vet ./...` before it is handed over.
- Existing files are never overwritten without asking.

## First run

After scaffolding:

```bash
cp .env.example .env     # then fill in real secrets
make up
make migrate
make run
```

Other Makefile targets: `build`, `test`, `vet`, `lint`, `vuln`, `check`, `migrate-down`, `down`.

## Installation

Place the skill folder (containing `SKILL.md` and `references/`) in one of these locations:

- Personal (all projects): `~/.claude/skills/go-api-scaffold/`
- Project only: `<project>/.claude/skills/go-api-scaffold/`

Claude loads it automatically when your request matches the description in the `SKILL.md` frontmatter. The files in `references/` are read only when a task needs them.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The skill itself: workflow, defaults, layout, security checklist |
| `references/auth.md` | Login methods, JWT, refresh rotation, audit log, account lifecycle, rate limiting |
| `references/authz-and-rls.md` | Object-level authz, Row Level Security, sync, quotas, uploads, logging |
| `references/testing.md` | Real-DB tests, JWT forgery, authz matrix, OpenAPI conformance, lint rules |
| `references/ci-deploy.md` | CI jobs, Docker and prod compose hardening, deploy and backup, handoff files |
| `README.md` | This overview |
