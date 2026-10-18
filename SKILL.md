---
name: go-api-scaffold
description: Use when scaffolding a new Go REST API from scratch, reviewing or auditing an existing Go API for security, or hardening its CI/Docker/deploy setup. Picks the framework and database, sets up a layered (clean architecture) project, env config, docker-compose, migrations, auth and security layers, and adapts them to the client type (mobile, web, or both). Triggers on requests like "new API", "set up a backend", "Go project skeleton", "API security review", "audit my Go API", "harden deploy".
---

# Go API Scaffold

Goal: turn a short project description into a working Go API skeleton with secure defaults. Don't ask unnecessary questions. Use the defaults below and only ask about whatever is missing from "Before you start" (one message, at most 4 questions).

## Before you start (ask only what is missing)

1. Client type: **mobile**, **web**, or **both**?
2. Auth: none, **email + password** (default), or **Google/OIDC sign-in** (no passwords)?
3. Project name and Go module path (e.g. `github.com/<user>/<project>`)
4. Special requirements? (file uploads, background jobs, websockets, multi-tenancy, production deploy, etc.)

If the user doesn't answer, proceed with defaults and state each assumption in one line at the end.

## Default technology choices

| Concern | Choice |
|---|---|
| Router | Gin |
| ORM | GORM (pgx driver), GORM tags only on infrastructure models |
| Database | PostgreSQL 16 |
| Migrations | golang-migrate, SQL files embedded (`embed`) and run by `cmd/migrate`. Never GORM AutoMigrate. (goose is an equal alternative.) |
| Config | **Environment variables only**, loaded with `viper`, typed and validated at startup. `.env` is for local dev and Docker, the app never reads secrets from yaml. |
| Validation | go-playground/validator |
| Logging | stdlib `log/slog` (JSON) |
| Auth | JWT (golang-jwt/v5) + opaque rotating refresh tokens; bcrypt (cost 12) if passwords |
| Rate limiting | `golang.org/x/time/rate` or `ulule/limiter` |
| API docs | **Spec-first**: hand-written `api/openapi.yaml`, embedded, checked by a conformance test. (swaggo/swag only if the user prefers annotations.) |
| Background jobs | River (Postgres-backed) when needed, no Redis by default |
| Tests | testing + testify + testcontainers (real Postgres) |

If the user prefers something else (Fiber, sqlc, MySQL, ...), use it, but apply the security rules unchanged.

## Project structure (layered / clean architecture)

```
<project>/
├── cmd/
│   ├── api/main.go          # wiring only: manual DI, graceful shutdown
│   ├── migrate/main.go      # up | down | status (embedded SQL)
│   └── worker/main.go       # only if background jobs are needed
├── internal/
│   ├── domain/              # entities, interfaces, Clock. No gin/gorm/viper imports
│   ├── usecase/{auth,user}/ # business rules
│   ├── delivery/http/       # router.go, handler/, middleware/, dto/
│   ├── infrastructure/      # postgres/ (GORM models + repos), jwt/, clock/, ...
│   ├── config/              # env loading + validation
│   └── pkg/                 # apperror/, logger/
├── api/openapi.yaml
├── migrations/              # 00001_init.up.sql / .down.sql, embed.go
├── deploy/                  # only when production deploy is requested
├── docs/DECISIONS.md
├── .env.example  .gitignore  .dockerignore  .golangci.yml
├── Dockerfile  docker-compose.yml  Makefile  README.md  AGENTS.md
```

Layering rules (also write them into AGENTS.md):
- `delivery -> usecase -> domain`; `infrastructure` implements interfaces declared in `domain`.
- Handlers contain no business rules and never touch the DB. GORM tags live only in `infrastructure/postgres`, which maps to domain entities.
- Manual dependency injection in `main.go`; no globals, no side effects in `init()`.
- `context.Context` is the first parameter everywhere; wrap errors with `%w`.
- Time comes from an injected `domain.Clock`; ban `time.Now` outside `infrastructure/clock` with `forbidigo`.
- Every repository method touching user data takes `userID`.

For a tiny throwaway project the user may ask for a flat feature layout (`internal/auth`, `internal/user`); keep the same security rules.

## Files to generate and their rules

- **.env.example** (never a real `.env`): `APP_ENV`, `APP_PORT`, `DATABASE_URL`, `JWT_SECRET`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_ACCESS_TTL=15m`, `JWT_REFRESH_TTL=720h`, `TRUSTED_PROXIES`, `RATE_LIMIT_*`, plus `CORS_ALLOWED_ORIGINS` (web only) and `GOOGLE_CLIENT_IDS` (Google sign-in only). Use placeholders, never real-looking secrets; tell the user to run `openssl rand -base64 48`.
- **Config validation**: in production, **refuse to start** if a secret is shorter than 32 characters or still a placeholder, if `TRUSTED_PROXIES` is empty, or if dev-only settings (dev API keys, auto-create bucket, docs) are enabled. All limits must be > 0.
- **docker-compose.yml**: Postgres with healthcheck and named volume; DB port published on `127.0.0.1` only; inside the compose network the API uses host `db`, not `localhost`.
- **Dockerfile**: multi-stage, `CGO_ENABLED=0`, `distroless/static:nonroot`, non-root user. Details in `references/ci-deploy.md`.
- **Makefile**: `run`, `build`, `test`, `vet`, `lint`, `vuln`, `check` (vet + lint + test), `migrate`, `migrate-down`, `up`, `down`.
- **main.go**: graceful shutdown (SIGINT/SIGTERM, 10 s); `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` on `http.Server`; `/healthz` (liveness) and `/readyz` (DB ping); Gin in `release` mode when `APP_ENV=production`.
- **Middleware order** (global): recovery, request ID, access log (route pattern, no bodies/tokens), secure headers, body limit (1 MB), timeout, IP rate limit. Then per-group: auth, app-version gate, per-user rate limit.
- **Initial migration**: UUID primary keys; `users` (case-insensitive unique email or `(provider, sub)`, `role` default `user`, `deleted_at`, timestamps), `refresh_tokens`, `audit_log`, and `devices` for mobile. Always write `.down.sql`. Column details in `references/auth.md`.
- **Error format**, one shape everywhere: `{"error":{"code","message","fields","request_id"}}`, mapped to statuses in `internal/pkg/apperror` only.
- **.golangci.yml** (v2): `standard` + `errorlint`, `gosec`, `bodyclose`, `noctx`, `forbidigo`.
- **AGENTS.md** with stack, layering rules, forbidden things, commands and an "open decisions" list; `docs/DECISIONS.md` as an append-only decision log.

## Behavior by client type

### Mobile
- Access token (15 min) in the `Authorization: Bearer` header + **rotating refresh token** with family reuse detection. No cookies, no CSRF, no CORS middleware.
- `X-App-Version` gate: below `MIN_APP_VERSION`, or missing/unparseable when a minimum is set, returns `426`. Public `GET /v1/config` exposes the minimum.
- Rate limit per user/device; unauthenticated routes fall back to IP.
- Cursor pagination (`limit`, `cursor`) is required on list endpoints.

### Web
- Access token in memory, refresh token in an httpOnly + Secure + SameSite cookie; CSRF protection on cookie-authenticated state-changing endpoints.
- CORS only for `CORS_ALLOWED_ORIGINS`; never `*` with credentials.
- Security headers: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy: default-src 'none'`.
- Rate limit per IP; configure `SetTrustedProxies`.

### Both
Separate `/api/v1/mobile/...` and `/api/v1/web/...` auth endpoints (different token transport); share the `usecase` layer.

## Security checklist (apply to every project)

1. Passwords: bcrypt cost >= 12, 8 to 72 **bytes**, normalized emails; never log plaintext.
2. Validate all input; limit request body size (default 1 MB).
3. Parameterized queries only (GORM or `?` placeholders); no SQL string concatenation.
4. **Object-level authorization**: every record is checked against the requesting user. Foreign IDs return the **same 404 as nonexistent IDs**.
5. DTOs for create/update; never bind to models (mass assignment).
6. Strict rate limits on login, register, refresh and password reset, plus temporary per-account lockout for passwords.
7. Login errors say "invalid email or password"; unknown emails still run a dummy bcrypt comparison.
8. Never return stack traces or SQL errors; log server-side, return a generic message plus `request_id`.
9. `request_id` in every log line; never log tokens, passwords, bodies, full emails, card or national ID numbers.
10. JWT: pinned algorithm (reject `none`), require `exp`/`iss`/`aud`, validate `sub` as UUID and that the user still exists. Read tokens from the header only.
11. Refresh tokens: opaque, SHA-256 hashed in the DB, rotated in one transaction, reuse revokes the family, expired tokens are cleaned up by a job.
12. Production DB: `sslmode=require` or stronger; the app's DB role is not a superuser (and, with RLS, not `BYPASSRLS`).
13. `make vuln` runs `govulncheck ./...`; CI also runs gitleaks.
14. File uploads: size limit, content type from the bytes, server-generated object keys, private bucket, re-encode images.
15. Role checks in a reusable `RequireRole` middleware; roles come from the DB or a signed claim, never from the request. Default role is least privileged.
16. Docs UI, debug and pprof endpoints are off in production (or behind auth); docs assets are embedded, not loaded from a CDN.
17. Compare secrets and tokens with `crypto/subtle.ConstantTimeCompare`.
18. Provide account deletion and data export for any app that stores personal data.

## Reviewing an existing project (security review mode)

1. Read the project (router, middleware, auth, repository layers).
2. Compare it against the checklist above, item by item.
3. Report findings as **Critical / High / Medium / Low**: `[Severity] file:line - title`, then *Risk*, then *Fix*. Close with the checklist items that passed.
4. Run `govulncheck ./...` if available and include the results.
5. Do not modify code until the user approves; present the report first, then propose fixes.

## Working rules

- If the target directory already contains files, list what would be overwritten and ask first.
- Pin a current stable Go version in `go.mod`, the Dockerfile and CI (same version everywhere).
- Generate the skeleton, then run `go mod tidy`, `go build ./...` and `go vet ./...`. Never hand over code that doesn't build.
- Minimum tests: password hashing (if used), JWT forgery cases, refresh rotation/reuse, one IDOR check, config rejecting weak secrets. See `references/testing.md`.
- One example resource (`user` + `auth`) and one migration are enough; don't guess the user's domain models.
- Never write real secrets, passwords or API keys into files, and never read the user's real `.env*` files; use `.env.example`.
- End with a short summary: chosen stack, assumptions made, and first-run commands (`cp .env.example .env`, `make up`, `make migrate`, `make run`).

## References (read only when relevant)

- `references/auth.md`: login methods, JWT, refresh rotation SQL, audit log, account lifecycle, mobile/web details, rate limiting.
- `references/authz-and-rls.md`: object-level authz, PostgreSQL Row Level Security, sync, quotas, upload flow, logging.
- `references/testing.md`: real-DB tests, JWT forgery, authz matrix with route guard, OpenAPI conformance, lint rules.
- `references/ci-deploy.md`: CI jobs, hardened Dockerfile and prod compose, deploy and backup scripts, deploy invariant tests, handoff files.
