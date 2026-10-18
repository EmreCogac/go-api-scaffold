# Auth reference

Read this when the project has authentication. SKILL.md only keeps the rules that apply to every project.

## Choosing the login method

| Method | Use when | Notes |
|---|---|---|
| Email + password | Default | bcrypt cost >= 12, 8 to 72 **byte** passwords, normalized (trim + lowercase) emails, dummy-hash comparison for unknown emails |
| Google (or other OIDC) | Mobile apps, no passwords wanted | No password table, no reset flow, no timing concern. Key users by the provider `sub`, never by email |

### Google ID token verification
The `idtoken` library does not check everything for you. Verify by hand:
- signature (via the library),
- `iss` is `accounts.google.com` or `https://accounts.google.com`,
- `aud` is in an **allow-list** of your client IDs (web, Android, iOS can differ),
- `exp` not passed,
- `email_verified` is true if you use the email at all,
- `sub` present. Store `(provider, sub)` as the unique identity.

## Access token (JWT)
- HS256 (single service) or RS256/EdDSA (several verifiers). Pin the algorithm in the parser and reject everything else, including `none`.
- Claims: `iss`, `aud`, `sub`, `exp`, `iat`, `nbf`, `jti`. The parser must **require** `exp`, `iss`, `aud`.
- Secret at least 32 bytes. If refresh tokens are opaque (see below), one JWT secret is enough. If you make refresh tokens JWTs, use a different secret.
- TTL 15 minutes.
- In the authenticate middleware, in this order:
  1. read the token **only** from the `Authorization` header (never the query string),
  2. verify signature and claims,
  3. check `sub` is a valid UUID **before** any DB lookup,
  4. load the user and reject if missing or soft-deleted (a valid signature for a deleted user must fail).
- Every rejection returns the same 401 body, whatever the reason.

## Refresh token
- Opaque: 32 bytes from `crypto/rand`, base64url. Stored as a **SHA-256 hash** (`bytea`, `CHECK (octet_length(token_hash) = 32)`). Not bcrypt: the value is high-entropy and looked up by hash.
- TTL 30 days (mobile) or as configured.
- Columns: `id`, `user_id` (FK `ON DELETE CASCADE`), `token_hash` (unique), `family_id`, `device_model`, `device_os`, `expires_at`, `revoked_at`, `created_at`.

### Rotation in one transaction
```
BEGIN;
  UPDATE refresh_tokens SET revoked_at = now()
   WHERE token_hash = $1 AND revoked_at IS NULL AND expires_at > now()
   RETURNING user_id, family_id;
  -- 1 row  -> INSERT the new token (same family_id), COMMIT
  -- 0 rows -> look the hash up:
  --            unknown/expired        -> 401
  --            exists but revoked     -> REUSE: revoke the whole family,
  --                                      write audit row 'token_reuse', 401
COMMIT;
```
The `WHERE revoked_at IS NULL` is what makes two concurrent refreshes with the same token safe: only one gets a row back.

### Logout
- `POST /auth/logout`: revoke the family of the supplied token. An unknown token or someone else's token is a silent no-op (200/204), so it can't be used to probe.
- `POST /auth/logout-all`: revoke every token of the user.

### Cleanup
Add a scheduled job (queue worker, or `pg_cron`) that deletes refresh tokens that expired or were revoked more than N days ago. Without it the table grows forever.

## Audit log
Table `audit_log(id, user_id NULL, event, ip, user_agent, meta jsonb, created_at)` with `user_id REFERENCES users ON DELETE SET NULL` so the trail survives account deletion. Append-only (no UPDATE/DELETE grants for the app role).
Events: `signup`, `login`, `login_failed`, `refresh_failed`, `token_reuse`, `logout_all`, `account_deleted`, `data_exported`.
Never put tokens, passwords or full emails in `meta`.

## Account lifecycle (needed for app stores and privacy law)
- `DELETE /me`: hard delete. Delete stored files first, then in one transaction write the `account_deleted` audit row and delete the user (FK cascades do the rest). If files can still be uploaded through a live presigned URL, schedule a second purge after the URL TTL.
- `GET /me/export`: write the `data_exported` audit row **first**; if that write fails, return nothing. Leave out push tokens and secrets.

## Mobile specifics
- Devices table: `user_id`, `device_id`, `platform`, `push_token`, `last_seen_at`. A push token is unique; when another account presents the same token, move it (document it).
- `X-App-Version`: parse numerically (1.10 > 1.9) and ignore `-beta`/`+build` suffixes. When a minimum is configured, a missing or unparseable header **also** gets `426`; otherwise omitting the header bypasses the gate. Put the minimum in the response `fields.min_app_version`.
- Public `GET /v1/config` (not behind the version gate) returns `min_app_version` so an old client can show an "update required" screen.
- Skip `X-Platform` if the app ships on one platform only.

## Web specifics
- Access token in memory, refresh token in an httpOnly + Secure + SameSite=Lax/Strict cookie.
- CSRF protection on every cookie-authenticated state-changing endpoint (double-submit token, or strict `Origin`/`Referer` check).
- CORS only for origins in `CORS_ALLOWED_ORIGINS`; never `*` with credentials.

## Rate limiting
- Login/register/refresh: strict per-IP limit, plus per-account lockout for passwords.
- Authenticated routes: per user (falls back to IP if the user ID is missing, so a wiring mistake can't bypass it).
- Reject with `429` and `Retry-After`. Sweep idle limiter entries (for example after 10 minutes) so the map doesn't grow.
- In-memory limiters are **per process**: they reset on restart and don't add up across replicas. With more than one replica, use a shared store (Redis, or Postgres).
