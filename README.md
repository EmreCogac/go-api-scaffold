# go-api-scaffold

A Claude skill that turns a short project description into a working, secure-by-default Go REST API skeleton, and can also review an existing Go API against a security checklist.

## What it does

- **Scaffold mode**: generates a Go API project (Gin + GORM + PostgreSQL by default) with config, migrations, Docker setup, auth and security middleware already wired in.
- **Review mode**: reads an existing Go API and reports security findings by severity (Critical / High / Medium / Low), with file and line, risk and suggested fix. It does not change code until you approve.

It adapts the auth and security layers to your client type: **mobile**, **web**, or **both**.

## When it triggers

Ask for things like:

- "Create a new API for my app"
- "Set up a backend in Go"
- "Go project skeleton"
- "Review my Go API for security"

## Questions it may ask

Only what is missing, in one message (at most 4 questions):

1. Client type: mobile, web, or both?
2. Is auth needed? (default: yes, email + password, JWT)
3. Project name and Go module path
4. Special requirements (file uploads, websockets, queues, multi-tenancy, ...)

If you don't answer, it proceeds with defaults and lists every assumption at the end.

## Default stack

| Concern | Choice |
|---|---|
| Router | Gin |
| ORM | GORM (pgx driver) |
| Database | PostgreSQL 16 |
| Migrations | golang-migrate |
| Config | `.env` + `config.yaml` via viper (env overrides yaml) |
| Validation | go-playground/validator |
| Logging | `log/slog` (JSON) |
| Auth | JWT (golang-jwt/v5) + bcrypt (cost 12) |
| Rate limiting | ulule/limiter or in-memory token bucket |
| API docs | swaggo/swag |
| Tests | testing + testify |

You can ask for something else (Fiber, sqlc, MySQL, ...). The security rules still apply.

## Generated project layout

```
<project>/
├── cmd/api/main.go
├── internal/
│   ├── config/
│   ├── server/
│   ├── middleware/
│   ├── auth/
│   ├── user/
│   └── platform/
├── migrations/
├── docs/
├── config.yaml
├── .env.example
├── .gitignore
├── .dockerignore
├── .golangci.yml
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── README.md
```

Request flow: handler (HTTP) -> service (business rules) -> repository (DB).

## Behavior by client type

| | Mobile | Web | Both |
|---|---|---|---|
| Token transport | `Authorization: Bearer` | Access token in memory, refresh token in httpOnly + Secure + SameSite cookie | Separate `/api/v1/mobile/...` and `/api/v1/web/...` auth endpoints |
| Refresh tokens | Rotating, stored as SHA-256 hash, reuse detection revokes the family | Same, delivered via cookie | Shared service layer |
| CSRF | Not used | Required for cookie-based state-changing endpoints | Web routes only |
| CORS | Not needed | Allow-list from `CORS_ALLOWED_ORIGINS` | Web routes only |
| Rate limit key | User / device (IP for unauthenticated) | IP (with trusted proxies configured) | Per route group |
| Extras | `devices` table, `X-App-Version` check (`426`), cursor pagination | Security headers (HSTS, nosniff, X-Frame-Options, CSP, Referrer-Policy) | |

## Security checklist

Applied to every generated project and used as the baseline in review mode:

1. bcrypt (cost >= 12), 8 to 72 byte passwords, normalized emails
2. Input validation and request body size limits
3. Parameterized queries only
4. Object-level authorization (IDOR prevention)
5. DTOs instead of binding to models (mass assignment)
6. Strict rate limits and temporary lockout on auth endpoints
7. Generic login errors, constant-time behavior for unknown accounts
8. No stack traces or SQL errors to clients; one error format with `request_id`
9. Request IDs in logs; no secrets or sensitive data in logs
10. Pinned JWT algorithm, claim validation, separate access/refresh secrets
11. TLS to the DB in production, non-superuser DB role
12. Gin release mode in production
13. `govulncheck` via `make vuln`
14. Safe file uploads
15. Reusable role-check middleware
16. Swagger and debug endpoints off in production
17. Constant-time secret comparison

## Safety rules

- Only `.env.example` is generated, never a real `.env`.
- No real-looking secrets, passwords or API keys are written to any file. Generate your own with `openssl rand -base64 48`.
- In production, the app refuses to start if a secret is shorter than 32 characters or still a placeholder.
- The skeleton is verified with `go mod tidy`, `go build ./...` and `go vet ./...` before it is handed over.
- Existing files are never overwritten without asking.

## First run

After scaffolding:

```bash
cp .env.example .env     # then fill in real secrets
make docker-up
make migrate-up
make run
```

Other Makefile targets: `build`, `test`, `lint`, `vuln`, `migrate-down`, `swag`.

## Installation

Place the skill folder (containing `SKILL.md`) in one of these locations:

- Personal (all projects): `~/.claude/skills/go-api-scaffold/SKILL.md`
- Project only: `<project>/.claude/skills/go-api-scaffold/SKILL.md`

Claude loads it automatically when your request matches the description in the `SKILL.md` frontmatter.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The skill itself: instructions Claude follows |
| `README.md` | This overview |
