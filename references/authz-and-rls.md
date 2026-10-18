# Authorization and data protection reference

## Object-level authorization
- Every repository method that touches user data takes `userID` and filters by it (`WHERE id = ? AND user_id = ?`). Put the exceptions (auth lookup by token hash, worker scans) behind **separate interfaces** so they can't be called by accident from a handler path.
- A resource owned by someone else returns **exactly the same response as a nonexistent ID** (404, same body). Returning 403 for foreign IDs tells an attacker which IDs exist.
- Validate path IDs as UUIDs before querying.
- Use UUID primary keys for anything user-visible.

## Mass assignment
Bind requests to DTOs in `delivery/http/dto`, never to domain or GORM models. Fields such as `role`, `user_id`, `plan` and `deleted_at` are set by the usecase only.

## PostgreSQL Row Level Security (recommended for multi-user data)
A second wall behind the `WHERE user_id = ?` filters: a forgotten filter then returns nothing instead of leaking.

1. Create the application role without superuser and without `BYPASSRLS` (for example in `deploy/postgres-init/01-app-role.sh`). Migrations run as a separate admin role.
2. For every table with a `user_id` column:
   ```sql
   ALTER TABLE questions ENABLE ROW LEVEL SECURITY;
   ALTER TABLE questions FORCE ROW LEVEL SECURITY;
   CREATE POLICY user_isolation ON questions
     USING (app_is_system() OR user_id = app_user_id())
     WITH CHECK (app_is_system() OR user_id = app_user_id());
   ```
   `app_user_id()` reads `current_setting('app.user_id', true)::uuid`, and returns `NULL` when unset, so the default is **no rows** (fail closed).
3. In Go, wrap each scoped operation in a transaction that runs `SELECT set_config('app.user_id', $1, true)` first. The `true` makes it transaction-local, so nothing leaks to the next request on a pooled connection. A GORM callback that does this for every scoped statement is cleaner than doing it by hand. A scoped query outside a transaction should return an error, not an empty result.
4. Add a trigger that forbids changing `user_id` on update.
5. At startup in production, read `pg_roles` for the current role and **refuse to start** if it is a superuser or has `BYPASSRLS`.
6. Add a test that fails when a table with `user_id` doesn't have RLS forced (query `pg_class.relrowsecurity` and `relforcerowsecurity`).

If the project is tiny, skip RLS, but keep the `userID` rule above.

## Soft delete and sync
- `deleted_at` on user content that clients sync, so deletions propagate. The authenticate middleware rejects soft-deleted users.
- Sync cursor: `(updated_at, id)` taken from the **database** clock, not the app clock, with a small overlap window (for example 15 s) so late commits aren't missed.
- Idempotency: the client generates UUIDs and writes are upserts. Batch endpoints cap their size (for example 100 operations) and run each operation in its own transaction with its own ownership check.

## Limits and quotas
- Read limits from one place (a `plans`/entitlements table or function), never hard-code numbers in handlers.
- Check quotas inside a transaction with `SELECT ... FOR UPDATE` on the user row so two parallel requests can't both pass.
- For presigned uploads, count **reserved** bytes (unexpired presigned + unprocessed) as well as stored bytes.

## File uploads (presigned object storage flow)
1. `POST /uploads/presign`: validate content type (allow-list, e.g. JPEG/WebP), size (min 1 byte, max e.g. 5 MB), batch size, UUIDs, quota. The server builds the object key (`u/{userID}/...{uploadID}.ext`); the client never chooses it. Sign `Content-Type` and `Content-Length` into the URL. TTL about 10 minutes.
2. Client PUTs the file directly to storage. The bucket is private.
3. `POST /uploads/{id}/confirm`: 404 unless the caller owns a pending upload. Enqueue a processing job keyed by upload ID (so repeating confirm is idempotent), then return `202`.
4. Worker: download with a size cap, detect the real type from the bytes (not from the header or filename), reject huge dimensions (for example > 25 megapixels), **re-encode** (drops EXIF and polyglot payloads), make thumbnails, delete the raw object. On permanent failure delete the object and mark the upload failed.
5. Reads use presigned GET URLs with a short TTL.

## Error responses
One shape for everything, defined in `internal/pkg/apperror`:
```json
{ "error": { "code": "not_found", "message": "not found", "fields": {}, "request_id": "..." } }
```
`apperror` maps error codes to HTTP statuses in a single place. Handlers return errors; one middleware writes them.

## Logging
- Log the **route pattern** (`/v1/questions/:id`), not the raw path, so IDs and tokens in URLs don't land in logs.
- Never log request bodies, tokens, passwords, emails in full, or card/ID numbers.
- Add a test that performs a login and asserts the captured log output contains no token and no email.
