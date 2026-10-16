---
name: postgres-patterns
description: PostgreSQL schema design, indexing, query and locking patterns, plus Go data access with pgx/v5 (pgxpool config, context timeouts, CollectRows, transactions, batches, COPY, PgError mapping, PgBouncer caveats). Use whenever writing or reviewing SQL, a repository/store layer in Go, pgx code, a schema or index, or when a query is slow or deadlocks, even if the user does not mention Postgres or pgx by name.
metadata:
  origin: ECC
---

# PostgreSQL Patterns (with Go and pgx/v5)

Recommended defaults for services that talk to PostgreSQL through `github.com/jackc/pgx/v5` and `pgxpool`. For changing a schema safely, use the database-migrations skill. For a full review pass, use the database-reviewer agent.

## When to Activate

- Designing tables, constraints, or indexes
- Writing SQL or a Go repository/store layer that uses pgx
- A query is slow, a table is bloated, or the service sees lock waits or deadlocks
- Configuring the connection pool or putting PgBouncer in front of Postgres
- Mapping database errors to domain errors

## Schema Design

### Data types

| Use case | Prefer | Avoid | Why |
|----------|--------|-------|-----|
| Surrogate IDs | `bigint GENERATED ALWAYS AS IDENTITY` | `int`, `serial` | `int` overflows at 2.1B; identity columns are the SQL-standard replacement for `serial` and block accidental manual inserts |
| Public / distributed IDs | UUIDv7 (`uuidv7()` on PG18+, or generated in Go) | random UUIDv4 as PK | v4 scatters inserts across the B-tree, causing page splits and cache misses; v7 is time-ordered |
| Strings | `text` (+ `CHECK (length(x) <= n)` if a limit is a business rule) | `varchar(255)` | Same storage; an arbitrary limit is a hidden validation rule that is painful to change |
| Timestamps | `timestamptz` | `timestamp` | `timestamp` drops the zone, so values silently shift when server or client zones differ; scans into `time.Time` |
| Money | `numeric(p,s)` or `bigint` minor units | `float`, `real` | Floats cannot represent decimal fractions exactly |
| Flags | `boolean NOT NULL DEFAULT false` | `int`, `varchar` | Three-state nullable booleans are a common source of bugs |
| Semi-structured | `jsonb` | `json`, `text` | `jsonb` is indexable (GIN) and normalized |

### Constraints are the last line of defense

- Every table has a primary key. Every foreign key column has an index (Postgres does not create one; without it, deletes on the parent and joins scan the child).
- Choose `ON DELETE` explicitly (`RESTRICT`, `CASCADE`, `SET NULL`) so the behavior is a decision, not a default.
- Use `NOT NULL` by default and allow NULL only when "unknown" is a real state.
- Enforce uniqueness with a unique index or constraint, not with "check then insert" in Go: two requests can both pass the check. Let the database reject the duplicate and map `23505` (see below).
- Use `lowercase_snake_case` identifiers; quoted mixed-case names must be quoted forever.

### Row Level Security (optional)

RLS is a general Postgres feature for multi-tenant tables. If you use it, set the tenant per transaction (`SELECT set_config('app.tenant_id', $1, true)`), reference it in policies as `(SELECT current_setting('app.tenant_id')::bigint) = tenant_id` (wrapping in `SELECT` makes Postgres evaluate it once per query instead of per row), and index the tenant column. Remember that table owners and superusers bypass RLS unless `FORCE ROW LEVEL SECURITY` is set, so run the service as a non-owner role.

## Indexes

| Query pattern | Index | Example |
|---------------|-------|---------|
| `WHERE col = $1`, ranges, `ORDER BY` | B-tree (default) | `CREATE INDEX ON t (col)` |
| `WHERE a = $1 AND b > $2` | Composite, equality columns first | `CREATE INDEX ON t (a, b)` |
| `jsonb @> '{...}'`, arrays, full text | GIN | `CREATE INDEX ON t USING gin (col)` |
| Append-only time series, huge tables | BRIN | `CREATE INDEX ON t USING brin (created_at)` |

- **Composite order**: equality columns first, then the range or sort column. `(status, created_at)` serves `WHERE status = 'pending' AND created_at > $1 ORDER BY created_at`; `(created_at, status)` does not serve it well.
- **Covering**: `CREATE INDEX ON users (email) INCLUDE (name)` enables index-only scans for `SELECT name FROM users WHERE email = $1`.
- **Partial**: `CREATE INDEX ON users (email) WHERE deleted_at IS NULL` keeps the index small for soft-deleted tables; the query must repeat the predicate to use it.
- **Expression**: `CREATE UNIQUE INDEX ON users (lower(email))` for case-insensitive lookups; the query must use the same expression.
- Every index slows writes and costs memory. Drop unused ones (check `pg_stat_user_indexes.idx_scan`).
- On existing tables, create indexes with `CONCURRENTLY` in a migration of its own (see the database-migrations skill).

## Query Patterns

**Select explicit columns.** `SELECT *` breaks `RowToStructByName` when a column is added, defeats index-only scans, and with pgx's statement cache can fail with "cached plan must not change result type" after a migration.

**Upsert** instead of read-modify-write:
```sql
INSERT INTO settings (user_id, key, value)
VALUES ($1, $2, $3)
ON CONFLICT (user_id, key) DO UPDATE SET value = EXCLUDED.value;
```

**Keyset (cursor) pagination** instead of `OFFSET`, which reads and discards every skipped row:
```sql
SELECT id, name, created_at FROM products
WHERE (created_at, id) < ($1, $2)          -- values from the last row of the previous page
ORDER BY created_at DESC, id DESC
LIMIT 50;
-- needs: CREATE INDEX ON products (created_at DESC, id DESC)
```

**Batch lookups instead of N+1.** pgx encodes a Go slice as a Postgres array, so one query replaces a loop:
```sql
SELECT id, email FROM users WHERE id = ANY($1);   -- $1 is []int64
```

**Verify with `EXPLAIN (ANALYZE, BUFFERS)`** on production-sized data. Look for `Seq Scan` on large tables, row estimates far from actuals (run `ANALYZE`), and `Sort` or `Hash` spilling to disk.

## Locking and Concurrency

- **Keep transactions short.** A transaction holds its row locks and blocks vacuum cleanup until it ends. Never call external services (HTTP, NATS publish, Redis) inside a transaction; do the I/O before or after, or use an outbox table written in the same transaction.
- **Lock in a consistent order** to avoid deadlocks: `SELECT ... WHERE id = ANY($1) ORDER BY id FOR UPDATE`.
- **Pick the weakest lock that works.** `FOR UPDATE` blocks other writers and `FOR UPDATE` readers; `FOR NO KEY UPDATE` is enough when you will not change the key and lets FK inserts proceed; `FOR SHARE` keeps the row from changing while you read it.
- **Optimistic concurrency** for user-facing edits: add `version bigint NOT NULL DEFAULT 0`, update with `WHERE id = $1 AND version = $2`, `SET version = version + 1`, and treat `RowsAffected() == 0` as a conflict.
- **Job queues with `SKIP LOCKED`** so workers do not block each other:
  ```sql
  UPDATE jobs SET status = 'processing', locked_at = now()
  WHERE id = (
    SELECT id FROM jobs WHERE status = 'pending'
    ORDER BY created_at LIMIT 1
    FOR UPDATE SKIP LOCKED
  ) RETURNING id, payload;
  ```
- **Advisory locks** (`pg_try_advisory_xact_lock(key)`) serialize work that has no natural row to lock, such as "only one replica runs this cron". The `_xact_` variant releases at transaction end, which survives panics and pool reuse; session-level advisory locks leak onto pooled connections and do not work through PgBouncer in transaction mode.
- **Isolation**: the default `READ COMMITTED` is right for most code. Use `SERIALIZABLE` for invariants that span rows (for example "at most N bookings per slot"), and retry on `40001`.
- **Guard rails** on the server or per role: `lock_timeout`, `statement_timeout`, and `idle_in_transaction_session_timeout` turn a stuck request into an error instead of an outage.

## Go Access with pgx/v5

Full, compile-checked code for everything in this section is in `references/pgx.md`. Read it when you write or review a pool setup, a repository, a transaction, a batch/COPY, or error mapping.

### Pool

- Build the pool once in `main` with `pgxpool.ParseConfig` + `pgxpool.NewWithConfig`, `Ping` with a timeout so a bad DSN fails at startup, and `defer pool.Close()`.
- Set `MaxConns` deliberately: the total across all replicas (`replicas * MaxConns`) must stay below Postgres `max_connections` minus headroom for migrations and admins. The default (max of 4 and `runtime.NumCPU()`) is rarely right for a container.
- Set `MaxConnLifetime` (+ `MaxConnLifetimeJitter`) so connections rotate after failovers and DNS changes, `MaxConnIdleTime` to shed idle connections, and `HealthCheckPeriod` so dead connections are found before a request gets them.
- Set `statement_timeout` and `idle_in_transaction_session_timeout` via `ConnConfig.RuntimeParams`, and an `application_name` so the service is identifiable in `pg_stat_activity`.
- Expose `pool.Ping(ctx)` in the readiness probe, not the liveness probe: a database outage should take the pod out of rotation, not restart it in a loop.

### Context and timeouts

Every query takes the request `ctx`. Add a per-operation deadline with `context.WithTimeout` for anything that may be slow, and always `defer cancel()`. When the context is cancelled, pgx sends a cancel request and closes that connection; the pool replaces it. That is why `statement_timeout` on the server is still worth having: it protects the database from clients that never cancel.

### Reading rows

- Prefer `pgx.CollectRows(rows, pgx.RowToStructByName[T])` and `pgx.CollectExactlyOneRow`. They close `rows` and check `rows.Err()` for you; hand-written `for rows.Next()` loops that forget `rows.Err()` silently return partial results.
- `RowToStructByName` matches columns to exported fields by `db:"col"` tag (or field name) and fails if a selected column has no field. Use `RowToStructByNameLax` only when the struct intentionally has extra fields.
- Scan nullable columns into pointers (`*string`) or `pgtype.Text`, never into a plain `string`.

### Not found and constraint errors

Map driver errors to domain errors inside the repository, so callers never import pgx:

- `errors.Is(err, pgx.ErrNoRows)` becomes the domain `ErrNotFound`, wrapped with context (`fmt.Errorf("user %d: %w", id, ErrNotFound)`).
- `errors.As(err, &pgErr)` with `*pgconn.PgError` exposes `Code` and `ConstraintName`. Use `github.com/jackc/pgerrcode` constants instead of string literals.

| Code | Name | Typical mapping |
|------|------|-----------------|
| `23505` | unique_violation | `ErrConflict` / "already exists" (switch on `ConstraintName` for the field) |
| `23503` | foreign_key_violation | referenced entity not found, or "still in use" on delete |
| `23502` | not_null_violation | bug or validation gap; treat as internal |
| `23514` | check_violation | invalid input |
| `40001` | serialization_failure | retry the whole transaction |
| `40P01` | deadlock_detected | retry, then fix the lock ordering |
| `55P03` | lock_not_available | `lock_timeout` hit or `NOWAIT`; retry or fail fast |
| `57014` | query_canceled | `statement_timeout` or context cancel; do not retry blindly |

See the error-handling skill for how domain errors map to API responses, and the hexagonal-architecture skill for where the repository sits.

### Transactions

- Prefer `pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {...})` (or `BeginTxFunc` with `pgx.TxOptions`). It commits when the function returns nil and rolls back on error or panic, so no code path forgets the rollback.
- Inside the function use `tx`, never `pool`. A query on `pool` runs on a different connection, outside the transaction, and can deadlock against your own locks.
- When you manage `tx` by hand, `defer` a rollback right after `Begin`. Rollback after a successful `Commit` is a no-op returning `pgx.ErrTxClosed`; ignore exactly that error.
- Let repositories accept a small `DBTX` interface (`Exec`, `Query`, `QueryRow`) that both `*pgxpool.Pool` and `pgx.Tx` satisfy, so the same repository works inside and outside a transaction.
- Retry `SERIALIZABLE` transactions on `40001`/`40P01` a bounded number of times with backoff, re-running the whole function; it must therefore be free of side effects outside the database.

### Bulk writes

- `pgx.Batch` + `SendBatch` sends many statements in one round trip. Read every result with `br.Exec()` (or `Query`/`QueryRow`) in order, then `br.Close()` and check its error; an unread result or unchecked `Close` hides failures.
- `pool.CopyFrom` with `pgx.CopyFromSlice`/`CopyFromRows` is the fastest way to load thousands of rows. COPY does not support `ON CONFLICT`; for upserts, COPY into a temp table and `INSERT ... SELECT ... ON CONFLICT`, or use a batch.
- Never insert row by row in a loop: each statement pays a full network round trip.

### PgBouncer and prepared statements

pgx's default exec mode (`QueryExecModeCacheStatement`) creates named prepared statements per connection. In PgBouncer transaction pooling, the next transaction may land on a different server connection that has never seen that statement, giving errors like `prepared statement "stmtcache_..." does not exist`. Options, in order of preference:

1. PgBouncer 1.21+ with `max_prepared_statements` > 0, which tracks protocol-level prepared statements; keep the pgx default.
2. `cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec`: unnamed statements, one round trip, no server-side cache.
3. `pgx.QueryExecModeSimpleProtocol`: client-side parameter interpolation; check its type caveats (for example `[]byte` is always `bytea`, so pass JSON as `string`).

Also, with PgBouncer: startup `RuntimeParams` such as `statement_timeout` may be rejected unless PgBouncer's `ignore_startup_parameters` lists them (set them on the database role instead), session-level `SET`, `LISTEN`, and session advisory locks do not work, and `MaxConns` sizing applies to PgBouncer's client limit rather than Postgres.

### sqlc (optional)

`sqlc` generates typed Go from SQL files and supports pgx/v5 (`sql_package: "pgx/v5"`). It keeps SQL as the source of truth, catches column/type mismatches at generate time, and its generated `DBTX` interface works with both pool and `pgx.Tx`. It is a good fit when a service has many static queries; hand-written pgx is fine for a few queries or highly dynamic filters. Either way, the error-mapping and transaction rules above still apply in the repository that wraps the generated code.

## Diagnostics

```sql
-- Foreign keys without a supporting index
SELECT c.conrelid::regclass AS table_name, a.attname AS column_name
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY (c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid AND i.indkey[0] = a.attnum
  );

-- Slowest queries (requires pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;

-- Unused indexes
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;

-- Dead tuples / vacuum lag
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Who is blocking whom
SELECT pid, pg_blocking_pids(pid) AS blocked_by, state, wait_event_type, left(query, 80)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Long-running or idle-in-transaction sessions
SELECT pid, application_name, state, now() - xact_start AS xact_age, left(query, 80)
FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY xact_start LIMIT 20;
```

## Server Settings Worth Checking

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- also needs shared_preload_libraries
ALTER ROLE app_user SET statement_timeout = '5s';
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '30s';
ALTER ROLE app_user SET lock_timeout = '3s';
REVOKE ALL ON SCHEMA public FROM PUBLIC;            -- least privilege; grant only what the app needs
```

Setting timeouts per role (rather than `ALTER SYSTEM`) keeps migrations and admin sessions free to use different limits. The application role should own nothing and have only `SELECT/INSERT/UPDATE/DELETE` on its tables; a separate migration role owns the schema.
