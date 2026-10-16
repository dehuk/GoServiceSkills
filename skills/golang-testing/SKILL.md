---
name: golang-testing
description: Go testing practice for backend services (Go 1.24+) - table-driven tests and subtests, internal vs external test packages, fakes over mocks, httptest handler tests, golden files, deterministic time and concurrency tests, benchmarks with b.Loop, fuzzing, integration tests against real Postgres/Redis/NATS with testcontainers-go, coverage targets, and CI setup. Use whenever writing, fixing, reviewing, or running Go tests, adding coverage, debugging a flaky or racy test, or setting up Go test CI, even if the user just says "add a test" or "make sure this works".
metadata:
  origin: ECC
---

# Go Testing

How to write Go tests that are fast, deterministic, and catch real bugs. Code style and concurrency patterns live in the **golang-patterns** skill; this one covers verifying the code.

## Approach

- **Writing the test first is recommended**, especially for bug fixes (reproduce the bug as a failing test, then fix) and for pure domain logic. It forces you to design the API from the caller's side. It is not a ritual: for exploratory adapter code, writing the test right after is fine, as long as the change does not land untested.
- **Test behavior, not implementation.** Assert on outputs, returned errors, and observable side effects. Tests that pin internal call sequences break on every refactor and stop being trusted.
- **Pick the cheapest test that gives real confidence:**
  - Domain logic: unit tests with in-memory fakes. Milliseconds, run on every save.
  - Adapters (SQL, Redis, NATS): integration tests against the real thing in a container. Fakes cannot tell you whether your SQL is valid or your JetStream consumer config works.
  - HTTP layer: `httptest` through the real router.
- **Always run with `-race`** in CI and before finishing a change. A data race is a bug even when the test passes.

## Test package: `package foo` vs `package foo_test`

Both live in `foo/*_test.go`; choose per file.

- **`package foo_test` (external)** sees only exported API, like a real caller. Use it by default for testing a package's contract. It also avoids import cycles when the test needs a package that itself imports `foo`, and doubles as usage documentation (examples should be external).
- **`package foo` (internal)** sees unexported identifiers. It is idiomatic and the standard library does it extensively. Use it when an unexported function has non-trivial logic worth testing directly (a parser, a pricing rule, a state transition) and reaching it through the public API would need elaborate setup that obscures what is being tested.
- Do not export something only to test it. If you need a seam from an external test, put `export_test.go` in `package foo` with `var ParseHeader = parseHeader`; it is compiled only during tests.

## Table-driven tests

The default shape whenever a function has more than one interesting case. Name every case: the name appears in failures and in `-run` filters.

```go
func TestParsePrice(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    Money
        wantErr error
    }{
        {name: "integer", in: "12", want: Money{Cents: 1200}},
        {name: "two decimals", in: "12.34", want: Money{Cents: 1234}},
        {name: "negative", in: "-1", wantErr: ErrNegative},
        {name: "garbage", in: "abc", wantErr: ErrInvalidPrice},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParsePrice(tt.in)
            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("ParsePrice(%q) error = %v, want %v", tt.in, err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("ParsePrice(%q) = %+v, want %+v", tt.in, got, tt.want)
            }
        })
    }
}
```

- Compare errors with `errors.Is` against an expected sentinel rather than a `wantErr bool`. A bool passes when the function fails for the wrong reason.
- Failure messages follow `Func(input) = got, want want` so a failure is readable without opening the test.
- `t.Fatalf` when continuing makes no sense (setup failed, nil result); `t.Errorf` to report several mismatches in one run.
- For structs, slices, and maps, `github.com/google/go-cmp/cmp` (`cmp.Diff(want, got)`) gives readable diffs; `reflect.DeepEqual` works but only says "not equal". Avoid assertion frameworks that hide the comparison; if a project already uses testify, stay consistent with it.
- Since Go 1.22 each loop iteration has its own `tt`, so `tt := tt` is unnecessary. Remove it when you touch old tests.

## Parallel tests

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

Mark tests parallel when they share no mutable state: no package globals, no shared fake without locking, no fixed ports or file paths. `t.Setenv` and `t.Chdir` cannot be used in parallel tests (they panic), because env and working directory are process-wide. Run `go test -shuffle=on` occasionally to surface hidden order dependencies.

## Helpers and cleanup

```go
func newTestService(t *testing.T) (*order.Service, *fakeUserStore) {
    t.Helper() // failures point at the caller's line, not this one
    users := newFakeUserStore()
    svc := order.NewService(users, slog.New(slog.DiscardHandler))
    t.Cleanup(func() { /* release resources; runs after the test and its subtests */ })
    return svc, users
}
```

- `t.Context()` (Go 1.24) gives a context cancelled just before cleanup runs. Use it instead of `context.Background()` so goroutines started by the code under test stop with the test.
- `t.TempDir()` for files (removed automatically), `t.Setenv` for environment variables (restored automatically).
- `slog.DiscardHandler` (Go 1.24) silences logs in tests. To assert on logs, write to a `bytes.Buffer` with a JSON handler.
- Shared test helpers used by several packages go in an `internal/testutil` package (or `<pkg>test`, like `net/http/httptest`), never in production code paths.

## Fakes over mocks

Consumer-defined small interfaces (see **golang-patterns**) make hand-written fakes cheap. A fake is a working in-memory implementation, so tests read as behavior rather than call scripts.

```go
type fakeUserStore struct {
    mu    sync.Mutex
    users map[string]user.User
}

func newFakeUserStore(us ...user.User) *fakeUserStore {
    f := &fakeUserStore{users: make(map[string]user.User)}
    for _, u := range us {
        f.users[u.ID] = u
    }
    return f
}

func (f *fakeUserStore) GetUser(_ context.Context, id string) (user.User, error) {
    f.mu.Lock()
    defer f.mu.Unlock()
    u, ok := f.users[id]
    if !ok {
        return user.User{}, user.ErrNotFound
    }
    return u, nil
}
```

- To inject failures, add a field such as `err error` that the method returns when set.
- Reach for generated mocks (mockgen, moq) only for interfaces you do not own or when a test must verify an interaction itself (for example, "the message was published exactly once"). Over-mocking leads to tests that pass while the real integration is broken.
- Never fake the database to test SQL. Test repositories against a real Postgres (below).

## HTTP handlers

Test through the real `ServeMux` so method and wildcard patterns (`"GET /users/{id}"`) and `r.PathValue` work. Calling the handler function directly skips routing and leaves `PathValue` empty, unless you call `req.SetPathValue` yourself.

```go
func TestGetUser(t *testing.T) {
    store := newFakeUserStore(user.User{ID: "u1", Name: "Alice"})
    mux := httpapi.NewMux(store) // registers "GET /users/{id}"

    tests := []struct {
        name       string
        path       string
        wantStatus int
    }{
        {"found", "/users/u1", http.StatusOK},
        {"not found", "/users/nope", http.StatusNotFound},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest(http.MethodGet, tt.path, nil)
            rec := httptest.NewRecorder()
            mux.ServeHTTP(rec, req)
            if rec.Code != tt.wantStatus {
                t.Fatalf("GET %s status = %d, want %d; body: %s", tt.path, rec.Code, tt.wantStatus, rec.Body)
            }
        })
    }
}
```

Decode JSON responses into a struct and compare fields instead of comparing raw strings, which break on key order and whitespace. Use `httptest.NewServer` when testing an HTTP client against a fake upstream. What the status codes and payloads should be is decided by the **api-design** skill.

## Golden files

For large outputs (rendered templates, generated SQL, serialized payloads) store expected output in `testdata/<name>.golden` and regenerate with a flag. Review golden diffs in code review like any other change.

```go
var update = flag.Bool("update", false, "rewrite golden files")

func assertGolden(t *testing.T, name string, got []byte) {
    t.Helper()
    path := filepath.Join("testdata", name+".golden")
    if *update {
        if err := os.WriteFile(path, got, 0o644); err != nil {
            t.Fatalf("update golden: %v", err)
        }
    }
    want, err := os.ReadFile(path)
    if err != nil {
        t.Fatalf("read golden: %v", err)
    }
    if !bytes.Equal(got, want) {
        t.Errorf("%s mismatch (run with -update to accept):\n%s", name, cmp.Diff(string(want), string(got)))
    }
}
```

## Time and concurrency

- Never `time.Sleep` to wait for something. Sleeps make tests slow on fast machines and flaky on slow CI. Wait on a channel, poll a condition with a deadline, or control time.
- Inject time: take a `now func() time.Time` (or a small `Clock` interface) instead of calling `time.Now()` in domain code, and pass a fixed time in tests.
- Go 1.25+: `testing/synctest` runs code in a bubble with a fake clock. Inside `synctest.Test(t, func(t *testing.T) { ... })`, timers and `time.Sleep` advance instantly once every goroutine in the bubble is blocked, and `synctest.Wait()` waits until they are. Use it for timeouts, retries with backoff, and tickers without real waiting.
- Run concurrent code under `-race` and with `-count=N` to shake out flakiness. A test that fails 1 time in 50 is a bug in either the test or the code; fix it or delete it, never retry it in CI.

## Benchmarks

Use `b.Loop()` (Go 1.24). It runs setup outside the timed region automatically and keeps the compiler from optimizing away the call, so `b.ResetTimer()` and sink variables are unnecessary.

```go
func BenchmarkEncodeOrder(b *testing.B) {
    o := sampleOrder() // setup before the loop is not timed
    for b.Loop() {
        if _, err := encodeOrder(o); err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkSort(b *testing.B) {
    for _, n := range []int{100, 10_000} {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := randomInts(n)
            for b.Loop() {
                s := slices.Clone(data) // sort a fresh copy each iteration
                slices.Sort(s)
            }
        })
    }
}
```

Run with `go test -run=^$ -bench=. -benchmem -count=10 ./pkg > new.txt` and compare before/after with `benchstat old.txt new.txt` (`golang.org/x/perf/cmd/benchstat`). A single run is noise; do not claim a speedup without benchstat.

## Fuzzing

Fuzz anything that parses untrusted input: decoders, query/filter parsers, ID formats. Assert properties, not specific outputs.

```go
func FuzzParseFilter(f *testing.F) {
    f.Add("status eq 'active'")
    f.Add("")
    f.Add("age gt 30 and name eq 'x'")
    f.Fuzz(func(t *testing.T, in string) {
        flt, err := ParseFilter(in)
        if err != nil {
            return // rejecting input is fine; panicking or hanging is not
        }
        again, err := ParseFilter(flt.String())
        if err != nil {
            t.Fatalf("round trip of %q failed: %v", in, err)
        }
        if !flt.Equal(again) {
            t.Fatalf("round trip changed %q: %v vs %v", in, flt, again)
        }
    })
}
```

Run locally with `go test -run=^$ -fuzz=FuzzParseFilter -fuzztime=60s ./pkg`. Failing inputs are written to `testdata/fuzz/FuzzParseFilter/`; commit them so they run as regular regression cases in every `go test`.

## Integration tests

Run adapters against real Postgres, Redis, and NATS. Two options:

- **testcontainers-go** (recommended): each test package starts the containers it needs from Go code, so `go test` works with only Docker installed and nothing to start beforehand. Modules exist for Postgres, Redis, and NATS.
- **docker compose**: start the stack once (`docker compose up -d --wait`) and point tests at it via environment variables. Faster for repeated local runs; requires the stack to be up.

Keep integration tests out of the fast loop with a build tag (`//go:build integration`, run with `go test -tags=integration ./...`) or by skipping under `testing.Short()`. Isolate tests from each other: a fresh database per package and truncation or unique IDs per test, never a shared database that tests mutate in order.

Read `references/integration-tests.md` for testcontainers-go setup for Postgres (pgxpool plus migrations), Redis, and NATS JetStream, sharing a container across a package with `TestMain`, and per-test isolation strategies.

## Coverage

```bash
go test -race -coverprofile=coverage.out -covermode=atomic ./...
go tool cover -func=coverage.out | tail -1   # total
go tool cover -html=coverage.out             # find untested branches
```

Targets (statement coverage, as guidance for what deserves attention, not a goal in itself):

| Code | Target |
|------|--------|
| Domain logic (pricing, state machines, authorization rules) | 95%+, every branch and error path |
| Services, handlers, adapters | 80%+ |
| Overall module | 80%+ |
| `main`, wiring, generated code | Excluded |

- Coverage shows what was executed, not what was verified. A test without assertions raises coverage and proves nothing; review the assertions, not the percentage.
- Exclude generated code by filtering the profile before computing totals, for example `grep -v -E '(\.pb\.go|_mock\.go|/mocks/)' coverage.out > coverage.filtered.out`. Build tags cannot do this.
- Integration tests cover adapters; to count them, run with `-tags=integration` and use `-coverpkg=./...` so cross-package coverage is attributed.

## Commands

```bash
go test ./...                              # all tests
go test -race -shuffle=on ./...            # what CI should run
go test -run 'TestParsePrice/negative' ./internal/money   # one subtest
go test -short ./...                       # skip slow tests
go test -count=1 ./...                     # bypass the test cache
go test -count=50 -run TestFlaky ./pkg     # reproduce flakiness
go test -tags=integration ./...            # include integration tests
go test -run=^$ -bench=. -benchmem ./pkg   # benchmarks only
go test -timeout 2m ./...                  # fail hung tests instead of waiting 10m
```

## CI

```yaml
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-go@v7
      with:
        go-version-file: go.mod   # one source of truth for the Go version
    - name: Unit and integration tests
      run: go test -race -shuffle=on -tags=integration -coverprofile=coverage.out ./...
    - name: Coverage gate
      run: |
        total=$(go tool cover -func=coverage.out | awk '/^total:/ {sub("%","",$3); print $3}')
        echo "total coverage: ${total}%"
        awk -v t="$total" 'BEGIN { exit (t < 80) }'
```

GitHub-hosted Ubuntu runners have Docker, so testcontainers-go works there without extra setup. Lint and security scanning steps belong to the **golang-patterns** (golangci-lint) and **go-security** skills.

## Checklist

- Each case has a descriptive name; error cases use `errors.Is` against a specific error.
- Error paths are tested, not only the happy path.
- Helpers call `t.Helper()`; resources are released with `t.Cleanup`.
- Tests use `t.Context()`, no `time.Sleep`, no real network outside integration tests.
- Repositories are tested against a real database, not a fake.
- `go test -race ./...` passes, also with `-shuffle=on`.
- No `tt := tt`, no `for i := 0; i < b.N; i++`, no `interface{}`.
