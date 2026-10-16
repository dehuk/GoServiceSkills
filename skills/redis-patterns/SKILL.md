---
name: redis-patterns
description: Redis patterns for Go services using go-redis/v9 - client and pool configuration, cache-aside with TTL jitter and singleflight stampede protection, invalidation and key versioning, serialization, redis.Nil handling, pipelines and transactions (TxPipelined, WATCH), Lua scripts via redis.NewScript, distributed locks with tokens, fixed and sliding window rate limiting, Pub/Sub vs Streams vs NATS, and testing with miniredis or testcontainers. Use whenever Go code touches Redis or the user wants caching, rate limiting, a distributed lock, session or token storage, or deduplication, even if they do not mention go-redis by name.
metadata:
  origin: ECC
---

# Redis Patterns for Go

Redis in a Go service is usually a cache, a rate-limit counter store, or a coordination aid. Postgres stays the source of truth and NATS JetStream carries durable messages. Most Redis bugs come from forgetting that Redis can be empty, stale, or unavailable at any moment, so design every use so the service is still correct when it is.

## When to Use

- Adding a cache in front of Postgres or a slow downstream API
- Rate limiting per user, API key or IP across replicas
- Short-lived shared state: sessions, one-time tokens, idempotency keys, deduplication windows
- Mutual exclusion for efficiency (avoid duplicate work), not for correctness
- Reviewing go-redis client config, key design or error handling

Do not use Redis as the only copy of data you cannot recompute, or as a message broker for domain events (use NATS JetStream; see the nats-jetstream-patterns skill).

## Client Setup

```go
func NewRedis(ctx context.Context, cfg Config) (*redis.Client, error) {
	rdb := redis.NewClient(&redis.Options{
		Addr:     cfg.Addr,
		Username: cfg.Username,
		Password: cfg.Password,
		DB:       0,

		// Pool: default is 10 per CPU (GOMAXPROCS). Size for peak concurrency of Redis
		// calls, not for request concurrency; most calls take well under a millisecond.
		PoolSize:        cfg.PoolSize,
		MinIdleConns:    2,
		ConnMaxIdleTime: 5 * time.Minute,

		DialTimeout:  2 * time.Second,
		ReadTimeout:  500 * time.Millisecond, // a cache that answers slowly is worse than a miss
		WriteTimeout: 500 * time.Millisecond,
		PoolTimeout:  1 * time.Second, // waiting for a free conn; fail fast instead of queueing

		// Honour ctx deadlines in addition to Read/WriteTimeout.
		ContextTimeoutEnabled: true,
		MaxRetries:            2,
	})

	pingCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()
	if err := rdb.Ping(pingCtx).Err(); err != nil {
		_ = rdb.Close() // nothing to report: the Ping error is the useful one
		return nil, fmt.Errorf("redis ping %s: %w", cfg.Addr, err)
	}
	return rdb, nil
}
```

- Create one client per process at startup and share it; it is safe for concurrent use. Call `rdb.Close()` during graceful shutdown after the HTTP server and consumers have stopped.
- Pass the request `ctx` to every command so cancelled requests stop waiting on Redis.
- Use `redis.ParseURL("rediss://user:pass@host:6379/0")` when config comes as a URL; `rediss://` enables TLS for managed Redis.
- Depend on `redis.UniversalClient` (or a narrow interface of your own) rather than `*redis.Client` in code that might move to Cluster or Sentinel; `redis.NewUniversalClient` picks the right implementation from options.
- go-redis v9 speaks RESP3 by default. Set `Protocol: 2` only if a proxy or old server rejects `HELLO`.
- Add tracing and metrics with `github.com/redis/go-redis/extra/redisotel/v9` (`redisotel.InstrumentTracing(rdb)`, `redisotel.InstrumentMetrics(rdb)`), and export `rdb.PoolStats()` (hits, misses, timeouts) so pool exhaustion is visible.
- Report Redis in `/readyz` only if the service cannot work without it. A pure cache should degrade, not make the pod unready.

## redis.Nil Is Not an Error

A missing key returns `redis.Nil`. Treat it as a normal outcome and check it with `errors.Is`, never by comparing strings.

```go
val, err := rdb.Get(ctx, key).Result()
switch {
case errors.Is(err, redis.Nil):
	// miss: load from the source of truth
case err != nil:
	// Redis down or timeout: decide explicitly (degrade for a cache, fail for a lock)
default:
	_ = val // hit
}
```

`redis.Nil` also comes from `HGET` on a missing field, `LPOP`/`BLPOP` on empty lists, `ZSCORE` for a missing member, and from `SetArgs` with `Mode: "NX"` when the key exists (whereas `SetNX` reports that as `false` with a nil error). `Pipelined` and `TxPipelined` return the first failed command's error, so a single missed `GET` in a pipeline makes the whole call return `redis.Nil`; inspect each `Cmd` instead (see Pipelines below).

## Key Naming and Versioning

Use `<service>:<schema-version>:<entity>:<id>[:<field>]`, for example `orders:v3:order:7f3c...` or `orders:v1:ratelimit:user:42`.

- **Service prefix**: several services often share one Redis; the prefix avoids collisions and lets you find a service's keys with `SCAN MATCH orders:*`.
- **Schema version**: bump it when the cached struct changes incompatibly. New code reads `v4:` keys, old `v3:` entries are never read and expire on their own. No flush, no deploy-ordering problems, and old and new replicas can run side by side during a rollout.
- Build keys in one place (`func orderKey(id uuid.UUID) string`) so a version bump touches one line.
- In Redis Cluster, multi-key commands and Lua scripts need all keys in one hash slot. Wrap the shared part in braces: `orders:v3:{user:42}:cart` and `orders:v3:{user:42}:cart:items`.
- Never use `KEYS pattern` in production; it blocks the server. Iterate with `rdb.Scan(ctx, 0, "orders:v2:*", 500).Iterator()` and check `iter.Err()` after the loop.

## Serialization

- JSON (`encoding/json`) is the default: debuggable with `redis-cli`, tolerant of added fields. Cache a dedicated struct with explicit tags, not your domain entity or a pgx row type; renaming a domain field should not silently break the cache.
- Use protobuf or msgpack only for large or hot values where JSON cost shows up in profiles.
- Avoid `encoding/gob` across deploys; type changes break decoding.
- If decoding a cached value fails, treat it as a miss, log it, and overwrite the entry. Never return an error to the user because of a bad cache entry.
- Keep values small (well under 100 KB). Large values block Redis's single thread and saturate the network. Cache an ID list and fetch the items, or store blobs in object storage.
- Store simple flat objects that you update field-by-field as hashes (`HSET`, `HGETALL`); use a JSON string when you always read and write the whole value.

## Cache-Aside with Jitter and Stampede Protection

Read the cache, on miss load from the source and populate. Add random jitter to TTLs so keys written together (after a deploy or a bulk load) do not expire together, and collapse concurrent misses for the same key with `golang.org/x/sync/singleflight` so one hot key expiring does not send hundreds of identical queries to Postgres.

```go
type Cache struct {
	rdb         redis.UniversalClient
	group       singleflight.Group
	ttl         time.Duration
	jitter      time.Duration // e.g. 10% of ttl
	loadTimeout time.Duration
	log         *slog.Logger
}

func (c *Cache) ttlWithJitter() time.Duration {
	if c.jitter <= 0 {
		return c.ttl
	}
	return c.ttl + rand.N(c.jitter) // math/rand/v2
}

// GetOrLoad returns the cached value for key or loads, caches and returns it.
// Redis failures degrade to calling load; they are logged, not returned.
func GetOrLoad[T any](ctx context.Context, c *Cache, key string, load func(context.Context) (T, error)) (T, error) {
	b, err := c.rdb.Get(ctx, key).Bytes()
	switch {
	case err == nil:
		var v T
		if err := json.Unmarshal(b, &v); err == nil {
			return v, nil
		}
		c.log.WarnContext(ctx, "cache decode failed, reloading", "key", key)
	case errors.Is(err, redis.Nil):
		// miss
	default:
		c.log.WarnContext(ctx, "cache get failed", "key", key, "err", err)
	}

	res, err, _ := c.group.Do(key, func() (any, error) {
		// Detach from the first caller's cancellation: other callers share this result,
		// and one client disconnecting must not fail them all.
		lctx, cancel := context.WithTimeout(context.WithoutCancel(ctx), c.loadTimeout)
		defer cancel()

		v, err := load(lctx)
		if err != nil {
			return nil, err
		}
		if b, err := json.Marshal(v); err != nil {
			c.log.WarnContext(lctx, "cache encode failed", "key", key, "err", err)
		} else if err := c.rdb.Set(lctx, key, b, c.ttlWithJitter()).Err(); err != nil {
			c.log.WarnContext(lctx, "cache set failed", "key", key, "err", err)
		}
		return v, nil
	})
	if err != nil {
		var zero T
		return zero, err
	}
	v, _ := res.(T) // comma-ok: a nil interface T would otherwise panic
	return v, nil
}
```

Notes:

- `singleflight` deduplicates within one process. With N replicas a cold key still causes up to N loads, which is usually fine. For very expensive loads, add probabilistic early refresh (refresh in the background when remaining TTL is below a random threshold) or a short distributed lock around the load.
- Do not cache errors from `load`. Do cache "not found" for a short TTL (a sentinel value such as `{"missing":true}` for 30-60 s) when lookups for nonexistent IDs are common, so they do not all hit Postgres.
- Choose the TTL from how stale the data may be, not from how often it changes. A TTL is the upper bound on staleness when invalidation fails.

## Invalidation

| Strategy | How | Use when |
|---|---|---|
| TTL only | Rely on expiry | Staleness up to the TTL is acceptable (reference data, aggregates). |
| Delete on write | After the DB transaction commits, `DEL` the key | Default for entity caches. Delete rather than overwrite: a concurrent reader may write an older value after your update, and a delete limits that window to one read. |
| Event-driven | Publish a change event (transactional outbox to NATS), every service holding a copy deletes its keys | The data is cached by services that do not own it. |
| Version bump | Change the key version or a per-entity version counter included in the key | Bulk invalidation (schema change, tenant-wide reset) without scanning. |
| Tag sets | `SADD` keys to a tag set in the same `TxPipelined` as the `SET`; on invalidate, read the set and `DEL` members | Invalidating groups (all products of a category). Give the tag set a TTL at least as long as its members'. |

Always invalidate after the commit, never before or inside the transaction: if the transaction rolls back or another reader repopulates before commit, the cache holds data that never existed or is already stale. If invalidation itself fails, log it and rely on the TTL; do not fail the write that already committed.

## Pipelines and Transactions

```go
// Pipeline: one round trip, not atomic. Good for batching independent reads/writes.
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	for _, id := range ids {
		pipe.Get(ctx, orderKey(id))
	}
	return nil
})
if err != nil && !errors.Is(err, redis.Nil) { // redis.Nil = at least one miss, not a failure
	return nil, fmt.Errorf("pipeline get: %w", err)
}
for i, cmd := range cmds {
	b, err := cmd.(*redis.StringCmd).Bytes()
	if errors.Is(err, redis.Nil) {
		continue // miss for ids[i]
	}
	if err != nil {
		return nil, fmt.Errorf("get %s: %w", ids[i], err)
	}
	_ = b // decode
}
```

For plain multi-key reads `MGET` is simpler (missing keys come back as `nil` entries).

```go
// MULTI/EXEC: all commands run without interleaving. There is no rollback: if one
// command fails at runtime, the others still apply.
_, err := rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
	pipe.Set(ctx, key, payload, ttl)
	pipe.SAdd(ctx, tagKey, key)
	pipe.Expire(ctx, tagKey, ttl)
	return nil
})
```

`WATCH` gives optimistic concurrency for read-modify-write. Retry on `redis.TxFailedErr`, with a bound:

```go
func incrementIfBelow(ctx context.Context, rdb *redis.Client, key string, limit int) error {
	const maxRetries = 5
	for range maxRetries {
		err := rdb.Watch(ctx, func(tx *redis.Tx) error {
			n, err := tx.Get(ctx, key).Int()
			if err != nil && !errors.Is(err, redis.Nil) {
				return err
			}
			if n >= limit {
				return ErrLimitReached
			}
			_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
				pipe.Set(ctx, key, n+1, 0)
				return nil
			})
			return err
		}, key)
		if !errors.Is(err, redis.TxFailedErr) {
			return err // nil, ErrLimitReached, or a real error
		}
	}
	return errors.New("increment: too much contention")
}
```

When logic depends on values read inside the operation, a Lua script is usually simpler and faster than `WATCH`: it runs atomically on the server with no retries.

## Lua Scripts

Define scripts once as package-level values with `redis.NewScript`. `Run` uses `EVALSHA` and falls back to `EVAL` on `NOSCRIPT`, so you never manage script loading.

```go
var incrWithTTL = redis.NewScript(`
local n = redis.call('INCR', KEYS[1])
if n == 1 then
  redis.call('PEXPIRE', KEYS[1], ARGV[1])
end
return n
`)

n, err := incrWithTTL.Run(ctx, rdb, []string{key}, window.Milliseconds()).Int64()
```

- Pass every key through `KEYS`, never build key names inside the script; Cluster routing and ACLs depend on it.
- Scripts block Redis while running. Keep them O(small): no loops over unbounded sets.
- Use `redis.call('TIME')` inside the script for time-based logic so all replicas of your service share Redis's clock rather than their own.

## Distributed Locks and Rate Limiting

Both are easy to get subtly wrong. Read `references/locks-and-rate-limiting.md` before implementing either; it contains the full Go code (token-based `SET NX PX` lock with Lua compare-and-delete and extend, fixed window and sliding window limiters) and the caveats. In short:

- A Redis lock is a lease, not a guarantee. GC pauses, network delays and failover can let two holders overlap. Use it to avoid duplicate work; protect correctness with database constraints (unique indexes, conditional `UPDATE ... WHERE version = $n`, `SELECT ... FOR UPDATE`, `pg_advisory_xact_lock`).
- Rate limiters must be atomic (Lua or `INCR` with expiry in one script), keyed per subject, and need an explicit fail-open or fail-closed decision for when Redis is down.

## Pub/Sub vs Streams vs NATS JetStream

| | Redis Pub/Sub | Redis Streams | NATS JetStream |
|---|---|---|---|
| Delivery | At most once; offline subscribers miss messages | At least once with consumer groups and `XACK` | At least once, redelivery with backoff, exactly-once publish dedup |
| Replay | No | By ID range | By sequence or time, per consumer |
| Retention | None | `MAXLEN`/`MINID` trimming, memory-bound | Limits, interest or work-queue policies, file storage |
| Ops cost | None if Redis exists | Pending-entry recovery (`XAUTOCLAIM`) is on you | Separate system, but built for this |

Recommendation: use NATS JetStream for domain events, commands and anything that must not be lost (see the nats-jetstream-patterns skill). Redis Pub/Sub is fine for ephemeral fan-out that can be missed safely, such as "drop your local in-memory cache for key X" broadcasts. Reach for Redis Streams only when Redis is the only infrastructure available.

If you do use Pub/Sub, `pubsub.Channel()` buffers messages and drops them if your handler is slow; keep handlers fast and close the subscription on shutdown:

```go
pubsub := rdb.Subscribe(ctx, "orders:v1:cache-invalidate")
defer pubsub.Close()
for msg := range pubsub.Channel() { // closed when pubsub is closed
	localCache.Delete(msg.Payload)
}
```

## Data Structures, TTLs and Eviction

| Use case | Structure | Note |
|---|---|---|
| Cached object | String (JSON) | `SET key val PX ttl` |
| Session / token | String or Hash | TTL equals session lifetime; delete on logout |
| Idempotency key / dedup | String with `SET NX PX` | TTL covers the client retry window |
| Counter, fixed window limit | String with `INCR` | Expiry set atomically with the first `INCR` |
| Sliding window limit, leaderboard | Sorted set | Trim old members in the same script |
| Unique count (approximate) | HyperLogLog | 12 KB per key regardless of cardinality |

Give every key a TTL unless it is truly permanent configuration; keys without a TTL accumulate until memory runs out. For a cache-only instance set `maxmemory` and `maxmemory-policy allkeys-lfu` (or `allkeys-lru`). If the same instance holds locks, rate-limit counters or sessions, eviction can silently drop them; use a separate instance with `noeviction`, or `volatile-*` policies so only TTL'd cache keys are evicted.

## Testing

- **Unit tests**: put Redis behind a small port in your domain (`type OrderCache interface { Get(...); Set(...) }`) and fake that, rather than mocking go-redis calls. The hexagonal-architecture skill covers where that port lives.
- **Adapter tests with miniredis** (`github.com/alicebob/miniredis/v2`): in-process, fast, supports most commands and Lua. `mr := miniredis.RunT(t)` then `redis.NewClient(&redis.Options{Addr: mr.Addr()})`. Use `mr.FastForward(d)` to test TTL expiry without sleeping. It does not implement eviction, cluster behavior or every command option, so it is not a substitute for a real server on tricky scripts.
- **Integration tests with testcontainers-go** (`github.com/testcontainers/testcontainers-go/modules/redis`) against the same Redis version as production, for Lua scripts, locks, rate limiters and pool/timeout behavior. See the golang-testing skill for build tags and container reuse.
- Test the degraded paths: cache returns an error, value fails to decode, `load` fails, Redis times out. These are where production incidents come from.

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Treating `redis.Nil` as an error, or `err == redis.Nil` on wrapped errors | Misses reported as failures, or wrapped Nil missed | `errors.Is(err, redis.Nil)` |
| Request fails when the cache is down | Cache outage becomes a full outage | Log and fall through to the source |
| `KEYS *`, `FLUSHALL` in application code | Blocks or wipes a shared instance | `SCAN` with a prefix; version bump for bulk invalidation |
| Same TTL for keys written together | Mass expiry, synchronized load spike | TTL jitter + singleflight |
| Lock released with plain `DEL` | Deletes another holder's lock after your lease expired | Token + Lua compare-and-delete |
| `INCR` then separate `EXPIRE` | Crash between them leaves a counter that never expires | Single Lua script or `SET NX PX` first |
| Unbounded `PoolSize`/no timeouts | Slow Redis ties up goroutines and connections | Bounded pool, short read/write and pool timeouts, ctx everywhere |
| Domain events over Pub/Sub | Messages lost when a consumer restarts | NATS JetStream |
| Caching pgx rows or domain structs directly | Field renames silently break cached data | Dedicated cache DTO plus key version |

## Related

- Skill: postgres-patterns - source-of-truth queries the cache sits in front of
- Skill: nats-jetstream-patterns - durable messaging and event-driven cache invalidation
- Skill: docker-patterns - local Redis with password and AOF in compose
- Skill: golang-testing - testcontainers and fakes
- Agent: database-reviewer - review of data access including caching
