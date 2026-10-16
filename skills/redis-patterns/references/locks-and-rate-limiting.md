# Distributed Locks and Rate Limiting with go-redis/v9

Read this before implementing a Redis lock or rate limiter. Both look like three lines of code, and both have failure modes that only show up under load or during a Redis failover.

## Distributed Locks

### First ask whether you need one

A Redis lock is a lease with a timeout. If the holder pauses (GC, CPU throttling, a slow downstream call) longer than the TTL, the lease expires, a second process acquires it, and both run the critical section. Redis replication is asynchronous, so a failover can also lose a lock that was just granted. Redlock does not fix the pause problem and adds operational cost; do not reach for it.

So use Redis locks for **efficiency**: avoid running the same expensive job twice, avoid a thundering herd on a cache rebuild. When overlap would corrupt data, make the database enforce correctness instead:

| Need | Database mechanism |
|---|---|
| Create something at most once | Unique index plus `INSERT ... ON CONFLICT DO NOTHING` |
| Update only if nobody else did | Conditional update: `UPDATE orders SET ..., version = version + 1 WHERE id = $1 AND version = $2`, check rows affected |
| Serialize work on one row | `SELECT ... FOR UPDATE` inside the transaction |
| Serialize work on a logical key across a transaction | `SELECT pg_advisory_xact_lock(hashtext($1))` |
| One consumer per job | `SELECT ... FOR UPDATE SKIP LOCKED` job table, or a NATS JetStream work-queue stream |

If you keep a Redis lock in front of a write that must not overlap, add a fencing token: `INCR` a counter when acquiring, pass the number to the write, and have the database reject writes with a token lower than the last one seen (`UPDATE ... WHERE last_fence < $token`). See the postgres-patterns skill for the query patterns.

### Implementation

The lock value is a random token unique to this holder. Release and extend must check the token atomically (Lua), otherwise a holder whose lease expired deletes or extends someone else's lock.

```go
package lock

import (
	"context"
	"crypto/rand"
	"errors"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

var (
	ErrNotAcquired = errors.New("lock not acquired")
	ErrLockLost    = errors.New("lock lost")
)

var releaseScript = redis.NewScript(`
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
end
return 0
`)

var extendScript = redis.NewScript(`
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('PEXPIRE', KEYS[1], ARGV[2])
end
return 0
`)

type Lock struct {
	rdb   redis.UniversalClient
	key   string
	token string
	ttl   time.Duration
}

// Acquire tries once to take the lock. It returns ErrNotAcquired if another holder has it.
func Acquire(ctx context.Context, rdb redis.UniversalClient, key string, ttl time.Duration) (*Lock, error) {
	token := rand.Text() // crypto/rand, Go 1.24+
	ok, err := rdb.SetNX(ctx, key, token, ttl).Result()
	if err != nil {
		return nil, fmt.Errorf("acquire lock %s: %w", key, err)
	}
	if !ok {
		return nil, ErrNotAcquired
	}
	return &Lock{rdb: rdb, key: key, token: token, ttl: ttl}, nil
}

// Release deletes the lock if this holder still owns it.
func (l *Lock) Release(ctx context.Context) error {
	n, err := releaseScript.Run(ctx, l.rdb, []string{l.key}, l.token).Int64()
	if err != nil {
		return fmt.Errorf("release lock %s: %w", l.key, err)
	}
	if n == 0 {
		return fmt.Errorf("release lock %s: %w", l.key, ErrLockLost)
	}
	return nil
}

// Extend resets the TTL if this holder still owns the lock.
func (l *Lock) Extend(ctx context.Context) error {
	n, err := extendScript.Run(ctx, l.rdb, []string{l.key}, l.token, l.ttl.Milliseconds()).Int64()
	if err != nil {
		return fmt.Errorf("extend lock %s: %w", l.key, err)
	}
	if n == 0 {
		return fmt.Errorf("extend lock %s: %w", l.key, ErrLockLost)
	}
	return nil
}
```

`WithLock` runs `fn` while holding the lock, renews it at a third of the TTL, and cancels `fn`'s context if renewal fails so the work stops as soon as ownership is uncertain:

```go
// WithLock runs fn while holding key. fn must stop promptly when its context is cancelled.
func WithLock(ctx context.Context, rdb redis.UniversalClient, key string, ttl time.Duration, fn func(context.Context) error) error {
	l, err := Acquire(ctx, rdb, key, ttl)
	if err != nil {
		return err
	}

	workCtx, cancel := context.WithCancelCause(ctx)
	defer cancel(nil)

	var renewErr error // written only by the renew goroutine, read after renewDone
	renewDone := make(chan struct{})
	go func() {
		defer close(renewDone)
		t := time.NewTicker(ttl / 3)
		defer t.Stop()
		for {
			select {
			case <-workCtx.Done():
				return
			case <-t.C:
				if err := l.Extend(workCtx); err != nil {
					if workCtx.Err() != nil {
						return // work finished or caller cancelled during the extend call
					}
					renewErr = err
					cancel(err)
					return
				}
			}
		}
	}()

	fnErr := fn(workCtx)
	cancel(nil)
	<-renewDone

	if renewErr != nil {
		return errors.Join(fnErr, renewErr) // lease already gone; nothing to release
	}
	// Release even if the caller's ctx is cancelled, otherwise the lock blocks others until TTL.
	relCtx, relCancel := context.WithTimeout(context.WithoutCancel(ctx), 2*time.Second)
	defer relCancel()
	return errors.Join(fnErr, l.Release(relCtx))
}
```

Usage, for a job that should run on one replica at a time:

```go
err := lock.WithLock(ctx, rdb, "orders:v1:lock:reconcile-payments", 30*time.Second, svc.ReconcilePayments)
switch {
case errors.Is(err, lock.ErrNotAcquired):
	return nil // another replica is on it
case err != nil:
	return fmt.Errorf("reconcile payments: %w", err)
}
```

Caveats to keep in the code review checklist:

- Choose the TTL well above the renewal interval plus worst-case Redis latency; with renewal, 10-30 s is typical.
- `fn` must honor its context. Renewal failure cannot stop code that ignores `ctx`.
- Waiting for a lock: retry `Acquire` with jittered backoff and a deadline from `ctx`; never spin.
- Redis being down means nobody gets the lock. Decide whether the job may run without it (efficiency lock) or must not run (then it should not be a Redis lock).
- If you prefer a library, `github.com/bsm/redislock` implements the same token pattern with retry strategies.

## Rate Limiting

Decide before writing code:

- **Subject**: user ID for authenticated APIs, API key for integrations, client IP only at the edge (NAT and proxies make IPs shared). Include the route group in the key when limits differ per endpoint.
- **Algorithm**: fixed window is cheapest and allows up to 2x the limit across a window boundary; a sliding log is exact but stores one entry per request; GCRA (via a library) is smooth and O(1).
- **Failure mode**: if Redis errors, fail open (allow, log, alert) for limits that protect capacity, fail closed (reject) for limits that protect against abuse such as login attempts or OTP sends.
- **Response**: HTTP 429 with `Retry-After`, in the error format defined by the api-design and error-handling skills.

### Fixed window (atomic INCR with expiry)

`INCR` followed by a separate `EXPIRE` can leave a counter without a TTL if the process dies in between, blocking the subject forever. Do both in one script.

```go
var fixedWindowScript = redis.NewScript(`
local n = redis.call('INCR', KEYS[1])
if n == 1 then
  redis.call('PEXPIRE', KEYS[1], ARGV[1])
end
return {n, redis.call('PTTL', KEYS[1])}
`)

type Decision struct {
	Allowed    bool
	Remaining  int64
	RetryAfter time.Duration
}

type FixedWindow struct {
	rdb    redis.UniversalClient
	limit  int64
	window time.Duration
}

func (l *FixedWindow) Allow(ctx context.Context, subject string) (Decision, error) {
	key := "orders:v1:rl:fw:" + subject
	res, err := fixedWindowScript.Run(ctx, l.rdb, []string{key}, l.window.Milliseconds()).Int64Slice()
	if err != nil {
		return Decision{}, fmt.Errorf("rate limit %s: %w", subject, err)
	}
	n, pttl := res[0], res[1]
	if n > l.limit {
		retry := time.Duration(pttl) * time.Millisecond
		if pttl < 0 { // key has no TTL (should not happen); fall back to the full window
			retry = l.window
		}
		return Decision{Allowed: false, RetryAfter: retry}, nil
	}
	return Decision{Allowed: true, Remaining: l.limit - n}, nil
}
```

### Sliding window log (sorted set)

Exact count of requests in the last `window`. Uses Redis's clock so replicas with skewed clocks agree, and a random member so concurrent requests in the same millisecond are counted separately. Memory is O(limit) per active subject, so use it for low limits (logins, sends), not for 10k requests per minute.

```go
var slidingLogScript = redis.NewScript(`
local t = redis.call('TIME')
local now = tonumber(t[1]) * 1000 + math.floor(tonumber(t[2]) / 1000)
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])

redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', now - window)
local count = redis.call('ZCARD', KEYS[1])
if count < limit then
  redis.call('ZADD', KEYS[1], now, ARGV[3])
  redis.call('PEXPIRE', KEYS[1], window)
  return {1, limit - count - 1}
end
return {0, 0}
`)

type SlidingLog struct {
	rdb    redis.UniversalClient
	limit  int64
	window time.Duration
}

func (l *SlidingLog) Allow(ctx context.Context, subject string) (Decision, error) {
	key := "orders:v1:rl:sl:" + subject
	res, err := slidingLogScript.Run(ctx, l.rdb, []string{key},
		l.window.Milliseconds(), l.limit, rand.Text()).Int64Slice()
	if err != nil {
		return Decision{}, fmt.Errorf("rate limit %s: %w", subject, err)
	}
	if res[0] == 0 {
		// Upper bound; computing the exact time needs the oldest member's score.
		return Decision{Allowed: false, RetryAfter: l.window}, nil
	}
	return Decision{Allowed: true, Remaining: res[1]}, nil
}
```

Rejected requests are not recorded, so a client hammering the endpoint regains access as soon as old entries age out. Record rejected attempts too (always `ZADD`) if you want sustained abuse to stay blocked.

### GCRA via library

For general API rate limiting, `github.com/go-redis/redis_rate/v10` implements GCRA (smooth, O(1) memory, built on go-redis v9):

```go
limiter := redis_rate.NewLimiter(rdb)
res, err := limiter.Allow(ctx, "orders:v1:rl:user:"+userID, redis_rate.PerMinute(100))
if err != nil {
	// fail open or closed, as decided above
}
if res.Allowed == 0 {
	w.Header().Set("Retry-After", strconv.Itoa(int(math.Ceil(res.RetryAfter.Seconds()))))
	// write 429
}
```

Prefer the library over hand-written scripts unless you need one of the specific behaviors above.

### Testing limiters and locks

- miniredis runs these scripts (including `TIME`) and `FastForward` advances key expiry; but `TIME` in miniredis follows the wall clock or `mr.SetTime`, not `FastForward`, so set both when testing the sliding log.
- Run a concurrency test against a real Redis (testcontainers) with `go test -race`: N goroutines calling `Allow`, assert exactly `limit` succeed; N goroutines calling `WithLock`, assert the critical section never overlaps.
