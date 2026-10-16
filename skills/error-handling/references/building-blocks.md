# Error-handling building blocks

Complete, compiling versions of the pieces SKILL.md describes (Go 1.24+, pgx v5). Read this when you are implementing the error kinds, the HTTP error writer, the recovery middleware, the pgx translation, or the retry helper, and want code to adapt rather than reinvent. Package names are illustrative; in a hexagonal layout `domain` is `internal/domain`, `httpx` lives under `internal/adapters/http`, `pg` under `internal/adapters/postgres`.

If the project has its own error-code or error-response specification, adapt `WriteError` and the `mappings` table to it; that spec wins over the shapes here.

## Domain error kinds and ValidationError

Sentinel kinds plus one typed error that carries field details and still matches its kind via an `Is` method.

```go
// domain/errors.go
package domain

import (
	"errors"
	"strings"
)

// Error kinds. Every error that crosses the app/adapter boundary should wrap
// exactly one of these (or none, meaning "internal").
var (
	ErrNotFound     = errors.New("not found")
	ErrConflict     = errors.New("conflict")
	ErrValidation   = errors.New("validation failed")
	ErrUnauthorized = errors.New("unauthorized")
	ErrForbidden    = errors.New("forbidden")
	ErrUnavailable  = errors.New("unavailable")
)

type FieldError struct {
	Field   string
	Message string
}

// ValidationError carries per-field details and still matches ErrValidation.
type ValidationError struct {
	Fields []FieldError
}

func (e *ValidationError) Error() string {
	parts := make([]string, 0, len(e.Fields))
	for _, f := range e.Fields {
		parts = append(parts, f.Field+": "+f.Message)
	}
	return "validation failed: " + strings.Join(parts, "; ")
}

func (e *ValidationError) Is(target error) bool { return target == ErrValidation }

// Add records a field problem; Err returns nil when there were none.
func (e *ValidationError) Add(field, msg string) { e.Fields = append(e.Fields, FieldError{field, msg}) }

func (e *ValidationError) Err() error {
	if len(e.Fields) == 0 {
		return nil
	}
	return e
}
```

## Collecting validation errors

Collect every field problem instead of failing on the first one, so clients can fix the whole form in one round trip.

```go
// domain/user.go
package domain

import "strings"

type NewUser struct {
	Email string
	Name  string
}

func (u NewUser) Validate() error {
	var v ValidationError
	if !strings.Contains(u.Email, "@") {
		v.Add("email", "must be a valid email address")
	}
	if u.Name == "" {
		v.Add("name", "is required")
	}
	return v.Err()
}
```

## Translating pgx errors

The only place that knows about `pgx` and SQLSTATE codes. Two `%w` verbs keep both the domain kind and the original driver error matchable. Codes are spelled out with comments; `github.com/jackc/pgerrcode` provides named constants if you prefer.

```go
// pg/translate.go
package pg

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"

	"example.com/svc/domain"
)

// translate maps pgx/Postgres errors to domain kinds while keeping the
// original error in the chain for logs. Two %w verbs keep both matchable
// with errors.Is/As.
func translate(err error, what string) error {
	if err == nil {
		return nil
	}
	if errors.Is(err, pgx.ErrNoRows) {
		return fmt.Errorf("%s: %w", what, domain.ErrNotFound)
	}
	if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
		return fmt.Errorf("%s: %w", what, err) // keep ctx errors as they are
	}
	var pgErr *pgconn.PgError
	if errors.As(err, &pgErr) {
		switch pgErr.Code {
		case "23505": // unique_violation
			return fmt.Errorf("%s: %w: %w", what, domain.ErrConflict, err)
		case "23503", "23514": // foreign_key_violation, check_violation
			return fmt.Errorf("%s: %w: %w", what, domain.ErrValidation, err)
		case "40001", "40P01", "55P03", "57P01": // serialization_failure, deadlock_detected, lock_not_available, admin_shutdown
			return fmt.Errorf("%s: %w: %w", what, domain.ErrUnavailable, err)
		}
		return fmt.Errorf("%s: %w", what, err)
	}
	if pgconn.SafeToRetry(err) || pgconn.Timeout(err) {
		return fmt.Errorf("%s: %w: %w", what, domain.ErrUnavailable, err)
	}
	return fmt.Errorf("%s: %w", what, err)
}

type querier interface {
	QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

type UserRepo struct{ q querier }

func (r *UserRepo) Email(ctx context.Context, id string) (string, error) {
	var email string
	err := r.q.QueryRow(ctx, `SELECT email FROM users WHERE id = $1`, id).Scan(&email)
	return email, translate(err, "select user "+id)
}
```

## HTTP error writer (RFC 9457 problem details)

The single function that turns an error into a response and logs it. Client-caused errors are logged at Info, server-side failures at Error with the full chain, client cancellation at Debug.

```go
// httpx/problem.go
package httpx

import (
	"context"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"

	"example.com/svc/domain"
)

// Problem is an RFC 9457 problem details body.
type Problem struct {
	Type     string       `json:"type"`
	Title    string       `json:"title"`
	Status   int          `json:"status"`
	Detail   string       `json:"detail,omitempty"`
	Instance string       `json:"instance,omitempty"`
	Errors   []fieldError `json:"errors,omitempty"`
}

// fieldError is the transport shape; domain.FieldError has no JSON tags.
type fieldError struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

type kindMapping struct {
	kind   error
	status int
	title  string
}

// Order matters: the first match wins.
var mappings = []kindMapping{
	{domain.ErrValidation, http.StatusUnprocessableEntity, "Validation failed"},
	{domain.ErrNotFound, http.StatusNotFound, "Resource not found"},
	{domain.ErrConflict, http.StatusConflict, "Conflict"},
	{domain.ErrUnauthorized, http.StatusUnauthorized, "Authentication required"},
	{domain.ErrForbidden, http.StatusForbidden, "Forbidden"},
	{domain.ErrUnavailable, http.StatusServiceUnavailable, "Service temporarily unavailable"},
}

// WriteError is the single place where an error becomes an HTTP response and
// where request-scoped errors are logged.
func WriteError(w http.ResponseWriter, r *http.Request, log *slog.Logger, err error) {
	ctx := r.Context()

	if errors.Is(err, context.Canceled) && ctx.Err() != nil {
		// The client went away; nobody will read a response.
		log.DebugContext(ctx, "request canceled by client", "path", r.URL.Path)
		return
	}

	p := Problem{Type: "about:blank", Status: http.StatusInternalServerError, Title: "Internal server error"}
	for _, m := range mappings {
		if errors.Is(err, m.kind) {
			p.Status, p.Title = m.status, m.title
			break
		}
	}
	if p.Status == http.StatusInternalServerError && errors.Is(err, context.DeadlineExceeded) {
		p.Status, p.Title = http.StatusServiceUnavailable, "Service temporarily unavailable"
	}

	var ve *domain.ValidationError
	if errors.As(err, &ve) {
		for _, f := range ve.Fields { // field messages are written for clients
			p.Errors = append(p.Errors, fieldError{Field: f.Field, Message: f.Message})
		}
	}

	attrs := []any{"status", p.Status, "method", r.Method, "path", r.URL.Path, "err", err}
	switch {
	case p.Status >= 500:
		log.ErrorContext(ctx, "request failed", attrs...)
	default:
		log.InfoContext(ctx, "request rejected", attrs...)
	}

	if p.Status == http.StatusServiceUnavailable {
		w.Header().Set("Retry-After", "1")
	}
	w.Header().Set("Content-Type", "application/problem+json")
	w.WriteHeader(p.Status)
	_ = json.NewEncoder(w).Encode(p) // headers are sent; nothing useful to do on failure
}
```

## Recovery middleware

Outermost middleware. Re-panics `http.ErrAbortHandler` because that is the documented way to abort a response.

```go
// httpx/recover.go
package httpx

import (
	"log/slog"
	"net/http"
	"runtime/debug"
)

// Recover turns a handler panic into a 500 and one log line with the stack.
// It must be the outermost middleware so it also covers other middleware.
func Recover(log *slog.Logger, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			rec := recover()
			if rec == nil {
				return
			}
			if rec == http.ErrAbortHandler {
				panic(rec) // deliberate abort: let net/http close the connection silently
			}
			log.ErrorContext(r.Context(), "panic in handler",
				"panic", rec, "method", r.Method, "path", r.URL.Path, "stack", string(debug.Stack()))
			// If the handler already wrote headers this is a no-op on the status,
			// but the connection is still closed cleanly.
			http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
		}()
		next.ServeHTTP(w, r)
	})
}
```

## Retry with exponential backoff and full jitter

Retries only transient errors, respects context cancellation, and never sleeps after the final attempt.

```go
// retry/retry.go
package retry

import (
	"context"
	"errors"
	"math/rand/v2"
	"time"

	"example.com/svc/domain"
)

type Policy struct {
	Attempts int           // total attempts, including the first
	Base     time.Duration // delay before the second attempt (before jitter)
	Max      time.Duration // cap for a single delay
}

// Transient reports whether retrying could plausibly succeed.
func Transient(err error) bool {
	return errors.Is(err, domain.ErrUnavailable) || errors.Is(err, context.DeadlineExceeded)
}

// Do calls fn until it succeeds, returns a non-transient error, runs out of
// attempts, or ctx is done. Delays use exponential backoff with full jitter.
func Do(ctx context.Context, p Policy, fn func(ctx context.Context) error) error {
	p.Attempts = max(p.Attempts, 1)
	p.Base = max(p.Base, time.Millisecond) // rand.N panics on 0
	var err error
	for attempt := range p.Attempts {
		if err = fn(ctx); err == nil || !Transient(err) {
			return err
		}
		if attempt == p.Attempts-1 {
			break
		}
		backoff := min(p.Base<<attempt, p.Max)
		delay := rand.N(backoff) + 1 // full jitter in (0, backoff]
		t := time.NewTimer(delay)
		select {
		case <-ctx.Done():
			t.Stop()
			return errors.Join(err, ctx.Err())
		case <-t.C:
		}
	}
	return err
}
```

## Retry test

Shows the two behaviors worth pinning: transient errors are retried up to the limit, permanent ones are returned immediately.

```go
// retry/retry_test.go
package retry

import (
	"context"
	"errors"
	"fmt"
	"testing"
	"time"

	"example.com/svc/domain"
)

func TestDo(t *testing.T) {
	calls := 0
	err := Do(context.Background(), Policy{Attempts: 3, Base: time.Millisecond, Max: 5 * time.Millisecond}, func(context.Context) error {
		calls++
		return fmt.Errorf("x: %w", domain.ErrUnavailable)
	})
	if calls != 3 || !errors.Is(err, domain.ErrUnavailable) {
		t.Fatal(calls, err)
	}
	calls = 0
	err = Do(context.Background(), Policy{Attempts: 3}, func(context.Context) error { calls++; return domain.ErrNotFound })
	if calls != 1 || !errors.Is(err, domain.ErrNotFound) {
		t.Fatal(calls, err)
	}
}
```
