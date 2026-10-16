---
name: database-migrations
description: Safe PostgreSQL schema and data migrations for Go services using golang-migrate - forward-only production changes, expand-contract renames, CREATE INDEX CONCURRENTLY, NOT NULL via CHECK NOT VALID, batched backfills, lock_timeout, dirty-state recovery, and embedding migrations with iofs. Use whenever adding or editing a migration file, changing a table, column, index, or constraint, backfilling data, planning a zero-downtime deploy, or wiring migrations into a Go service, even if the user only says "add a column".
metadata:
  origin: ECC
---

# Database Migrations (PostgreSQL + golang-migrate)

How to change a production schema without downtime, and how to run those changes with `github.com/golang-migrate/migrate/v4`. For schema and index design itself, use the postgres-patterns skill.

## When to Activate

- Creating or altering tables, columns, indexes, or constraints
- Backfilling or transforming existing data
- Planning a rename or type change that must not break running replicas
- Setting up or debugging golang-migrate (dirty state, ordering, CI)

## Core Principles

1. **Every change is a migration file.** Manual SQL in production has no review, no audit trail, and leaves environments drifting apart.
2. **Forward-only in production.** If a deployed migration was wrong, write a new migration that fixes it. Running `down` in production drops data and races with replicas still using the new schema.
3. **Deployed migrations are immutable.** golang-migrate tracks only the version number, so editing an applied file changes nothing where it already ran and creates drift with fresh environments.
4. **Old and new code must both work with the schema in between.** During a rolling deploy, the previous release keeps running against the migrated schema. Every migration has to be compatible with the code running before and after it.
5. **Separate schema changes from data changes.** DDL is fast if it avoids rewrites; backfills are slow. Mixing them keeps DDL locks held for the entire backfill.
6. **Test against production-sized data.** A migration that takes 10 ms on 100 rows can hold an `ACCESS EXCLUSIVE` lock for minutes on 50M rows.

## Locks: why "fast" DDL still causes outages

Most `ALTER TABLE` forms take an `ACCESS EXCLUSIVE` lock. Even if the change itself is instant, the lock request waits behind any long-running query on the table, and every new query (including plain `SELECT`s) then queues behind the waiting `ALTER`. One slow report plus one migration equals a full outage.

Always bound the wait so the migration fails fast and can be retried, instead of stalling traffic:

```sql
BEGIN;
SET LOCAL lock_timeout = '3s';
SET LOCAL statement_timeout = '60s';
ALTER TABLE users ADD COLUMN avatar_url text;
COMMIT;
```

`SET LOCAL` scopes the setting to this transaction, so it does not leak into later migrations that run on the same connection. A `55P03 lock_not_available` failure is safe: nothing was applied, and the version is left dirty only so you look at it (see Dirty State below).

## Safe Operation Cheat Sheet

| Change | Safe approach |
|--------|---------------|
| Add nullable column | `ADD COLUMN x text` (instant) |
| Add column with constant default | `ADD COLUMN x boolean NOT NULL DEFAULT false` (instant since PG11; volatile defaults like `random()` still rewrite) |
| Add NOT NULL to existing column | `CHECK (x IS NOT NULL) NOT VALID`, then `VALIDATE`, then `SET NOT NULL` (below) |
| Add index | `CREATE INDEX CONCURRENTLY`, alone in its own migration file |
| Add unique constraint | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... UNIQUE USING INDEX` |
| Add foreign key | `ADD CONSTRAINT ... FOREIGN KEY ... NOT VALID`, then `VALIDATE CONSTRAINT` in a later migration |
| Add CHECK constraint | `NOT VALID`, then `VALIDATE CONSTRAINT` |
| Rename column or table | Expand-contract (new column, dual write, backfill, switch reads, drop old) |
| Change column type | Expand-contract; in-place `ALTER TYPE` rewrites the table except for a few binary-compatible cases (for example `varchar(n)` to `text`) |
| Drop column | Remove all code references and deploy first, then drop |
| Drop table | Same, and keep a backup until you are sure |
| Backfill | Batched, outside the migration transaction (below) |

### Adding NOT NULL without a long lock

`SET NOT NULL` on its own scans the whole table under `ACCESS EXCLUSIVE`. A validated CHECK constraint lets Postgres (12+) skip that scan:

```sql
-- 000010_users_email_not_null_check.up.sql
-- Instant: existing rows are not checked, new writes are.
ALTER TABLE users ADD CONSTRAINT users_email_not_null CHECK (email IS NOT NULL) NOT VALID;

-- 000011_users_email_validate.up.sql
-- Scans the table but holds only SHARE UPDATE EXCLUSIVE, so reads and writes continue.
ALTER TABLE users VALIDATE CONSTRAINT users_email_not_null;

-- 000012_users_email_set_not_null.up.sql
BEGIN;
SET LOCAL lock_timeout = '3s';
ALTER TABLE users ALTER COLUMN email SET NOT NULL;  -- uses the valid CHECK, no scan
ALTER TABLE users DROP CONSTRAINT users_email_not_null;
COMMIT;
```

Backfill any NULLs between the first and second step, or `VALIDATE` will fail. Foreign keys follow the same `NOT VALID` then `VALIDATE` pattern.

### Concurrent indexes

`CREATE INDEX CONCURRENTLY` builds the index without blocking writes, but it cannot run inside a transaction block. golang-migrate's Postgres drivers send a whole file in one `Exec`, and Postgres runs a multi-statement simple query as one implicit transaction. Therefore:

- Put each `CREATE INDEX CONCURRENTLY` (or `DROP INDEX CONCURRENTLY`) **alone** in its own file with no other statement, not even a `SET`.
- If it fails midway, it leaves an `INVALID` index behind. `IF NOT EXISTS` will then silently skip creating it, so check `pg_index.indisvalid`, drop the invalid index with `DROP INDEX CONCURRENTLY`, and re-run.

```sql
-- 000013_orders_user_id_idx.up.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS orders_user_id_idx ON orders (user_id);

-- 000013_orders_user_id_idx.down.sql
DROP INDEX CONCURRENTLY IF EXISTS orders_user_id_idx;
```

### Expand-contract for renames and type changes

Never rename a column that running code reads. Spread the change over several deploys:

```
1. EXPAND    migration: add display_name (nullable)
             deploy:    write both username and display_name, read username
2. BACKFILL  job: copy username -> display_name in batches
3. SWITCH    deploy:    read display_name, still write both
             verify:    no rows where display_name IS NULL
4. CONTRACT  deploy:    stop writing username
             migration: drop username (one release later, after no old code is running)
```

Each step is independently deployable and reversible by deploying the previous release, which is what makes the whole change zero-downtime.

## Batched Backfills

A single `UPDATE big_table SET ...` holds row locks on every row until commit, bloats the table, and can run past every timeout. Run backfills in small committed batches, outside golang-migrate (its implicit transaction would wrap all batches into one). A Go job or a subcommand of the service binary is the simplest place:

```go
// Each Exec is its own short transaction; SKIP LOCKED avoids fighting live traffic.
func BackfillNormalizedEmail(ctx context.Context, pool *pgxpool.Pool) error {
	const batch = `
		WITH batch AS (
			SELECT id FROM users
			WHERE normalized_email IS NULL
			ORDER BY id
			LIMIT 5000
			FOR UPDATE SKIP LOCKED
		)
		UPDATE users u SET normalized_email = lower(u.email)
		FROM batch WHERE u.id = batch.id`
	for {
		tag, err := pool.Exec(ctx, batch)
		if err != nil {
			return fmt.Errorf("backfill batch: %w", err)
		}
		if tag.RowsAffected() == 0 {
			return nil
		}
		select { // give autovacuum and replicas room to keep up
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(100 * time.Millisecond):
		}
	}
}
```

The job must be idempotent and restartable (the `WHERE ... IS NULL` predicate does that here), and the application must already write the new column for new rows, or the backfill never finishes. Small reference-data inserts (a few rows) are fine inside a normal migration.

## golang-migrate Workflow

### Files and naming

```bash
migrate create -ext sql -dir migrations -seq -digits 6 add_user_avatar
# -> migrations/000014_add_user_avatar.up.sql
#    migrations/000014_add_user_avatar.down.sql
```

- Files are `{version}_{title}.up.sql` and `{version}_{title}.down.sql`; the version is all golang-migrate tracks (table `schema_migrations`: `version`, `dirty`).
- **Sequential vs timestamp versions.** With `-seq`, two branches can both create `000014`; the duplicate fails loudly, so CI catches it and the second author renumbers. Timestamps (the default without `-seq`) rarely collide, but a branch merged later with an older timestamp than the current DB version is **silently never applied** by `up`. Prefer `-seq` plus a CI check that runs all migrations on a fresh database.
- One logical change per file. Wrap multi-statement files in explicit `BEGIN; ... COMMIT;` with `SET LOCAL lock_timeout`, so the transaction boundary and timeouts are visible to reviewers.
- Write a `down` file for local development and tests. For irreversible changes (dropping data), make the down file fail on purpose instead of pretending:
  ```sql
  DO $$ BEGIN RAISE EXCEPTION 'migration 000015 is irreversible'; END $$;
  ```

### CLI

```bash
export DATABASE_URL='pgx5://app:secret@localhost:5432/app?sslmode=disable'
migrate -path migrations -database "$DATABASE_URL" up        # apply all pending
migrate -path migrations -database "$DATABASE_URL" version   # current version and dirty flag
migrate -path migrations -database "$DATABASE_URL" down 1    # dev only
migrate -path migrations -database "$DATABASE_URL" force 13  # mark version, no SQL runs
```

The `pgx5://` scheme selects the pgx/v5 driver (`database/pgx/v5`), so the service does not also pull in `lib/pq`. Driver options go in the URL query:

| Option | Use |
|--------|-----|
| `x-statement-timeout` | Milliseconds; caps each migration statement |
| `x-multi-statement=true` | Split the file on `;` and execute statements one by one, outside a single implicit transaction |
| `x-multi-statement-max-size` | Max size per statement in multi-statement mode (default 10MB) |
| `x-migrations-table` | Custom name for `schema_migrations` (useful when several services share a database) |

`x-multi-statement` lets you combine `SET lock_timeout` with `CREATE INDEX CONCURRENTLY`, but the naive `;` split breaks function bodies and `DO $$ ... $$` blocks, and a failure leaves earlier statements applied. Prefer the default mode with one-statement files for concurrent operations.

With docker compose locally:

```bash
docker run --rm -v "$PWD/migrations:/migrations" --network host migrate/migrate \
  -path=/migrations -database "postgres://app:secret@localhost:5432/app?sslmode=disable" up
```

The example uses `postgres://` (the lib/pq-based driver), which every build of the CLI includes; the one-implicit-transaction-per-file behavior is the same as with `pgx5://`.

### Dirty state recovery

If a migration fails, golang-migrate records `version = N, dirty = true` and refuses to run anything until a human decides what happened. Do not just `force` and move on:

1. Read the error and inspect the database: which objects from file N exist? (`\d table`, `pg_index.indisvalid`, row counts.)
2. If the file ran as one implicit or explicit transaction (default mode), nothing from it was applied. Run `force N-1`, fix the cause (lock timeout, bad data), and run `up` again.
3. If it was partially applied (multi-statement mode, or a `CONCURRENTLY` operation that left an invalid index), either finish the remaining statements by hand and `force N`, or undo the partial work by hand and `force N-1`.
4. Never edit file N if it already succeeded in another environment; add a follow-up migration instead.

### Embedding migrations in the service binary

Embedding with `iofs` ships migrations inside the binary, so the image needs no extra files and the code and schema versions always match:

```go
import (
	"embed"
	"errors"
	"fmt"
	"log/slog"

	"github.com/golang-migrate/migrate/v4"
	_ "github.com/golang-migrate/migrate/v4/database/pgx/v5" // registers pgx5://
	"github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

func MigrateUp(databaseURL string, log *slog.Logger) (err error) {
	src, err := iofs.New(migrationsFS, "migrations")
	if err != nil {
		return fmt.Errorf("open embedded migrations: %w", err)
	}
	m, err := migrate.NewWithSourceInstance("iofs", src, databaseURL)
	if err != nil {
		return fmt.Errorf("init migrate: %w", err)
	}
	defer func() {
		srcErr, dbErr := m.Close()
		err = errors.Join(err, srcErr, dbErr)
	}()

	if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
		return fmt.Errorf("migrate up: %w", err)
	}

	version, dirty, err := m.Version()
	if err != nil && !errors.Is(err, migrate.ErrNilVersion) {
		return fmt.Errorf("read migration version: %w", err)
	}
	log.Info("database migrated", "version", version, "dirty", dirty)
	return nil
}
```

`m.Up()` takes no context. To stop it on shutdown, send to `m.GracefulStop` (it finishes the current migration and then stops), rather than killing the process mid-file.

### On service start vs a separate job

| | Run in `main` on start | Separate step (job / init container / CI deploy step) |
|---|---|---|
| Setup | Simplest | Needs a pipeline step |
| Concurrency | Safe: golang-migrate takes a Postgres advisory lock, so replicas wait for each other | One runner |
| Slow migrations | Block startup; probes may kill the pod mid-migration and leave the version dirty | Run to completion with their own timeout |
| Privileges | App role needs DDL rights | Migration role owns the schema; app role stays least-privilege |
| Failure | Every replica crash-loops | Deploy stops before new code rolls out |

Recommended default: ship one binary with a `migrate` subcommand (`app migrate up`) that calls `MigrateUp`, run it as a separate step before the rollout in shared environments, and call it on start only in local docker compose and tests. Because of principle 4, the old release keeps working while the job runs.

## Testing Migrations

- In integration tests (testcontainers-go or docker compose, see the golang-testing skill), start a fresh Postgres, run `up`, then `down` to zero, then `up` again. This proves the down files work and catches non-idempotent SQL.
- In CI, apply all migrations from scratch on every PR. This also catches duplicate sequential versions.
- Before a risky migration, run it against a production-sized copy and time it; check `pg_locks` while it runs.

## Anti-Patterns

| Anti-pattern | Why it fails | Instead |
|--------------|--------------|---------|
| Editing an applied migration | Never re-runs where it already ran; environments drift | New migration |
| `force` without inspecting | Hides a half-applied migration | Follow Dirty State recovery |
| `CREATE INDEX` (non-concurrent) on a live table | Blocks writes for the whole build | `CONCURRENTLY`, alone in a file |
| `CONCURRENTLY` next to other statements | Fails: runs inside the implicit transaction | One statement per file |
| DDL without `lock_timeout` | Queues all traffic behind the waiting lock | `SET LOCAL lock_timeout` |
| `ADD COLUMN ... NOT NULL` without default on existing table | Fails outright once the table has rows | Nullable, backfill, CHECK NOT VALID, VALIDATE, SET NOT NULL |
| One giant `UPDATE` backfill in a migration | Long locks, bloat, timeouts | Batched Go job |
| Dropping a column the old release still reads | Errors during rolling deploy | Remove code first, drop one release later |
| `down` in production | Data loss, breaks the running release | Forward fix |
