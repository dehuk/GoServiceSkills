# Integration tests with testcontainers-go

Start real Postgres, Redis, and NATS from test code. Requires a running Docker daemon (Docker Desktop, Colima, or the CI runner's Docker). Container images should match the major versions used in production and in the project's docker compose file, so tests exercise the same behavior.

Modules used below:

```text
github.com/testcontainers/testcontainers-go
github.com/testcontainers/testcontainers-go/modules/postgres
github.com/testcontainers/testcontainers-go/modules/redis
github.com/testcontainers/testcontainers-go/modules/nats
```

Store-specific usage (queries, keys, streams and consumers) belongs to the **postgres-patterns**, **redis-patterns**, and **nats-jetstream-patterns** skills; migration tooling to **database-migrations**. This file only covers getting a real instance into a test.

## Gate integration tests

Put integration tests in files with a build tag so `go test ./...` stays fast and does not need Docker:

```go
//go:build integration

package postgres_test
```

Run them with `go test -tags=integration ./...`. Alternatively skip at runtime with `if testing.Short() { t.Skip("needs Docker") }`; the tag is more explicit and keeps the test binary free of container dependencies.

## One container per test (simplest)

Fine for packages with a handful of integration tests. `CleanupContainer` is registered before the error check on purpose: if `Run` fails after creating the container, it still gets removed.

```go
func newPostgres(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := t.Context()

    ctr, err := tcpostgres.Run(ctx, "postgres:17-alpine",
        tcpostgres.WithDatabase("app"),
        tcpostgres.WithUsername("app"),
        tcpostgres.WithPassword("app"),
        tcpostgres.BasicWaitStrategies(), // waits until Postgres really accepts connections
    )
    testcontainers.CleanupContainer(t, ctr)
    if err != nil {
        t.Fatalf("start postgres: %v", err)
    }

    dsn, err := ctr.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("postgres dsn: %v", err)
    }
    if err := applyMigrations(dsn); err != nil {
        t.Fatalf("migrate: %v", err)
    }

    pool, err := pgxpool.New(ctx, dsn)
    if err != nil {
        t.Fatalf("connect: %v", err)
    }
    t.Cleanup(pool.Close)
    return pool
}
```

Apply the project's real migrations rather than a hand-written test schema; otherwise tests pass against a schema production never has. With golang-migrate and its pgx v5 driver:

```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/pgx/v5" // registers the pgx5:// scheme
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func applyMigrations(dsn string) error {
    m, err := migrate.New("file://../../migrations", strings.Replace(dsn, "postgres://", "pgx5://", 1))
    if err != nil {
        return err
    }
    defer m.Close() // returns source and database close errors; irrelevant for a throwaway container
    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return err
    }
    return nil
}
```

The relative `file://` path is resolved from the package directory, since `go test` runs each package in its own directory.

## One container per package (faster)

Starting Postgres takes a few seconds. For packages with many tests, start it once in `TestMain` and give each test an isolated database cloned from a migrated template.

```go
var adminDSN string // connection to the "postgres" maintenance database

func TestMain(m *testing.M) {
    os.Exit(runTests(m)) // os.Exit skips defers, so the real work lives in runTests
}

func runTests(m *testing.M) int {
    ctx := context.Background()
    ctr, err := tcpostgres.Run(ctx, "postgres:17-alpine",
        tcpostgres.WithDatabase("app_template"),
        tcpostgres.WithUsername("app"),
        tcpostgres.WithPassword("app"),
        tcpostgres.BasicWaitStrategies(),
    )
    defer func() {
        if err := testcontainers.TerminateContainer(ctr); err != nil { // nil-safe
            log.Printf("terminate postgres: %v", err)
        }
    }()
    if err != nil {
        log.Printf("start postgres: %v", err)
        return 1
    }

    templateDSN, err := ctr.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        log.Printf("postgres dsn: %v", err)
        return 1
    }
    if err := applyMigrations(templateDSN); err != nil {
        log.Printf("migrate: %v", err)
        return 1
    }
    adminDSN = strings.Replace(templateDSN, "/app_template?", "/postgres?", 1)
    return m.Run()
}

// newTestDB returns a pool on a fresh copy of the migrated template database.
func newTestDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    ctx := t.Context()

    admin, err := pgx.Connect(ctx, adminDSN)
    if err != nil {
        t.Fatalf("connect admin: %v", err)
    }
    defer admin.Close(context.Background())

    name := "test_" + strings.ToLower(rand.Text()) // crypto/rand, Go 1.24
    ident := pgx.Identifier{name}.Sanitize()
    if _, err := admin.Exec(ctx, "CREATE DATABASE "+ident+" TEMPLATE app_template"); err != nil {
        t.Fatalf("create database: %v", err)
    }
    t.Cleanup(func() {
        // t.Context() is already cancelled when cleanups run.
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        conn, err := pgx.Connect(ctx, adminDSN)
        if err != nil {
            t.Errorf("connect admin for cleanup: %v", err)
            return
        }
        defer conn.Close(ctx)
        if _, err := conn.Exec(ctx, "DROP DATABASE "+ident+" WITH (FORCE)"); err != nil {
            t.Errorf("drop database: %v", err)
        }
    })

    pool, err := pgxpool.New(ctx, strings.Replace(adminDSN, "/postgres?", "/"+name+"?", 1))
    if err != nil {
        t.Fatalf("connect test db: %v", err)
    }
    t.Cleanup(pool.Close) // cleanups run LIFO: the pool closes before the database is dropped
    return pool
}
```

`CREATE DATABASE ... TEMPLATE` copies files and takes tens of milliseconds, so each test, including parallel ones, gets a clean schema with real constraints. It requires that nothing is connected to the template; that holds because the migration connection is closed before `m.Run()`.

Cheaper alternatives when per-test databases are too slow: run each test inside a transaction that is rolled back (does not work when the code under test manages its own transactions), or `TRUNCATE ... CASCADE` the touched tables in cleanup (tests in the package then cannot run in parallel).

## Redis

```go
func newRedis(t *testing.T) *redis.Client {
    t.Helper()
    ctx := t.Context()

    ctr, err := tcredis.Run(ctx, "redis:7-alpine")
    testcontainers.CleanupContainer(t, ctr)
    if err != nil {
        t.Fatalf("start redis: %v", err)
    }
    uri, err := ctr.ConnectionString(ctx) // redis://host:port
    if err != nil {
        t.Fatalf("redis uri: %v", err)
    }
    opts, err := redis.ParseURL(uri)
    if err != nil {
        t.Fatalf("parse redis uri: %v", err)
    }
    rdb := redis.NewClient(opts)
    t.Cleanup(func() {
        if err := rdb.Close(); err != nil {
            t.Errorf("close redis: %v", err)
        }
    })
    return rdb
}
```

When sharing one Redis container across a package, isolate tests with a unique key prefix per test (for example derived from `t.Name()`) rather than `FLUSHDB`, which breaks parallel tests.

## NATS with JetStream

```go
func newJetStream(t *testing.T) (*nats.Conn, jetstream.JetStream) {
    t.Helper()
    ctx := t.Context()

    ctr, err := tcnats.Run(ctx, "nats:2.14-alpine") // the module starts the server with JetStream (-js) enabled
    testcontainers.CleanupContainer(t, ctr)
    if err != nil {
        t.Fatalf("start nats: %v", err)
    }
    url, err := ctr.ConnectionString(ctx)
    if err != nil {
        t.Fatalf("nats url: %v", err)
    }
    nc, err := nats.Connect(url)
    if err != nil {
        t.Fatalf("connect nats: %v", err)
    }
    t.Cleanup(nc.Close)

    js, err := jetstream.New(nc) // new jetstream package API, not legacy nc.JetStream()
    if err != nil {
        t.Fatalf("jetstream: %v", err)
    }
    return nc, js
}
```

Create streams and consumers in the test with the same configuration code the service uses at startup, so the test also verifies that configuration. To wait for an asynchronously consumed message, receive from a channel with a `select` on a timeout (for example 5s) instead of sleeping.

## docker compose alternative

If the project already has a `compose.yaml` for local development, tests can use it instead:

```bash
docker compose up -d --wait   # --wait blocks until healthchecks pass
TEST_DATABASE_URL=postgres://app:app@localhost:5432/app?sslmode=disable \
  go test -tags=integration ./...
```

Read the connection settings from environment variables in the test helpers and skip with a clear message when they are missing. Isolation rules are the same as above, because the stack is shared by every test and every run.
