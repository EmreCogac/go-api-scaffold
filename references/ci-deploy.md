# CI, Docker and deploy reference

Use this when the user wants a production-ready pipeline. For a local-only skeleton, SKILL.md is enough.

## CI (`.github/workflows/ci.yml`)
Jobs:
- `go vet ./...`
- `go test -race -p 2 -count=1 ./...` (testcontainers need Docker; the hosted runner has it)
- golangci-lint (pin the version)
- `govulncheck ./...` (pin the version)
- **gitleaks** over full history (`fetch-depth: 0`). Keep a `.gitleaksignore` with per-commit, per-file, per-line fingerprints for test fixtures, not path allowlists.
- hygiene: `go mod verify`, `go mod tidy` then `git diff --exit-code go.mod go.sum`, and `test -z "$(gofmt -l .)"`
- Docker build matrix, build-only (no push) for api, worker, migrate

Hardening:
- Pin every action to a **commit SHA** (comment the tag next to it).
- `permissions: contents: read` at the top. CI uses no secrets.
- `concurrency` with `cancel-in-progress: true`.

## Dockerfile
- Multi-stage. Builder uses `--platform=$BUILDPLATFORM` and cross-compiles. Use BuildKit cache mounts for the module and build caches, and run `go mod verify`.
- `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=$GIT_SHA"`.
- One build stage, one `--target` per binary (`api`, `worker`, `migrate`), each on `gcr.io/distroless/static-debian12:nonroot` with `USER 65532`.
- `import _ "time/tzdata"` because distroless has no zoneinfo.
- Distroless has no shell or curl, so give the binary a `healthcheck` subcommand (`/api healthcheck ready` calls `/readyz` on localhost and exits 0 or 1) and use it in `HEALTHCHECK` and in compose.
- OCI labels (`org.opencontainers.image.source`, `revision`).
- `.dockerignore` excludes `.git`, `.env*`, docs, tests data you don't need.

## Compose
Dev (`docker-compose.yml`): only dependencies (Postgres, object storage). The app runs locally with `make up && make migrate && make run`. Publish ports on `127.0.0.1` only.

Prod (`deploy/docker-compose.prod.yml`):
- Fixed project name (`name:`), so a different directory name can't create a second stack.
- API/worker: `read_only: true`, `tmpfs: [/tmp]`, `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`, log rotation (`json-file` with `max-size`/`max-file`), `stop_grace_period`.
- Postgres: **no published ports**, healthcheck, named volume. The admin password is **not** passed to app containers (the app gets only its own role).
- `${VAR:?message}` for every required variable so a missing value fails fast.
- `TRUSTED_PROXIES` set to the reverse proxy's fixed IP on an internal network.
- Reverse proxy (Caddy): automatic Let's Encrypt, JSON logs, `-Server` header, `/readyz` returns 404 publicly (only the internal network and healthchecks see it), upstream health checks.

## Deploy (`deploy/deploy.sh`)
Order: lock -> pull images -> backup -> migrate (separate `migrate` image, fail the deploy if it fails) -> `up -d --no-deps --force-recreate api worker` -> wait for readiness (and check the worker is not restart-looping) -> record the SHA.
- Lock with atomic `mkdir` and clean it up with `trap`.
- Never recreate Postgres in a deploy (`--no-recreate`/`--no-deps`).
- On failed readiness roll back to `.deployed_sha`. Distinct exit codes (backup failed, migrate failed, rolled back, manual intervention needed) so the workflow can print a message for each.
- Append to `.deploy_history`.
- `lib.sh`: require the server `.env` to be mode 600; shared helpers.

### GitHub Actions deploy workflow
- Trigger with `workflow_run` after CI succeeds, `main` and same-repo only.
- Build and push images to GHCR tagged `:sha` and `:latest`.
- `production` environment holds secrets. SSH with a key and a **pinned `known_hosts`** (no trust on first use); remove the key under `if: always()`.
- Registry login on the server with the short-lived `GITHUB_TOKEN` over stdin, not a PAT.
- Validate workflow inputs with a regex. `cancel-in-progress: false` for deploys.
- After deploy, an external health check against the public URL.

## Backup (`deploy/backup.sh`)
- `pg_dump -Fc` plus `pg_dumpall --globals-only`.
- Run as the **admin** role. With RLS forced, the app role would silently produce an empty dump.
- Verify: `pg_restore --list` works and the expected tables contain data.
- Write to `*.partial`, then `mv` after verification. `umask 077`, files mode 600. Keep the last 14.
- Rehearse a restore at least once and document it in `docs/SERVER_SETUP.md` (first server setup, firewall, deploy user and SSH, restore rehearsal, disaster recovery, things not to do).

## Deploy invariants as a Go test
`deploy/deploy_test.go` parses the compose file, Caddyfile and scripts and fails when:
- a service other than the proxy publishes ports,
- `read_only`, `cap_drop: ALL` or `no-new-privileges` is missing,
- `/readyz` is reachable publicly,
- a script contains `down -v` or `docker volume rm`,
- a workflow action isn't pinned to a SHA, or CI references secrets.
Cheap to write and it keeps hardening from being quietly removed.

## Makefile (typical)
`run`, `build`, `test`, `vet`, `lint`, `vuln`, `check` (vet + lint + test), `migrate` (`go run ./cmd/migrate up`), `migrate-down`, `up`/`down` (dev dependencies), `images`.

## Handoff files for AI-assisted work
- `AGENTS.md`: single source of truth (stack, layering rules, forbidden things, commands, test rules, out-of-scope list, "open decisions: do not assume"). `CLAUDE.md` contains only `@AGENTS.md`; `.github/copilot-instructions.md` points to it.
- `docs/DECISIONS.md`: append-only decision log (new decisions are added, old ones are never deleted; mark superseded ones).
- `docs/PROGRESS.md` (optional): task checklist by phase; one commit per task (`T-12: ...`); any new dependency is justified there.
