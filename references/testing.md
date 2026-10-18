# Testing reference

## Principles
- Use a **real PostgreSQL** (testcontainers, one container per package, a template database or truncate between tests). No sqlite, no mocked repositories for repository or HTTP tests.
- Connect tests as the **non-superuser application role**, so RLS and grants apply exactly as in production.
- Inject `domain.Clock` and use a fake clock so token expiry and TTL tests don't sleep.
- Run with `go test -race -count=1 ./...`.
- Domain logic that is pure (scheduling, scoring, intervals) gets table tests and, if it also exists on the client, a shared golden file.

## Minimum set for every project
1. Password hashing (if passwords): hash differs per call, verify works, over-72-byte input rejected.
2. JWT forgery table test (below).
3. Refresh rotation, reuse detection and concurrent refresh.
4. One IDOR check per resource (full matrix below if time allows).
5. Config: production refuses weak secrets and placeholders.

## JWT forgery test
Create a control token that must pass (proves the harness works), then require **identical 401 bodies** for each of:
- `alg: none`
- wrong algorithm (HS512 when HS256 is pinned; RS256 signed with the HMAC secret as key)
- wrong secret
- wrong `aud`, wrong `iss`
- expired, `nbf` in the future
- missing `sub`, non-UUID `sub` (for example `"1 OR 1=1; --"`)
- valid token for an unknown user, and for a soft-deleted user
- a refresh token used as an access token
- a valid token passed in the query string (must be ignored)

## Refresh token tests
- Rotation returns a new token and revokes the old one.
- Presenting the old token again revokes the whole family and writes `token_reuse`.
- Two concurrent refreshes with the same token: exactly one succeeds.
- Two concurrent first logins for the same new user create one user.
- The DB stores only the hash (the raw token appears nowhere in the table).

## Authorization matrix
For every resource and every route that takes an ID:
1. Create data as user A. Snapshot A's tables.
2. As user B, call read, update, delete and any action on A's real IDs.
3. Assert the response is **byte-identical** to the response for a random nonexistent UUID (no 403/404 difference).
4. Assert A's data is unchanged and nothing was created for B.
5. **Route guard**: enumerate `router.Routes()`; the test fails if a route is in neither the attack list nor an explicit `scopedRoutes` allow-list (with a reason). New endpoints can't ship untested.

## OpenAPI conformance
With a hand-written `api/openapi.yaml`:
- `openapi_test.go` loads and validates the spec.
- `conformance_test.go` sends real requests through the router and validates every request and response with `kin-openapi/openapi3filter`. A field or status code that isn't in the spec fails the test.

## Other worthwhile tests
- Middleware order (`router_chain_test.go`): recovery first, request ID before access log, rate limit before auth on public routes.
- Logs contain no tokens or emails after a login.
- RLS: user B cannot see A's rows even with a raw query on the scoped connection; every table with `user_id` has forced RLS; the app refuses to start as a superuser.
- Dev-only features: the app refuses to start in production when dev keys are set; `/docs` is absent in production.
- Deploy invariants (see `ci-deploy.md`).

## Lint rules that enforce architecture
In `.golangci.yml` (v2, `default: standard`) enable `errorlint`, `gosec`, `bodyclose`, `noctx`, `forbidigo`. With `forbidigo`, ban `time.Now` outside `internal/infrastructure/clock`, and ban `gin`/`gorm` imports in `internal/domain` and `internal/usecase` (via `depguard` or `importas`). Exclude `gosec` for `_test.go`.
