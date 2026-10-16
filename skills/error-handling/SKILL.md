---
name: error-handling
description: Cross-cutting error strategy for Go services - domain error kinds (not found, conflict, validation, unauthorized, forbidden, unavailable), sentinel vs typed errors, wrapping with %w, errors.Is/As/Join, translating pgx errors to domain errors and domain errors to HTTP problem responses or NATS ack/nak/term, logging once with slog, panic recovery, retries with backoff and jitter, and context cancellation. Use whenever writing or reviewing Go code that creates, wraps, returns, logs, maps or retries errors, designs an HTTP error response, or handles a failed NATS message - even if the user only asks to "add a handler" or "fix this bug".
metadata:
  origin: ECC
---

# Error Handling for Go Services

One strategy for errors from the database to the wire: adapters translate infrastructure errors into a small set of domain kinds, use cases wrap with context, and exactly one boundary per request or message decides the response and logs. Everything else just returns.

If the project has its own error-code or error-response specification (for example an error-codes document in a spec repo), it takes precedence over the response shapes and status mappings shown here. Keep the mechanics (kinds, wrapping, translate at boundaries, log once) and adapt the output format to the spec.

Complete, compiling code for every building block below is in `references/building-blocks.md`; read it when you are implementing these pieces rather than reviewing them.

## Principles

1. **Errors are values with a kind.** Callers branch on kind (`errors.Is(err, domain.ErrNotFound)`), never on message text, because messages change and get wrapped.
2. **Translate at boundaries.** pgx errors stop in the postgres adapter; HTTP statuses start in the HTTP adapter. The domain and use cases speak only domain kinds. That keeps `pgx` out of handlers and `http.StatusX` out of business code.
3. **Wrap on the way up, decide at the top.** Each layer adds what it knows (`"mark order %s paid: %w"`). The boundary (HTTP error writer, NATS message handler, `main`) logs and responds.
4. **Log once.** Log-and-return produces the same failure three times in the logs with partial context each time. Return the error; the boundary logs it with the full wrapped chain.
5. **Never leak internals.** SQL, stack traces, hostnames and driver messages go to logs, not to clients.

## Domain error kinds

Define a small, closed set in `internal/domain` and wrap every error that crosses the app/adapter boundary with one of them. An error that wraps none is "internal" (500, nak).

```go
var (
	ErrNotFound     = errors.New("not found")
	ErrConflict     = errors.New("conflict")          // duplicate, version mismatch, illegal state transition
	ErrValidation   = errors.New("validation failed") // input breaks a domain rule
	ErrUnauthorized = errors.New("unauthorized")      // caller is not authenticated
	ErrForbidden    = errors.New("forbidden")         // authenticated but not allowed
	ErrUnavailable  = errors.New("unavailable")       // transient: dependency down, timeout, serialization failure
)
```

Resist adding a kind per situation (`ErrOrderAlreadyPaid`). Add specific sentinels only when a caller must branch on them, and make them wrap a kind so boundaries still map them without changes: `ErrAlreadyPaid = fmt.Errorf("order already paid: %w", ErrConflict)`.

## Sentinel vs typed errors

- **Sentinel** (`var ErrX = errors.New(...)`): the caller only needs to know *what kind* of failure. Cheap, comparable with `errors.Is`. Default choice.
- **Typed** (`type ValidationError struct{ Fields []FieldError }`): the caller needs *data* from the error (which fields, retry-after, conflicting ID). Extract with `errors.As`.

Give typed errors an `Is` method so they also match their kind; boundaries can then map by kind and extract details only when present:

```go
func (e *ValidationError) Is(target error) bool { return target == ErrValidation }
```

Return typed errors as pointers and return a literal `nil` (not a nil `*ValidationError`) when there is no error; a typed nil pointer in an `error` interface is non-nil and is a classic bug. The `Err()` helper in the building blocks avoids it.

## Wrapping

- Wrap with `fmt.Errorf("verb object %s: %w", id, err)`. Describe the operation that failed, not "error occurred" or "failed to"; the chain reads as a sentence: `create order: insert order: conflict: ERROR: duplicate key ...`.
- Use `%w` when callers may need to inspect the cause; use `%v` deliberately when you want to hide an implementation detail from `errors.Is/As` (e.g. a library's error type you do not want callers to depend on).
- Go 1.20+ supports several `%w` in one call. Use it to attach a domain kind while keeping the driver error inspectable: `fmt.Errorf("insert user: %w: %w", domain.ErrConflict, err)`.
- Do not wrap when the error already says everything (`return err` from a one-line passthrough is fine).
- Never compare with `==` or `err.Error() ==`; use `errors.Is`. Use `errors.As` with a pointer to the target type (`var pgErr *pgconn.PgError; errors.As(err, &pgErr)`).

## errors.Join

Use `errors.Join` when several independent errors are all relevant:

- Collecting cleanup errors without losing the primary one:

  ```go
  func writeReport(path string, data []byte) (err error) {
  	f, err := os.Create(path)
  	if err != nil {
  		return fmt.Errorf("create report: %w", err)
  	}
  	defer func() { err = errors.Join(err, f.Close()) }() // Close errors matter for writes
  	_, err = f.Write(data)
  	return err
  }
  ```

- Shutting down several components and reporting all failures.
- Returning the last attempt's error together with `ctx.Err()` when a retry loop is cancelled.

`errors.Join` output is newline-separated, which is awkward in single-line logs. For "kind + cause" prefer two `%w` verbs as above.

## Translating at boundaries

### Postgres (pgx) -> domain

All in the postgres adapter:

| pgx / SQLSTATE | Domain kind | Why |
|---|---|---|
| `pgx.ErrNoRows` | `ErrNotFound` | Single-row lookup found nothing |
| `23505` unique_violation | `ErrConflict` | Duplicate key |
| `23503` foreign_key_violation, `23514` check_violation | `ErrValidation` (or `ErrConflict` for FK on delete) | Input refers to something missing or breaks a constraint |
| `40001` serialization_failure, `40P01` deadlock_detected, `55P03` lock_not_available | `ErrUnavailable` | Safe to retry the whole transaction |
| `pgconn.Timeout(err)`, `pgconn.SafeToRetry(err)`, connection refused | `ErrUnavailable` | Dependency problem, not the caller's fault |
| `context.Canceled` / `DeadlineExceeded` | leave as is | Boundaries handle context errors specially |

Check `RowsAffected()` on `UPDATE`/`DELETE` and return `ErrNotFound` on zero; pgx does not error there. Database constraints are the last line of defense for invariants; translating their violations correctly is what makes race-condition duplicates come back as 409 instead of 500.

### Domain -> HTTP

One function, used by every handler, maps kinds to statuses and writes an RFC 9457 `application/problem+json` body (unless the project spec says otherwise):

| Kind | Status | Body detail |
|---|---|---|
| `ErrValidation` | 422 (400 for malformed syntax caught in the adapter) | Field errors from `*ValidationError` |
| `ErrNotFound` | 404 | Generic title; never echo internal IDs you did not receive |
| `ErrConflict` | 409 | Safe, domain-level description |
| `ErrUnauthorized` | 401 (+ `WWW-Authenticate`) | Generic |
| `ErrForbidden` | 403 (or 404 if existence itself is secret) | Generic |
| `ErrUnavailable`, `context.DeadlineExceeded` | 503 (+ `Retry-After`) | Generic |
| anything else | 500 | Generic message plus a request/trace ID for support |

Map by `errors.Is` in a fixed order, not with a type switch on concrete types, so wrapped and typed errors both match. Never put `err.Error()` into a 500 response: it contains SQL, table names and hostnames. For 4xx, send only text written for clients (field messages, a fixed title).

If handlers are generated with oapi-codegen strict server, plug the same mapping into its `ResponseErrorHandlerFunc` and `RequestErrorHandlerFunc` so generated and hand-written routes respond identically (see the contract-first skill).

### Domain -> NATS JetStream

A consumer has three outcomes per message, and the error kind picks one:

| Outcome | When | Effect |
|---|---|---|
| `msg.Ack()` | Success, or a duplicate already processed (`ErrConflict` on an idempotency key) | Done |
| `msg.NakWithDelay(d)` | `ErrUnavailable`, `DeadlineExceeded`, shutdown-time `context.Canceled`, unknown errors | Redelivered after `d`, up to `MaxDeliver` |
| `msg.TermWithReason(r)` | Undecodable payload, `ErrValidation`, unknown schema version: retrying cannot help | Never redelivered; log it and route to a dead-letter path |

```go
func (c *Consumer) handle(ctx context.Context, msg jetstream.Msg) {
	err := c.process(ctx, msg)
	switch {
	case err == nil, errors.Is(err, domain.ErrConflict): // conflict = already applied
		err = msg.Ack()
	case errors.Is(err, errDecode), errors.Is(err, domain.ErrValidation):
		c.log.ErrorContext(ctx, "poison message", "subject", msg.Subject(), "err", err)
		err = msg.TermWithReason("invalid payload")
	default:
		if !errors.Is(err, context.Canceled) {
			c.log.WarnContext(ctx, "message failed, will retry", "subject", msg.Subject(), "err", err)
		}
		err = msg.NakWithDelay(c.redeliveryDelay(msg))
	}
	if err != nil {
		c.log.WarnContext(ctx, "ack/nak failed", "subject", msg.Subject(), "err", err)
	}
}
```

Base `redeliveryDelay` on `msg.Metadata()`'s `NumDelivered` (exponential, capped), or configure `BackOff` on the consumer instead. Always set `MaxDeliver` so an unknown error cannot loop forever, and alert on max-deliveries advisories. Treating `ErrConflict` as success assumes the consumer is idempotent; the nats-jetstream-patterns skill covers that and dead-letter handling.

## Logging once, at the boundary

```go
// Wrong: logs here, then the caller logs again, then the handler logs a third time.
if err != nil {
	slog.Error("failed to load order", "err", err)
	return err
}

// Right: add context and return; the boundary logs once.
if err != nil {
	return fmt.Errorf("load order %s: %w", id, err)
}
```

At the boundary:

- Use `slog.ErrorContext(ctx, ...)` / `InfoContext` so request ID and trace IDs from the context handler are attached.
- Log 5xx and unknown errors at Error with the full chain (`"err", err`). Log 4xx at Info or Debug; they are client behavior, and logging them at Error makes alerts useless.
- Client cancellation (`context.Canceled` while `r.Context().Err() != nil`) is Debug at most and gets no response.
- Do not log secrets or full request bodies alongside errors.

The exceptions to "never log and return": background goroutines and fire-and-forget work have no caller to return to, so they *are* a boundary and must log (or report on a channel) themselves.

## Panics and recovery

- Panic only for programmer errors that make continuing unsafe (impossible state, nil dependency at construction). Never use panic for expected failures such as bad input or a missing row.
- Install one recovery middleware as the outermost HTTP handler: recover, log once with `debug.Stack()`, return 500. Re-panic `http.ErrAbortHandler`, which is the documented way to abort a response.
- Recovery middleware does not cover goroutines started by the handler. A panic in any goroutine crashes the process. Every `go func()` in long-running code either cannot panic or has its own `defer recover()` that logs and reports.
- In NATS consumers, recover inside the per-message handler and `Nak` (or `Term` if the same message keeps panicking, judged by `NumDelivered`), so one poison message cannot kill the consumer loop.
- Do not recover in libraries or use cases; let panics reach the boundary that owns the process or request.

## Retries

Retry only when another attempt can plausibly succeed:

- Retry: `ErrUnavailable` (timeouts, connection errors, `40001`/`40P01`), HTTP 502/503/504 and 429 (honor `Retry-After`), NATS publish timeouts.
- Never retry: validation, not found, conflict, unauthorized, forbidden, any other 4xx, `context.Canceled`.
- Only retry operations that are idempotent, or make them idempotent (idempotency key, upsert, JetStream `Nats-Msg-Id` dedup). Retrying a non-idempotent payment call doubles the charge.

Use exponential backoff with full jitter (`delay = rand(0, min(max, base*2^attempt))`) so a fleet of clients does not retry in lockstep, cap attempts and total time, and stop immediately when `ctx` is done. Retry at one layer only; retries nested at the client, the use case and the consumer multiply (3 x 3 x 3 = 27 attempts against a struggling dependency). For transactions, retry the whole `InTx` call, never a single statement inside it.

The `retry.Do` helper in the building blocks implements this; libraries like `cenkalti/backoff/v5` are fine too. Do not hand-roll `time.Sleep` loops: they ignore cancellation and hold the request open during shutdown.

## context.Canceled and DeadlineExceeded

- `context.Canceled` means someone upstream gave up (client disconnected, shutdown). It is not a failure of your service: do not log at Error, do not retry, do not count it in error-rate SLOs. In HTTP, write nothing. In consumers, `Nak` so the message is redelivered after restart.
- `context.DeadlineExceeded` means a timeout you or a caller set expired. Treat it as transient (`503`, nak with delay), and log at Warn with what timed out. Retrying makes sense only if the outer context still has time left; check `ctx.Err()` before each attempt.
- Check with `errors.Is`, never `==`: drivers wrap context errors. pgx returns errors that match `context.Canceled` when the query was interrupted.
- Pass `ctx` to every I/O call so cancellation actually stops work; an operation that ignores `ctx` cannot be timed out.
- When work must finish even if the request is cancelled (writing an audit record after the response), use `context.WithoutCancel(ctx)` with its own timeout rather than `context.Background()`, so values like trace IDs survive.

## Review checklist

- [ ] No ignored errors (`_ =` only with a comment saying why).
- [ ] Errors are wrapped with operation context and `%w`; no `err.Error()` string comparisons.
- [ ] Every error leaving an adapter wraps a domain kind or is intentionally internal.
- [ ] `pgx.ErrNoRows`, unique violations and zero `RowsAffected` are translated in the repository.
- [ ] One error-to-response function per transport; no ad-hoc `http.Error` with `err.Error()`.
- [ ] 500 responses and logs: generic body to client, full chain in logs, request/trace ID in both.
- [ ] Each error is logged exactly once, at the boundary, with the right level.
- [ ] Recovery middleware is outermost; long-running goroutines recover and report.
- [ ] Consumers ack, nak with delay, or term based on error kind; `MaxDeliver` is set.
- [ ] Retries only for transient errors on idempotent operations, with jitter, caps and `ctx` checks, at one layer.
- [ ] `context.Canceled` is not logged as an error or retried.

The silent-failure-hunter agent is a good second pass for swallowed errors and empty fallbacks.
