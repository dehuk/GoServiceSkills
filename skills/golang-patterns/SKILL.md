---
name: golang-patterns
description: Idiomatic, modern Go (1.24+) for backend services - package layout and naming, interface design, constructors and dependency injection, functional options, error-wrapping mechanics, goroutine/channel/errgroup patterns that do not leak or deadlock, graceful shutdown, slog, performance idioms, and golangci-lint v2 tooling. Use whenever writing, reviewing, or refactoring Go code, designing a Go package or service skeleton, or choosing between Go idioms, even if the user only asks for "a quick Go function" and never mentions patterns or style.
metadata:
  origin: ECC
---

# Go Patterns

Defaults for writing Go services that stay boring, predictable, and easy to review. Target Go 1.24+ and write modern Go: the older idioms that still circulate (`interface{}`, `tt := tt`, `for i := 0; i < b.N; i++`) are noise today and signal outdated code to reviewers.

Related skills, used instead of restating their content here:
- **golang-testing** - tests, benchmarks, fuzzing, coverage targets.
- **error-handling** - cross-cutting error strategy (what to expose to clients, logging policy, error codes). This skill only covers Go error mechanics.
- **go-security** - input validation, secrets, crypto, gosec/govulncheck.
- **api-design** - HTTP routes, status codes, payloads.
- **postgres-patterns**, **redis-patterns**, **nats-jetstream-patterns** - data stores and messaging.
- **hexagonal-architecture** - how to split domain, ports, and adapters across packages.

## Core principles

- **Clear over clever.** Early returns, the happy path at the left margin, no nested closures to save three lines. Code is read far more often than written.
- **Make the zero value useful.** `sync.Mutex`, `bytes.Buffer`, and `strings.Builder` work without initialization; design your own types the same way. When a type cannot be valid at zero (it needs a nil-unsafe map or a dependency), give it a constructor and keep fields unexported so nobody builds a half-initialized value.
- **Accept interfaces, return concrete types.** Callers get the full API of what you return, and you can add methods without breaking anyone. Return an interface only when you genuinely have several implementations behind a factory, or to hide an unexported type.
- **Pass `context.Context` as the first parameter** of anything that does I/O, blocks, or spawns work. Never store it in a struct: a context is scoped to one call, and a stored one outlives its cancellation.
- **No package-level mutable state.** Globals initialized in `init()` make tests order-dependent and hide dependencies. Wire everything in `main` (or a `run` function) and pass it down.
- **gofmt/goimports are non-negotiable.** Formatting is never a review topic.

## Packages and layout

```text
myservice/
├── cmd/myservice/main.go   # flag/env parsing, wiring, calls run(ctx)
├── internal/               # everything not meant for other modules
│   ├── user/               # one package per domain concept
│   ├── order/
│   ├── postgres/           # adapters named after the technology they wrap
│   └── httpapi/            # transport layer
├── migrations/
├── testdata/
└── go.mod
```

- Put code under `internal/` by default. The compiler then guarantees nobody outside the module depends on it, so you can refactor freely. Use `pkg/` only for code you deliberately publish, which a service rarely does.
- Organize by domain (`user`, `order`), not by kind (`models`, `services`, `utils`). Kind-based packages grow into import cycles and god packages.
- Package names: short, lowercase, single word, no underscores or mixedCaps. The name is part of every call site, so avoid stutter: `user.Service`, not `user.UserService`.
- Do not name a package after a standard library package (`http`, `context`, `errors`, `log`, `sql`). Every importer then has to alias one of the two, and readers misread call sites. Use `httpapi`, `apperr`, `postgres` instead.
- Avoid `util`, `common`, `helpers`, `base`. They say nothing about content and attract unrelated code.

## Interfaces

- **Define interfaces in the consumer package**, sized to what the consumer actually calls. The provider returns a concrete type and does not know the interface exists. This keeps interfaces small (1-3 methods is typical) and makes fakes trivial in tests.

```go
package order

// UserGetter is all the order service needs from the user store.
type UserGetter interface {
    GetUser(ctx context.Context, id string) (user.User, error)
}

type Service struct {
    users UserGetter
    log   *slog.Logger
}

func NewService(users UserGetter, log *slog.Logger) *Service {
    return &Service{users: users, log: log}
}
```

- Do not create an interface "for mocking" next to its only implementation in the provider package. That couples every consumer to the full method set.
- Use type assertions for optional behavior (`if f, ok := w.(http.Flusher); ok { f.Flush() }`) instead of widening an interface.
- Add a compile-time check when a type must satisfy an interface it is not otherwise checked against: `var _ order.UserGetter = (*postgres.UserStore)(nil)`.

## Constructors and options

Use plain constructor parameters for required dependencies. Use functional options only for optional settings with sensible defaults; they keep the constructor signature stable as options grow.

```go
type Client struct {
    baseURL string
    timeout time.Duration
    log     *slog.Logger
}

type Option func(*Client)

func WithTimeout(d time.Duration) Option { return func(c *Client) { c.timeout = d } }
func WithLogger(l *slog.Logger) Option   { return func(c *Client) { c.log = l } }

func NewClient(baseURL string, opts ...Option) *Client {
    c := &Client{baseURL: baseURL, timeout: 10 * time.Second, log: slog.Default()}
    for _, opt := range opts {
        opt(c)
    }
    return c
}
```

If an option can be invalid, make `Option` return an error (`type Option func(*Client) error`) and fail in the constructor rather than at first use. For more than a handful of settings, a plain `Config` struct is often clearer than options.

Embedding promotes every method of the embedded type into your public API. Embed only when you want that; otherwise use a named field.

## Error mechanics

Strategy (what to log, what reaches clients, error codes) belongs to the **error-handling** skill. The Go mechanics:

- Wrap with context using `%w`: `return fmt.Errorf("load config %s: %w", path, err)`. Describe what this function was doing, lowercase, no "failed to"/"error:" prefix; the chain then reads `start server: load config app.yaml: open app.yaml: no such file`.
- Check with `errors.Is` (sentinels) and `errors.As` (types), never `==` or string matching, because wrapping breaks both.
- Sentinels (`var ErrNotFound = errors.New("user: not found")`) for conditions callers branch on; error types (`*ValidationError`) when callers need fields. Exported sentinels and types are API: changing them breaks callers.
- Use `%v` instead of `%w` when you deliberately do not want callers to depend on the underlying error (for example, a driver error crossing an adapter boundary).
- Handle each error once: either return it (wrapped) or log it and stop. Logging and returning produces duplicate log lines for one failure.
- `errors.Join` combines independent failures (cleanup, validation of several fields).
- Never discard an error silently. If ignoring is correct, write `_ = f()` with a comment saying why.
- `Close` on something you wrote to can fail and lose data. Do not drop that error:

```go
func writeReport(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create report: %w", err)
    }
    defer func() { err = errors.Join(err, f.Close()) }()

    if _, err := f.Write(data); err != nil {
        return fmt.Errorf("write report: %w", err)
    }
    return nil
}
```

- `panic` only for programmer errors that make continuing meaningless (impossible states, failed `Must*` setup in `main`). Never for control flow or expected failures.

## Concurrency

Rules that prevent most goroutine bugs:

1. **Every goroutine needs a known exit.** Before writing `go`, decide what makes it stop: input channel closed, context cancelled, or work done. A goroutine blocked forever on a send is a memory leak that never shows up in tests.
2. **The sender closes the channel**, and only when nothing else will send. Closing from the receiver side panics the sender.
3. **`wg.Wait()` and `close(results)` go in their own goroutine** when the caller consumes results, otherwise workers block on a full channel while the caller blocks in `Wait`.
4. **Prefer `errgroup`** (`golang.org/x/sync/errgroup`) over hand-rolled `WaitGroup` + error channel. It propagates the first error, cancels siblings via context, and bounds concurrency with `SetLimit`.
5. **Run tests with `-race`.** Data races are undefined behavior, not just flaky output.

Bounded fan-out with errgroup (loop variables are per-iteration since Go 1.22, so no copying is needed):

```go
func FetchAll(ctx context.Context, urls []string) ([][]byte, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(8) // bound concurrency; unbounded fan-out exhausts sockets and memory

    results := make([][]byte, len(urls)) // each goroutine writes its own index: no lock needed
    for i, url := range urls {
        g.Go(func() error {
            data, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            results[i] = data
            return nil
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

Returning a result asynchronously without leaking: send the error too (otherwise the receiver waits forever on failure) and buffer the channel so the send never blocks if the receiver has gone away.

```go
type fetchResult struct {
    data []byte
    err  error
}

func fetchAsync(ctx context.Context, url string) <-chan fetchResult {
    ch := make(chan fetchResult, 1) // buffered: goroutine exits even if nobody reads
    go func() {
        data, err := fetch(ctx, url) // fetch must honor ctx so cancellation ends it promptly
        ch <- fetchResult{data: data, err: err}
    }()
    return ch
}
```

Other rules:
- Protect shared state with a `sync.Mutex` next to the fields it guards; use channels to transfer ownership or signal, not as a lock replacement.
- `time.After` in a loop allocates a timer per iteration; use `time.NewTimer`/`Ticker` and stop it.
- `sync.OnceValue`/`OnceValues` replace hand-written lazy-init with `sync.Once` plus fields. Go 1.25 adds `wg.Go(func())`, which replaces the `Add(1)`/`defer Done()` pair.

Read `references/concurrency.md` when you need a worker pool, a pipeline, a background loop with clean stop, or a full graceful-shutdown `main`.

## Service skeleton

Keep `main` tiny: it calls `run`, which returns an error. `run` is testable and every deferred cleanup executes (unlike `log.Fatal`/`os.Exit` deep in the code, which skip defers).

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    if err := run(context.Background(), logger); err != nil {
        logger.Error("service stopped", "err", err)
        os.Exit(1) // no defers in main, so nothing is skipped
    }
}
```

`run` installs the signal handling (`signal.NotifyContext`), loads config, builds dependencies, starts servers, and blocks until shutdown. The full version with an HTTP server and shutdown sequence is in `references/concurrency.md`.

## Logging

- Use `log/slog`. Create one logger in `main`, inject it, add scope with `logger.With("component", "orders")`. Do not use the global `log` package or `slog.Default()` deep in library code.
- Log with key-value attributes, not formatted strings: `log.InfoContext(ctx, "order created", "order_id", id)`. Structured fields are searchable; interpolated strings are not.
- Use the `*Context` variants so handlers can pull request-scoped values (trace IDs) from the context.
- Never log secrets, tokens, or full request bodies (see **go-security**).

## Modern standard library

Prefer these over hand-written helpers or old third-party packages:

| Need | Use |
|------|-----|
| Generic slice ops | `slices.Contains`, `slices.Index`, `slices.SortFunc`, `slices.Collect` |
| Map keys/values | `maps.Keys`/`maps.Values` (iterators; wrap in `slices.Collect` or `slices.Sorted`) |
| Loops over a count | `for i := range n` |
| Min/max | built-in `min`, `max` |
| Clone | `slices.Clone`, `maps.Clone`, `bytes.Clone`, `strings.Clone` |
| HTTP routing | `mux.HandleFunc("GET /users/{id}", h)` and `r.PathValue("id")` (no router dependency needed) |
| Detached cleanup work | `context.WithoutCancel(ctx)` keeps values but not cancellation |
| JSON zero omission | `json:",omitzero"` (1.24) instead of pointer fields for `time.Time` and structs |
| Line/field iteration | `strings.Lines`, `strings.SplitSeq`, `strings.FieldsSeq` (1.24) |
| Sandboxed file access | `os.Root` (1.24) for paths derived from input |

## Performance idioms

Measure first (see **golang-testing** for benchmarks; `go test -cpuprofile`/`-memprofile`, `pprof`). Then:

- Preallocate when the size is known: `make([]T, 0, len(in))`.
- Build strings with `strings.Builder` or `strings.Join`, not `+=` in a loop.
- Pass large structs by pointer; small ones (a few words) by value is fine and avoids heap escapes.
- `sync.Pool` only for hot paths with measured allocation pressure. Never return memory that still belongs to a pooled object: once it is `Put` back, another goroutine reuses and overwrites it.

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func encodeOrder(o Order) ([]byte, error) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer func() {
        if buf.Cap() <= 64<<10 { // do not pin huge buffers in the pool
            bufPool.Put(buf)
        }
    }()

    if err := json.NewEncoder(buf).Encode(o); err != nil {
        return nil, fmt.Errorf("encode order: %w", err)
    }
    return bytes.Clone(buf.Bytes()), nil // copy: buf is reused after Put
}
```

Where possible, write straight into an `io.Writer` instead of returning bytes; then no copy is needed.

## Tooling and edit loop

After editing Go files, run at least:

```bash
gofmt -l . && goimports -w .        # or rely on golangci-lint formatters
go vet ./...
golangci-lint run ./...
go test -race ./...
```

Run these on the changed packages during iteration and on `./...` before finishing. Use golangci-lint v2 (config starts with `version: "2"`); a recommended config and the full command list are in `references/tooling.md`. Read it when creating or fixing `.golangci.yml`, a Makefile, or CI lint steps.

## Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|--------------|--------------|---------|
| `interface{}` | Outdated spelling | `any` |
| Naked returns in long functions | Reader cannot see what is returned | Explicit returns; named results only for docs or deferred error capture |
| Mixed value and pointer receivers on one type | Method sets differ, confusing interface satisfaction; copies mutex fields | Pointer receivers for all methods if any needs one |
| `ctx` stored in a struct | Outlives its cancellation scope | First parameter of each call |
| Goroutine started in a constructor or `init` | Hidden lifecycle, no way to stop it | Explicit `Run(ctx)`/`Start`/`Close` |
| `log.Fatal` outside `main` | Skips defers, untestable | Return the error |
| Ignoring errors from `Close` on writers | Silent data loss | `errors.Join` in a deferred func |
| Shadowing `err` in nested scopes then checking the outer one | Real error never checked | Keep one `err` per scope; `govet shadow` catches it |
| Copying values that contain `sync.Mutex` | Two independent locks, race | Pointer receivers; `go vet copylocks` |
| Generic `util` packages and premature generics | Unfocused code | Concrete code first; generics when two real call sites need it |
