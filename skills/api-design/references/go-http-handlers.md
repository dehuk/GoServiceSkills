# Go HTTP Handler Reference Implementation

Standard-library-only helpers and a handler for Go 1.24+ `net/http`. These compile as-is (`go vet` clean). Adapt the problem format to the project's error spec if it has one.

Read this when writing the JSON helpers, request decoding, error mapping, cursor encoding, or server setup of a new service. The SKILL.md explains the reasoning; this file is the code.

## Response and problem helpers, strict request decoding

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"strings"
)

// Problem is an RFC 9457 problem details object.
type Problem struct {
	Type     string       `json:"type"`
	Title    string       `json:"title"`
	Status   int          `json:"status"`
	Detail   string       `json:"detail,omitempty"`
	Instance string       `json:"instance,omitempty"`
	Errors   []FieldError `json:"errors,omitempty"` // extension member
}

type FieldError struct {
	Field   string `json:"field"`
	Code    string `json:"code"`
	Message string `json:"message"`
}

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	// The status line is already sent; an encode error here can only be logged.
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("encode response", "err", err)
	}
}

func writeProblem(w http.ResponseWriter, p Problem) {
	w.Header().Set("Content-Type", "application/problem+json")
	w.WriteHeader(p.Status)
	if err := json.NewEncoder(w).Encode(p); err != nil {
		slog.Error("encode problem", "err", err)
	}
}

const maxBodyBytes = 1 << 20 // 1 MiB

// decodeJSON decodes exactly one JSON object from the request body into dst.
// It returns a *Problem describing the client error, or nil on success.
func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) *Problem {
	if ct := r.Header.Get("Content-Type"); ct != "" && !strings.HasPrefix(ct, "application/json") {
		return &Problem{Type: "about:blank", Title: "Unsupported Media Type", Status: http.StatusUnsupportedMediaType}
	}
	r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	if err := dec.Decode(dst); err != nil {
		var syntaxErr *json.SyntaxError
		var typeErr *json.UnmarshalTypeError
		var maxErr *http.MaxBytesError
		bad := func(detail string) *Problem {
			return &Problem{Type: "about:blank", Title: "Bad Request", Status: http.StatusBadRequest, Detail: detail}
		}
		switch {
		case errors.As(err, &syntaxErr):
			return bad(fmt.Sprintf("malformed JSON at offset %d", syntaxErr.Offset))
		case errors.Is(err, io.ErrUnexpectedEOF):
			return bad("malformed JSON")
		case errors.As(err, &typeErr):
			return bad(fmt.Sprintf("field %q has the wrong type", typeErr.Field))
		case strings.HasPrefix(err.Error(), "json: unknown field "):
			// encoding/json has no typed error for unknown fields.
			return bad(strings.TrimPrefix(err.Error(), "json: "))
		case errors.Is(err, io.EOF):
			return bad("request body is empty")
		case errors.As(err, &maxErr):
			return &Problem{Type: "about:blank", Title: "Content Too Large", Status: http.StatusRequestEntityTooLarge,
				Detail: fmt.Sprintf("body exceeds %d bytes", maxErr.Limit)}
		default:
			return bad("invalid request body")
		}
	}
	// Reject trailing data such as `{"a":1}{"b":2}`; Decode alone would ignore it.
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return &Problem{Type: "about:blank", Title: "Bad Request", Status: http.StatusBadRequest,
			Detail: "body must contain a single JSON object"}
	}
	return nil
}
```

Why each step in `decodeJSON` exists:

- `http.MaxBytesReader` caps memory per request; without it a single client can make the service buffer an arbitrarily large body. It also tells the server to close the connection after the limit is hit.
- `DisallowUnknownFields` turns a misspelled field (`"emial"`) into a 400 instead of a silently ignored value that later fails validation with a confusing message, or worse, passes because the field was optional.
- The second `Decode` rejects trailing data; `Decode` alone stops after the first value.
- Errors are mapped to client-safe messages. Do not echo raw `err.Error()` for unexpected errors; it can leak Go type names.

## Handler with validation, error mapping, and Location header

```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"net/mail"
	"strings"
	"time"
	"unicode/utf8"
)

var (
	ErrNotFound   = errors.New("not found")
	ErrEmailTaken = errors.New("email taken")
)

type User struct {
	ID        string    `json:"id"`
	Email     string    `json:"email"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"created_at"`
	DeletedAt time.Time `json:"deleted_at,omitzero"` // omitted when zero (Go 1.24+)
}

type UserService interface {
	Create(ctx context.Context, email, name string) (User, error)
	Get(ctx context.Context, id string) (User, error)
}

type createUserRequest struct {
	Email string `json:"email"`
	Name  string `json:"name"`
}

func (r createUserRequest) validate() []FieldError {
	var errs []FieldError
	if _, err := mail.ParseAddress(r.Email); err != nil {
		errs = append(errs, FieldError{Field: "email", Code: "invalid_format", Message: "must be a valid email address"})
	}
	if n := utf8.RuneCountInString(strings.TrimSpace(r.Name)); n == 0 || n > 100 {
		errs = append(errs, FieldError{Field: "name", Code: "out_of_range", Message: "must be 1-100 characters"})
	}
	return errs
}

type UserHandler struct {
	svc UserService
	log *slog.Logger
}

func (h *UserHandler) Register(mux *http.ServeMux) {
	mux.HandleFunc("POST /v1/users", h.create)
	mux.HandleFunc("GET /v1/users/{id}", h.get)
}

func (h *UserHandler) create(w http.ResponseWriter, r *http.Request) {
	var req createUserRequest
	if p := decodeJSON(w, r, &req); p != nil {
		writeProblem(w, *p)
		return
	}
	if errs := req.validate(); len(errs) > 0 {
		writeProblem(w, Problem{
			Type: "about:blank", Title: "Validation Failed", Status: http.StatusUnprocessableEntity,
			Errors: errs,
		})
		return
	}

	user, err := h.svc.Create(r.Context(), req.Email, req.Name)
	switch {
	case err == nil:
	case errors.Is(err, ErrEmailTaken):
		writeProblem(w, Problem{Type: "about:blank", Title: "Conflict", Status: http.StatusConflict,
			Detail: "email is already registered"})
		return
	default:
		h.internalError(w, r, err)
		return
	}

	w.Header().Set("Location", "/v1/users/"+user.ID)
	writeJSON(w, http.StatusCreated, user)
}

func (h *UserHandler) get(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	user, err := h.svc.Get(r.Context(), id)
	switch {
	case err == nil:
		writeJSON(w, http.StatusOK, user)
	case errors.Is(err, ErrNotFound):
		writeProblem(w, Problem{Type: "about:blank", Title: "Not Found", Status: http.StatusNotFound})
	default:
		h.internalError(w, r, err)
	}
}

// internalError logs the real cause and returns a generic 500, so SQL errors
// and stack details never reach the client.
func (h *UserHandler) internalError(w http.ResponseWriter, r *http.Request, err error) {
	h.log.ErrorContext(r.Context(), "request failed", "method", r.Method, "path", r.URL.Path, "err", err)
	writeProblem(w, Problem{Type: "about:blank", Title: "Internal Server Error", Status: http.StatusInternalServerError})
}
```

Notes:

- `r.PathValue("id")` reads the `{id}` wildcard registered on the Go 1.22+ ServeMux; no router library is needed.
- Validation collects all field errors in one response, so clients fix everything in one round trip.
- Domain errors (`ErrNotFound`, `ErrEmailTaken`) are mapped to HTTP in the handler, the only layer that knows about HTTP. The service layer must not return status codes. See the **error-handling** and **hexagonal-architecture** skills.

## Opaque cursor

```go
package main

import (
	"encoding/base64"
	"encoding/json"
	"errors"
	"time"
)

// cursor is the keyset position after the last returned row.
type cursor struct {
	CreatedAt time.Time `json:"c"`
	ID        string    `json:"i"`
}

func encodeCursor(c cursor) (string, error) {
	b, err := json.Marshal(c)
	if err != nil {
		return "", err
	}
	return base64.RawURLEncoding.EncodeToString(b), nil
}

var errBadCursor = errors.New("invalid cursor")

// decodeCursor treats the cursor as untrusted client input.
func decodeCursor(s string) (cursor, error) {
	var c cursor
	b, err := base64.RawURLEncoding.DecodeString(s)
	if err != nil || len(b) > 256 {
		return c, errBadCursor
	}
	if err := json.Unmarshal(b, &c); err != nil || c.ID == "" || c.CreatedAt.IsZero() {
		return cursor{}, errBadCursor
	}
	return c, nil
}
```

The matching keyset query (pgx, `$n` placeholders), fetching one extra row to compute `has_more`:

```sql
SELECT id, email, name, created_at
FROM users
WHERE ($1::timestamptz IS NULL OR (created_at, id) < ($1, $2))
ORDER BY created_at DESC, id DESC
LIMIT $3 + 1;
```

It needs an index on `(created_at DESC, id DESC)`. The `id` tiebreaker matters: without it, rows sharing a `created_at` value are skipped or repeated across pages.

## Server timeouts and graceful shutdown

```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func run(handler http.Handler) error {
	srv := &http.Server{
		Addr:              ":8080",
		Handler:           handler,
		ReadHeaderTimeout: 5 * time.Second,  // slowloris protection
		ReadTimeout:       15 * time.Second, // whole request incl. body
		WriteTimeout:      30 * time.Second, // must exceed the slowest handler
		IdleTimeout:       120 * time.Second,
	}

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	errCh := make(chan error, 1)
	go func() { errCh <- srv.ListenAndServe() }()

	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return err
	}
	if err := <-errCh; !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}

```

`WriteTimeout` covers the whole handler run, so it must exceed the slowest legitimate request; for per-route budgets, set a `context.WithTimeout` on `r.Context()` in the handler or middleware and pass that context down to pgx, Redis, and NATS calls so they abort together.
