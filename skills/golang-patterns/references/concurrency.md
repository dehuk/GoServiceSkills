# Concurrency patterns (long form)

Complete, leak-free versions of the patterns summarized in the golang-patterns skill. Each one states how its goroutines exit.

## Worker pool

Fixed number of workers reading from a jobs channel. The closer goroutine waits for workers and then closes `results`, so the caller can `range` over results while workers run. Calling `wg.Wait()` before returning would deadlock as soon as `results` fills up, because nobody reads it yet.

```go
// RunPool processes jobs with n workers. The caller must close jobs when done
// sending, and must drain the returned channel (or cancel ctx).
func RunPool[J, R any](ctx context.Context, jobs <-chan J, n int, fn func(context.Context, J) R) <-chan R {
    results := make(chan R)
    var wg sync.WaitGroup
    for range n {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs { // exits when jobs is closed
                select {
                case results <- fn(ctx, job):
                case <-ctx.Done(): // exits when the consumer gave up
                    return
                }
            }
        }()
    }
    go func() {
        wg.Wait()
        close(results) // only after every sender has stopped
    }()
    return results
}
```

The producer that feeds `jobs` must also `select` on `ctx.Done()` when sending, otherwise it blocks forever once workers exit on cancellation.

When each job can fail and the first failure should stop everything, use `errgroup` with `SetLimit(n)` instead; it is shorter and handles cancellation for you.

## Pipeline stage

Each stage owns its output channel, closes it when its input is exhausted, and stops on cancellation.

```go
func mapStage[T, U any](ctx context.Context, in <-chan T, f func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- f(v):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

## Background loop with clean stop

Long-lived workers (pollers, outbox relays, cache refreshers) expose `Run(ctx) error` and return when the context ends. The owner decides when to start and stop; nothing starts in a constructor.

```go
func (r *Refresher) Run(ctx context.Context) error {
    t := time.NewTicker(r.interval)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return nil // normal shutdown, not an error
        case <-t.C:
            if err := r.refresh(ctx); err != nil {
                r.log.WarnContext(ctx, "refresh failed", "err", err) // keep looping; next tick retries
            }
        }
    }
}
```

## Service run function with graceful shutdown

Runs an HTTP server and a background worker under one errgroup. A signal or a failure of either component cancels the group context, which triggers shutdown of the others.

```go
func run(ctx context.Context, logger *slog.Logger) error {
    ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer stop()

    cfg, err := loadConfig()
    if err != nil {
        return fmt.Errorf("load config: %w", err)
    }

    // ... build stores, services, handlers ...
    mux := http.NewServeMux()
    srv := &http.Server{
        Addr:              cfg.Addr,
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second, // without it, slow clients can hold connections open
    }
    refresher := NewRefresher(logger, time.Minute)

    g, gctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        logger.Info("http server listening", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            return fmt.Errorf("http server: %w", err)
        }
        return nil
    })

    g.Go(func() error { return refresher.Run(gctx) })

    g.Go(func() error {
        <-gctx.Done() // signal received or another component failed
        // The parent context is already cancelled, so give shutdown its own deadline.
        shutdownCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 25*time.Second)
        defer cancel()
        if err := srv.Shutdown(shutdownCtx); err != nil {
            return fmt.Errorf("http shutdown: %w", err)
        }
        return nil
    })

    return g.Wait()
}
```

Shutdown order matters in services with consumers and pools: stop accepting new work (HTTP server, message consumers) first, let in-flight work finish, then close outbound resources (database pool, broker connection) after `g.Wait()` returns, typically via `defer pool.Close()` placed right after creating them in `run`. Keep the total shutdown deadline below the orchestrator's kill timeout (Kubernetes default: 30s).

## Mutex-guarded state

Keep the mutex next to the fields it protects and never return internal maps or slices without copying, since the caller would then read them without the lock.

```go
type Registry struct {
    mu    sync.RWMutex
    items map[string]Item
}

func NewRegistry() *Registry { return &Registry{items: make(map[string]Item)} }

func (r *Registry) Put(id string, it Item) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.items[id] = it
}

func (r *Registry) Snapshot() map[string]Item {
    r.mu.RLock()
    defer r.mu.RUnlock()
    return maps.Clone(r.items)
}
```

Use `RWMutex` only when reads clearly dominate and critical sections are non-trivial; otherwise a plain `Mutex` is faster. Use `atomic.Int64` and friends for single counters.
