---
name: latency-critical-systems
description: Measure, profile, and reduce latency in Go services -- p50/p95/p99 and tail latency, per-hop timeout budgets, pprof CPU/heap/block/mutex profiles, execution traces, allocation reduction, GC tuning (GOGC, GOMEMLIMIT), lock contention, and pgx/Redis/NATS connection pool sizing, verified with benchmarks and benchstat. Use whenever a Go service is slow, has latency spikes or high p99, burns CPU or memory, needs a performance budget, or the user asks to optimize, profile, or benchmark Go code -- even if they only say "this endpoint is slow".
metadata:
  origin: ECC
---

# Latency-Critical Go Systems

Use this skill when response time, throughput, or data freshness matters. The goal is measured improvement on the right metric, not code that "looks faster".

## 1. Split the Metrics

Do not collapse everything into "fast". Decide which of these matters and measure it separately:

- p50, p95, p99 (and max) latency per endpoint / subject -- averages hide the tail users feel
- throughput (req/s, msg/s) at a stated concurrency
- freshness age of cached or streamed data
- queue depth and consumer lag (JetStream `NumPending`, `NumAckPending`)
- pool wait time and saturation (pgxpool, go-redis)
- CPU per request, allocations per request, GC pause and GC CPU fraction
- error and retry rate under load

Tail latency compounds: a request that fans out to 10 backends, each with a 1% chance of a slow response, is slow about 10% of the time. That is why p99 per hop matters more than p50.

## 2. Map the Hot Path and Its Budget

Write the path from event to visible result, and assign a budget to each hop:

```text
client -> LB -> HTTP handler (decode, auth) -> use case
       -> Redis GET (5ms) -> Postgres query (30ms) -> NATS publish (10ms)
       -> encode response
total budget: 100ms p99
```

Enforce budgets with context deadlines so a slow dependency fails fast instead of eating the whole request:

```go
func (s *Service) GetOrder(ctx context.Context, id string) (Order, error) {
	// Child deadline bounded by the caller's: whichever is sooner wins.
	dbCtx, cancel := context.WithTimeout(ctx, 30*time.Millisecond)
	defer cancel()

	o, err := s.repo.Get(dbCtx, id)
	if err != nil {
		return Order{}, fmt.Errorf("get order %s: %w", id, err)
	}
	return o, nil
}
```

Rules that follow from budgets:

- Set `http.Server` `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`; never run with zero values in production -- slow clients hold goroutines and connections.
- Set `http.Client.Timeout` or per-request contexts on every outbound call; `http.DefaultClient` has no timeout.
- Retries must fit inside the remaining budget (check `ctx.Deadline()`), use jittered backoff, and only target idempotent operations. Retry storms are a common cause of p99 collapse.
- Consider hedged requests only for idempotent reads with a clear tail problem, and cap the extra load.

## 3. Measure Before Changing Anything

### Profiles from a running service

Expose pprof on a separate, non-public listener:

```go
import (
	"net/http"
	_ "net/http/pprof" // registers /debug/pprof on http.DefaultServeMux
)

go func() {
	// Internal port only; pprof leaks internals and can be used for DoS.
	if err := http.ListenAndServe("127.0.0.1:6060", nil); err != nil {
		slog.Error("pprof server", "err", err)
	}
}()
```

Block and mutex profiles are off by default; enable them at startup when investigating contention (they add overhead, so keep rates modest):

```go
runtime.SetBlockProfileRate(10_000)  // sample blocking events >= ~10us
runtime.SetMutexProfileFraction(100) // sample 1 in 100 contention events
```

Collect under realistic load:

```bash
go tool pprof -http=:8081 'http://localhost:6060/debug/pprof/profile?seconds=30'  # CPU
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/heap                  # in-use memory
go tool pprof -sample_index=alloc_space -http=:8081 http://localhost:6060/debug/pprof/heap  # allocation churn
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/block                 # time blocked on chans/locks
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/mutex                 # lock contention
curl -o trace.out 'http://localhost:6060/debug/pprof/trace?seconds=5' && go tool trace trace.out
```

Which tool answers which question:

| Symptom | Look at |
|---|---|
| High CPU, slow everywhere | CPU profile (flame graph) |
| GC pressure, memory growth | heap `alloc_space` (churn) vs `inuse_space` (retention) |
| Low CPU but high latency | block / mutex profiles, then execution trace |
| Latency spikes, scheduler delays, GC pauses | `go tool trace` (goroutine analysis, GC events) |
| Goroutine growth | `/debug/pprof/goroutine?debug=1` |

CPU profiles show where CPU is spent, not where time is spent waiting. If p99 is high but CPU is low, the problem is waiting (pool exhaustion, locks, network, GC assist), and the CPU profile will mislead you.

### Benchmarks

Use `b.Loop()` (Go 1.24+): it keeps setup out of the timing and prevents the compiler from optimizing the body away.

```go
func BenchmarkEncodeOrder(b *testing.B) {
	o := sampleOrder()
	b.ReportAllocs()
	for b.Loop() {
		if _, err := json.Marshal(o); err != nil {
			b.Fatal(err)
		}
	}
}
```

Compare with statistics, not a single run:

```bash
go test -run='^$' -bench=EncodeOrder -benchmem -count=10 ./internal/codec > old.txt
# apply change
go test -run='^$' -bench=EncodeOrder -benchmem -count=10 ./internal/codec > new.txt
go run golang.org/x/perf/cmd/benchstat@latest old.txt new.txt
```

Only claim an improvement benchstat reports as significant. Run on a quiet machine; noisy laptops produce noisy numbers. Microbenchmarks prove a function got faster, not that p99 improved -- confirm with a load test (e.g. `vegeta`, `k6`, `ghz`) against the running service and compare percentiles.

## 4. Optimization Order

Work from the biggest structural wins to the smallest micro-optimizations:

1. **Remove round trips**: N+1 queries, sequential calls that could be one query or a batch (`pgx.Batch`, Redis pipelines).
2. **Fix queries**: `EXPLAIN (ANALYZE, BUFFERS)`, missing indexes, over-fetching columns (see **postgres-patterns**).
3. **Cache stable reads** with explicit TTL and freshness metadata; protect against stampedes (singleflight, see **redis-patterns**).
4. **Parallelize independent calls** with `errgroup.WithContext`, bounded with `SetLimit`, so one failure cancels the rest.
5. **Split hot and cold paths**: move non-critical work (audit logs, notifications) off the request path to NATS.
6. **Apply backpressure**: bounded channels/worker pools and JetStream `MaxAckPending` rather than unbounded queues that convert overload into latency.
7. **Reduce contention**: see section 6.
8. **Reduce allocations and GC**: see section 5. Do this last; it is rarely the first bottleneck in an I/O-bound service.

## 5. Allocations and GC

Allocations cost CPU twice: once to allocate, again to collect. Check `-benchmem` and the heap `alloc_space` profile first.

- Preallocate when the size is known: `make([]T, 0, n)`, `strings.Builder.Grow`, `maps` with size hint.
- Avoid `fmt.Sprintf` and `interface{}`/`any` boxing on hot paths; use `strconv.AppendInt` and similar.
- Reuse buffers per request scope rather than per call; stream JSON with `json.NewEncoder(w)` instead of marshal-then-write for large responses.
- Check escapes: `go build -gcflags=-m ./internal/... 2>&1 | grep 'escapes to heap'`.
- Avoid holding a small slice of a huge backing array (retains the whole array); copy with `slices.Clone` when retaining.

### sync.Pool caveats

`sync.Pool` helps only for frequently allocated, short-lived, similarly sized objects (buffers, encoders). Pitfalls:

- Always reset the object before `Put`, or data leaks between requests.
- Do not pool objects of wildly varying size: one huge buffer returned to the pool stays alive and inflates memory. Drop oversized buffers instead of returning them.
- Pools are cleared across GCs; they are a cache, not a free list. Never rely on an object being there.
- Store pointers (`*bytes.Buffer`), not values, to avoid an allocation on `Put`.

```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func render(w io.Writer, v any) error {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer func() {
		if buf.Cap() <= 64<<10 { // do not retain huge buffers
			bufPool.Put(buf)
		}
	}()
	if err := json.NewEncoder(buf).Encode(v); err != nil {
		return err
	}
	_, err := w.Write(buf.Bytes())
	return err
}
```

Prove the pool helps with a benchmark; it often does not.

### GC tuning

- `GOMEMLIMIT` (soft memory limit) is the main lever in containers: set it to roughly 80-90% of the container memory limit so the GC works harder near the limit instead of the process being OOM-killed. Go does not read cgroup limits automatically for this.
- `GOGC` (default 100) trades memory for CPU: raising it means fewer collections and more heap. With `GOMEMLIMIT` set, `GOGC=off` is possible for batch-like workloads but risks GC thrashing near the limit; prefer modest increases.
- Since Go 1.25 the runtime respects container CPU limits for `GOMAXPROCS`; on older versions set it explicitly (or use `go.uber.org/automaxprocs`) so the scheduler does not oversubscribe a throttled container, which shows up as p99 spikes.
- Verify with `GODEBUG=gctrace=1` or `runtime/metrics` (`/cpu/classes/gc/total:cpu-seconds`, `/sched/pauses/total/gc:seconds`), not by feel.

Tune GC only after reducing allocations; tuning hides allocation problems rather than fixing them.

## 6. Lock Contention

The mutex profile tells you which lock; then pick the fix:

- Shrink the critical section: never hold a lock across I/O, channel sends, or calls into unknown code.
- Use `sync.RWMutex` only when reads vastly dominate and critical sections are non-trivial; for tiny sections it is often slower than `sync.Mutex`.
- Shard hot maps by key hash (N independent mutex+map pairs).
- Use `atomic.Pointer[T]` for read-mostly config/snapshots: readers load, writers build a new copy and swap.
- `sync.Map` fits only append-mostly or disjoint-key workloads; it is not a general faster map.
- Avoid a single global rate limiter, logger, or metric with a lock on every request path at high QPS.

## 7. Connection Pools

Pool exhaustion shows up as latency, not errors: requests queue waiting for a connection.

**pgx (`pgxpool`)**
- Size `MaxConns` from the database side: total connections across all replicas must stay under Postgres `max_connections` (minus admin headroom). More connections than CPU cores on the DB rarely helps.
- Set `MinConns` to avoid cold-start connection latency, `MaxConnLifetime` (with jitter) and `MaxConnIdleTime` so connections rotate behind load balancers/PgBouncer.
- Monitor `pool.Stat()`: `EmptyAcquireCount` and `AcquireDuration` rising means the pool is the bottleneck.
- Keep transactions short; a transaction holds a connection for its full duration, including any network call you make inside it.
- Behind PgBouncer in transaction mode, the default statement cache breaks (prepared statements live on a server connection you may not get back). Use `QueryExecModeExec` or `QueryExecModeSimpleProtocol`, or enable PgBouncer's `max_prepared_statements` (1.21+).

**go-redis**
- Tune `PoolSize`, `MinIdleConns`, `PoolTimeout`, and `ReadTimeout`/`WriteTimeout` to fit the hop budget.
- Use pipelines for multi-key work; avoid `KEYS` and large `SMEMBERS`/`HGETALL` on hot paths (blocks Redis for everyone).
- Watch `client.PoolStats()` for `Timeouts` and `Misses`.

**NATS**
- One `*nats.Conn` per process is normal; it multiplexes. Do not create a connection per request.
- For JetStream publishes, `PublishAsync` with a bounded `MaxPending` gives much higher throughput than synchronous `Publish`; wait on the futures to detect failures.
- Consumers: tune `MaxAckPending` and pull batch size; slow handlers plus high `MaxAckPending` produce redeliveries via `AckWait`, which look like latency spikes (see **nats-jetstream-patterns**).

## 8. Verification

Before calling a path fast:

- Load test at expected and 2x peak concurrency; record p50/p95/p99 and error rate before and after.
- Check pool stats, consumer lag, and GC metrics during the test.
- Confirm freshness timestamps on cached/streamed data under load.
- Re-profile after the change; the bottleneck usually moves.
- Run `go test -race ./...`; many "optimizations" introduce races.

## Guardrails

- Do not drop validation, authorization, or error handling to win latency.
- Do not hide stale data behind fast cache hits; expose freshness.
- Do not claim improvements without before/after numbers from benchstat or a load test.
- Do not expose pprof on a public port.
- Do not run destructive migrations or customer-impacting load tests without explicit approval.
- Keep secrets and personal data out of profiles, traces, and benchmark artifacts.
