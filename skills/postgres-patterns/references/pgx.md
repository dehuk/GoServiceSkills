# pgx/v5 Reference Code

Complete examples for the "Go Access with pgx/v5" section of the postgres-patterns skill. They were compile-checked against `github.com/jackc/pgx/v5` v5.11 and `github.com/jackc/pgerrcode`. All snippets live in one `store` package with these imports:

```go
import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/jackc/pgerrcode"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jackc/pgx/v5/pgxpool"
)
```

## Pool setup

```go
func NewPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse postgres dsn: %w", err)
	}

	// replicas * MaxConns must stay below max_connections minus admin/migration headroom.
	cfg.MaxConns = 20
	cfg.MinConns = 2
	cfg.MaxConnLifetime = time.Hour            // rotate after failover / DNS change
	cfg.MaxConnLifetimeJitter = 5 * time.Minute // avoid reconnect storms
	cfg.MaxConnIdleTime = 15 * time.Minute
	cfg.HealthCheckPeriod = 30 * time.Second
	cfg.ConnConfig.ConnectTimeout = 5 * time.Second

	// Server-side guard rails. Values are milliseconds. Behind PgBouncer, set these
	// on the role instead (ALTER ROLE ... SET), see the skill.
	cfg.ConnConfig.RuntimeParams["application_name"] = "orders-service"
	cfg.ConnConfig.RuntimeParams["statement_timeout"] = "5000"
	cfg.ConnConfig.RuntimeParams["idle_in_transaction_session_timeout"] = "10000"

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}

	// NewWithConfig connects lazily; Ping makes a bad DSN or unreachable DB fail at startup.
	pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()
	if err := pool.Ping(pingCtx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("ping postgres: %w", err)
	}
	return pool, nil
}

// Use when PgBouncer runs in transaction mode without prepared-statement support.
func ConfigureForPgBouncer(cfg *pgxpool.Config) {
	cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec
}
```

In `main`: `pool, err := store.NewPool(ctx, cfg.DatabaseURL)`, then `defer pool.Close()`. Close the pool after the HTTP server has finished shutting down, so in-flight requests can still query.

## Domain errors and the DBTX interface

```go
// In a real service these live in the domain package; the repository wraps them.
var (
	ErrNotFound = errors.New("not found")
	ErrConflict = errors.New("conflict")
)

// DBTX is satisfied by *pgxpool.Pool, *pgx.Conn and pgx.Tx, so one repository
// works both inside and outside a transaction.
type DBTX interface {
	Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
	Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
	QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

var (
	_ DBTX = (*pgxpool.Pool)(nil)
	_ DBTX = (pgx.Tx)(nil)
)
```

## Repository: reading rows, not found, constraint errors

```go
type User struct {
	ID        int64     `db:"id"`
	Email     string    `db:"email"`
	Name      *string   `db:"name"` // nullable column -> pointer
	CreatedAt time.Time `db:"created_at"`
}

type UserRepo struct {
	db DBTX
}

func NewUserRepo(db DBTX) *UserRepo { return &UserRepo{db: db} }

// Explicit column list: SELECT * breaks RowToStructByName when a column is added.
const userColumns = `id, email, name, created_at`

func (r *UserRepo) GetByID(ctx context.Context, id int64) (User, error) {
	rows, err := r.db.Query(ctx, `SELECT `+userColumns+` FROM users WHERE id = $1`, id)
	if err != nil {
		return User{}, fmt.Errorf("query user %d: %w", id, err)
	}
	// CollectExactlyOneRow closes rows, checks rows.Err, and returns ErrNoRows / ErrTooManyRows.
	u, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
	if errors.Is(err, pgx.ErrNoRows) {
		return User{}, fmt.Errorf("user %d: %w", id, ErrNotFound)
	}
	if err != nil {
		return User{}, fmt.Errorf("scan user %d: %w", id, err)
	}
	return u, nil
}

// QueryRow + Scan is fine for a few scalar columns.
func (r *UserRepo) EmailByID(ctx context.Context, id int64) (string, error) {
	var email string
	err := r.db.QueryRow(ctx, `SELECT email FROM users WHERE id = $1`, id).Scan(&email)
	if errors.Is(err, pgx.ErrNoRows) {
		return "", fmt.Errorf("user %d: %w", id, ErrNotFound)
	}
	if err != nil {
		return "", fmt.Errorf("get email for user %d: %w", id, err)
	}
	return email, nil
}

// One query for many IDs instead of N+1: pgx encodes []int64 as bigint[].
func (r *UserRepo) ListByIDs(ctx context.Context, ids []int64) ([]User, error) {
	rows, err := r.db.Query(ctx,
		`SELECT `+userColumns+` FROM users WHERE id = ANY($1) ORDER BY id`, ids)
	if err != nil {
		return nil, fmt.Errorf("query users: %w", err)
	}
	users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
	if err != nil {
		return nil, fmt.Errorf("scan users: %w", err)
	}
	return users, nil
}

func (r *UserRepo) Create(ctx context.Context, email string) (int64, error) {
	var id int64
	err := r.db.QueryRow(ctx,
		`INSERT INTO users (email) VALUES ($1) RETURNING id`, email).Scan(&id)
	if err != nil {
		return 0, fmt.Errorf("insert user: %w", mapWriteErr(err))
	}
	return id, nil
}

// mapWriteErr turns constraint violations into domain errors and leaves
// everything else untouched (still wrapped by the caller).
func mapWriteErr(err error) error {
	var pgErr *pgconn.PgError
	if !errors.As(err, &pgErr) {
		return err
	}
	switch pgErr.Code {
	case pgerrcode.UniqueViolation: // 23505
		return fmt.Errorf("%s: %w", pgErr.ConstraintName, ErrConflict)
	case pgerrcode.ForeignKeyViolation: // 23503
		return fmt.Errorf("%s: %w", pgErr.ConstraintName, ErrNotFound)
	default:
		return err
	}
}

func IsRetryable(err error) bool {
	var pgErr *pgconn.PgError
	if !errors.As(err, &pgErr) {
		return false
	}
	switch pgErr.Code {
	case pgerrcode.SerializationFailure, pgerrcode.DeadlockDetected: // 40001, 40P01
		return true
	}
	return false
}
```

Do not put `pgErr.Message` or `pgErr.Detail` into API responses: they can contain other users' values (for example the duplicate key).

## Transactions with BeginFunc

```go
type Order struct {
	ID     int64
	UserID int64
	Total  int64 // minor units
}

var ErrInsufficientFunds = errors.New("insufficient funds")

func PlaceOrder(ctx context.Context, pool *pgxpool.Pool, o Order) (int64, error) {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	var orderID int64
	// Commit on nil, Rollback on error or panic. Use tx (not pool) inside.
	err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
		var balance int64
		err := tx.QueryRow(ctx,
			`SELECT balance FROM accounts WHERE user_id = $1 FOR UPDATE`, o.UserID).Scan(&balance)
		if errors.Is(err, pgx.ErrNoRows) {
			return fmt.Errorf("account for user %d: %w", o.UserID, ErrNotFound)
		}
		if err != nil {
			return fmt.Errorf("lock account: %w", err)
		}
		if balance < o.Total {
			return ErrInsufficientFunds
		}
		if _, err := tx.Exec(ctx,
			`UPDATE accounts SET balance = balance - $1 WHERE user_id = $2`, o.Total, o.UserID); err != nil {
			return fmt.Errorf("debit account: %w", err)
		}
		if err := tx.QueryRow(ctx,
			`INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING id`,
			o.UserID, o.Total).Scan(&orderID); err != nil {
			return fmt.Errorf("insert order: %w", mapWriteErr(err))
		}
		return nil
	})
	if err != nil {
		return 0, err
	}
	return orderID, nil
}
```

To reuse repositories inside the transaction, construct them from `tx`: `users := NewUserRepo(tx)`. A unit-of-work helper in the application layer (for example `WithTx(ctx, func(repos Repos) error)`) keeps pgx out of the service code.

## Manual transaction with correct rollback

Use this shape only when `BeginFunc` does not fit (for example when the transaction spans several methods).

```go
func Transfer(ctx context.Context, pool *pgxpool.Pool, from, to, amount int64) (err error) {
	tx, err := pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin: %w", err)
	}
	defer func() {
		// After a successful Commit, Rollback is a no-op that returns ErrTxClosed.
		if rbErr := tx.Rollback(ctx); rbErr != nil && !errors.Is(rbErr, pgx.ErrTxClosed) {
			err = errors.Join(err, fmt.Errorf("rollback: %w", rbErr))
		}
	}()

	// Lock both rows in id order so concurrent transfers A->B and B->A cannot deadlock.
	if _, err = tx.Exec(ctx,
		`SELECT 1 FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE`, from, to); err != nil {
		return fmt.Errorf("lock accounts: %w", err)
	}
	if _, err = tx.Exec(ctx, `UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amount, from); err != nil {
		return fmt.Errorf("debit: %w", err)
	}
	if _, err = tx.Exec(ctx, `UPDATE accounts SET balance = balance + $1 WHERE id = $2`, amount, to); err != nil {
		return fmt.Errorf("credit: %w", err)
	}
	if err = tx.Commit(ctx); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}
```

## Serializable with bounded retry

```go
// fn may run several times, so it must not have side effects outside the database.
func RunSerializable(ctx context.Context, pool *pgxpool.Pool, fn func(pgx.Tx) error) error {
	const maxAttempts = 3
	var err error
	for attempt := range maxAttempts {
		err = pgx.BeginTxFunc(ctx, pool, pgx.TxOptions{IsoLevel: pgx.Serializable}, fn)
		if err == nil || !IsRetryable(err) {
			return err
		}
		select {
		case <-ctx.Done():
			return errors.Join(err, ctx.Err())
		case <-time.After(time.Duration(attempt+1) * 20 * time.Millisecond):
		}
	}
	return fmt.Errorf("after %d attempts: %w", maxAttempts, err)
}
```

## Batch

```go
type OrderItem struct {
	OrderID   int64
	ProductID int64
	Qty       int32
}

// Satisfied by *pgxpool.Pool, *pgx.Conn and pgx.Tx.
type batchSender interface {
	SendBatch(ctx context.Context, b *pgx.Batch) pgx.BatchResults
}

func UpsertItems(ctx context.Context, db batchSender, items []OrderItem) error {
	b := &pgx.Batch{}
	for _, it := range items {
		b.Queue(`INSERT INTO order_items (order_id, product_id, qty) VALUES ($1, $2, $3)
		         ON CONFLICT (order_id, product_id) DO UPDATE SET qty = EXCLUDED.qty`,
			it.OrderID, it.ProductID, it.Qty)
	}
	br := db.SendBatch(ctx, b)
	// Read results in queue order; Close must always run and its error matters.
	for range b.Len() {
		if _, err := br.Exec(); err != nil {
			return errors.Join(fmt.Errorf("upsert item: %w", mapWriteErr(err)), br.Close())
		}
	}
	return br.Close()
}
```

A batch sent on the pool is not atomic: statements before a failing one stay committed. Send it on a `pgx.Tx` when all-or-nothing matters.

## COPY

```go
func CopyItems(ctx context.Context, pool *pgxpool.Pool, items []OrderItem) (int64, error) {
	n, err := pool.CopyFrom(ctx,
		pgx.Identifier{"order_items"},
		[]string{"order_id", "product_id", "qty"},
		pgx.CopyFromSlice(len(items), func(i int) ([]any, error) {
			it := items[i]
			return []any{it.OrderID, it.ProductID, it.Qty}, nil
		}),
	)
	if err != nil {
		return 0, fmt.Errorf("copy order_items: %w", err)
	}
	return n, nil
}
```

COPY is all-or-nothing for the call and has no `ON CONFLICT`. For an upsert of many rows, run inside a transaction: `CREATE TEMP TABLE tmp (LIKE order_items) ON COMMIT DROP`, `CopyFrom` into `tmp`, then `INSERT INTO order_items SELECT ... FROM tmp ON CONFLICT ...`.

## sqlc configuration (optional)

```yaml
version: "2"
sql:
  - engine: postgresql
    schema: migrations        # golang-migrate .up.sql files are read as the schema
    queries: internal/store/queries
    gen:
      go:
        package: db
        out: internal/store/db
        sql_package: pgx/v5
        emit_interface: true
        emit_pointers_for_null_types: true
```

sqlc returns `pgx.ErrNoRows` from `:one` queries and raw `*pgconn.PgError` from writes, so wrap the generated `Queries` in a repository that applies the same `ErrNotFound` / `mapWriteErr` mapping.
