---
name: go-security
description: Security patterns and review checklist for Go backend services - secrets handling and log redaction, input validation, SQL injection with pgx, command injection, path traversal, SSRF, JWT and authorization, crypto, HTTP server and TLS hardening, rate limiting, dependency scanning, race conditions, Docker secrets, and NATS/Redis access control. Use whenever writing or reviewing Go code that touches HTTP handlers, auth, user or message input, SQL, files, outbound requests, secrets, crypto, or service configuration, even if the user does not mention security.
metadata:
  origin: ECC
---

# Go Service Security

Secure-by-default patterns for Go services (net/http, pgx, go-redis, NATS). Every rule here exists because the insecure version is the easy one to write; the goal is to make the safe version the habit.

## When to Use

- Adding or changing HTTP handlers, middleware, or message consumers
- Authentication, authorization, sessions, API keys, webhooks
- Building SQL dynamically, running external commands, touching the filesystem with user-supplied names
- Making outbound HTTP calls to URLs that come from users or data
- Loading configuration and secrets, logging request data
- Generating tokens, hashing passwords, encrypting data
- Writing Dockerfiles or connecting to Postgres, NATS, Redis
- Reviewing a diff before merge (use the checklist reference)

## Reference Files

- `references/checklist.md` - review checklist by area with severities and a pre-deploy list. Read it when reviewing a diff or auditing a service.
- `references/http-hardening.md` - complete server setup, body limits, security headers, CORS, CSRF, client IP behind proxies, rate limiting, TLS and mTLS, SSRF-safe HTTP client. Read it when building or reviewing the HTTP layer or outbound clients.
- `references/infrastructure.md` - secrets in Docker images, NATS auth and permissions, Redis ACLs, CI supply chain. Read it when touching Dockerfiles, compose files, broker/cache connections, or CI.

## Principles

1. **Validate at the boundary, trust inside.** HTTP requests, broker messages, files, env vars, and responses from other services are all input. Convert them into validated domain types once, so the core never sees raw input.
2. **Least privilege everywhere.** DB role, NATS user, Redis ACL user, container user - each gets only what this service needs, so one compromised service cannot reach the rest.
3. **Fail closed.** If a token cannot be verified, a permission cannot be looked up, or a config value is missing, refuse - do not fall through to "allowed".
4. **Errors inward, generic messages outward.** Details go to logs; clients get a stable error code.
5. **Let tools catch the mechanical bugs.** govulncheck, gosec, and `go test -race` in CI; humans review logic (authz, trust boundaries).

## 1. Secrets

Load secrets from the environment or a mounted secret file (Docker/Kubernetes secrets, Vault agent) at startup, and fail fast if one is missing - a service that starts with an empty signing key is worse than one that does not start.

```go
type Secret string

// LogValue keeps the secret out of slog output even when a whole config struct is logged.
func (Secret) LogValue() slog.Value { return slog.StringValue("[REDACTED]") }

// String and GoString cover fmt verbs (%s, %v, %+v, %#v) and accidental error wrapping.
func (Secret) String() string   { return "[REDACTED]" }
func (Secret) GoString() string { return "[REDACTED]" }

// Reveal is the single, greppable place where the raw value leaves the type.
func (s Secret) Reveal() string { return string(s) }

func requireSecret(key string) (Secret, error) {
	if path := os.Getenv(key + "_FILE"); path != "" { // Docker/K8s secret mounts
		b, err := os.ReadFile(path)
		if err != nil {
			return "", fmt.Errorf("read %s_FILE: %w", key, err)
		}
		return Secret(strings.TrimSpace(string(b))), nil
	}
	v := os.Getenv(key)
	if v == "" {
		return "", fmt.Errorf("%s is not set", key)
	}
	return Secret(v), nil
}
```

- Never log request headers wholesale (`Authorization`, `Cookie`), full DSNs (they embed passwords), or request bodies on auth endpoints. Use a `slog.HandlerOptions.ReplaceAttr` as a second net for keys like `password`, `token`, `authorization`.
- Rotate anything that was ever committed; deleting it from HEAD does not remove it from history.
- `.env` files belong in `.gitignore` and `.dockerignore`; commit `.env.example` with placeholder values only.

## 2. Input Validation

Decode into a request struct, then validate explicitly. Reject rather than "fix" bad input - silent normalization hides attacks and bugs.

```go
type CreateUserRequest struct {
	Email string `json:"email"`
	Name  string `json:"name"`
	Role  string `json:"role"`
}

var allowedRoles = map[string]bool{"member": true, "viewer": true} // "admin" is not self-assignable

func (r CreateUserRequest) Validate() error {
	var errs []error
	if _, err := mail.ParseAddress(r.Email); err != nil || len(r.Email) > 254 {
		errs = append(errs, errors.New("email: invalid"))
	}
	if n := utf8.RuneCountInString(r.Name); n == 0 || n > 100 {
		errs = append(errs, errors.New("name: must be 1-100 characters"))
	}
	if !allowedRoles[r.Role] {
		errs = append(errs, errors.New("role: not allowed"))
	}
	return errors.Join(errs...)
}
```

- Bound everything: body size (`http.MaxBytesReader`), string lengths, slice lengths, page sizes, numeric ranges. Unbounded input is a DoS vector even when it is not an injection vector.
- Parse IDs into their real type (`uuid.Parse`, `strconv.ParseInt`) at the boundary; a typed ID cannot carry SQL or path fragments.
- Never bind fields like `role`, `owner_id`, `tenant_id`, `is_admin` from the request body - derive them from the authenticated principal (mass-assignment).
- Integer conversions from input (`int64` to `int32`, `int` to `uint`) can overflow silently; range-check before converting (gosec G115).
- Use `html/template`, never `text/template`, for anything rendered as HTML; it escapes by context.
- Do not decode untrusted input with `encoding/gob`; it is not designed for hostile data. `encoding/xml` does not resolve external entities, so XXE is not a concern, but still limit the input size.

## 3. SQL Injection (pgx)

Pass every value as a parameter. pgx sends parameters separately from the SQL text, so they can never change the query structure.

```go
// Wrong: user input becomes SQL.
q := fmt.Sprintf("SELECT id, email FROM users WHERE email = '%s'", email)

// Right.
row := pool.QueryRow(ctx, "SELECT id, email FROM users WHERE email = $1", email)

// Lists: one array parameter instead of building "IN ($1, $2, ...)".
rows, err := pool.Query(ctx, "SELECT id, email FROM users WHERE id = ANY($1)", ids)
```

Identifiers and keywords (ORDER BY column, direction, table names) cannot be parameters. Map user input through an allowlist to a constant string; never pass the raw value through.

```go
var userSortColumns = map[string]string{
	"created": "created_at",
	"email":   "email",
}

func listUsersQuery(sort string, desc bool) (string, error) {
	col, ok := userSortColumns[sort]
	if !ok {
		return "", fmt.Errorf("unsupported sort field %q", sort)
	}
	dir := "ASC"
	if desc {
		dir = "DESC"
	}
	// col and dir come only from constants above, so formatting is safe here.
	return fmt.Sprintf("SELECT id, email FROM users ORDER BY %s %s, id LIMIT $1 OFFSET $2", col, dir), nil
}
```

When an identifier genuinely comes from configuration or metadata (e.g., per-tenant schema), quote it with `pgx.Identifier{schema, "users"}.Sanitize()` - and still validate it against a known set.

- `LIKE`/`ILIKE` patterns: user input containing `%` or `_` changes the match; escape them (`strings.NewReplacer(`\`, `\\`, "%", `\%`, "_", `\_`)`) and pass the pattern as a parameter.
- Give the service's DB role only the privileges it needs (no superuser, no DDL at runtime; migrations run with a separate role). See the postgres-patterns and database-migrations skills.

## 4. Command Injection

`exec.Command` does not invoke a shell, so arguments are not interpreted - keep it that way.

```go
// Wrong: the shell interprets ; | $() in filename.
bad := exec.CommandContext(ctx, "sh", "-c", "convert "+filename+" out.png")

// Right: fixed binary, input as discrete arguments, "--" so a name like "-rf" is not parsed as a flag.
cmd := exec.CommandContext(ctx, "convert", "--", filename, outPath)
```

- Never let input choose the program name. Validate arguments against an allowlist or a strict pattern even without a shell - argument injection (`--output=/etc/...`) is still possible.
- Always use `CommandContext` with a timeout so a hostile input cannot hang the worker.
- Prefer a Go library over shelling out when one exists.

## 5. Path Traversal

Any filesystem path built from input can escape the intended directory via `../`, absolute paths, or symlinks. `filepath.Join` and `filepath.Clean` do not prevent this.

```go
// Go 1.24+: os.Root confines all operations, including symlink resolution, to one directory.
root, err := os.OpenRoot(uploadDir)
if err != nil {
	return err
}
defer root.Close()

f, err := root.Open(name) // fails if name escapes uploadDir
```

- Before Go 1.24, or when only validating a name: `filepath.IsLocal(name)` rejects absolute paths, `..` escapes, empty names, and (on Windows) reserved device names. It does not handle symlinks, so prefer `os.Root` for actual I/O.
- Apply the same check to entries from zip/tar archives ("zip slip"); never extract to `filepath.Join(dst, hdr.Name)` without it.
- Best of all: do not use client-supplied names on disk. Store files under a generated ID and keep the original name as metadata.
- For serving static files, `http.FileServerFS(os.DirFS(dir))` already rejects `..`; never hand-roll path handling around `http.ServeFile` with a raw query parameter.

## 6. SSRF

If a URL or host comes from a user (webhooks, image fetch, "import from URL"), the service can be turned into a proxy into the internal network, cloud metadata (`169.254.169.254`), or local admin ports.

- Prefer an allowlist of hosts when the set of destinations is known.
- When it is not, block private/loopback/link-local destinations in the dialer's `Control` hook. It runs after DNS resolution on the actual IP being connected to, so it also defeats DNS rebinding and redirects - unlike checking the hostname up front.
- Disable proxies on that client (`Transport.Proxy = nil`); with a proxy, the dialer only sees the proxy address.
- Allow only `http`/`https` schemes, set timeouts, cap response size, and limit redirects.

The complete client is in `references/http-hardening.md`.

## 7. Authentication and Authorization

### JWT verification

Most JWT bugs come from trusting the token header. Pin the algorithm, require expiry, and check issuer and audience.

```go
// github.com/golang-jwt/jwt/v5
claims := &jwt.RegisteredClaims{}
_, err := jwt.ParseWithClaims(raw, claims,
	func(t *jwt.Token) (any, error) {
		kid, _ := t.Header["kid"].(string)
		return keys.PublicKey(kid) // from JWKS cache; error if unknown kid
	},
	jwt.WithValidMethods([]string{"RS256"}), // blocks "none" and RS/HS confusion
	jwt.WithIssuer(cfg.Issuer),
	jwt.WithAudience(cfg.Audience),
	jwt.WithExpirationRequired(),
	jwt.WithLeeway(30*time.Second),
)
if err != nil {
	return nil, ErrUnauthenticated // do not echo err to the client
}
```

- Never accept the key or algorithm from the token itself (`jwk`, `jku`, `x5u` headers).
- Keep access tokens short-lived; revocation needs either short TTLs or a denylist (e.g., in Redis keyed by `jti`).

### Secret comparison

`==` on strings returns early at the first differing byte, which leaks how much of a guess was correct. Use constant-time comparison for API keys, webhook signatures, and CSRF tokens.

```go
func secretsEqual(got, want string) bool {
	// Hashing first makes both inputs equal length, so the length is not leaked either.
	g := sha256.Sum256([]byte(got))
	w := sha256.Sum256([]byte(want))
	return subtle.ConstantTimeCompare(g[:], w[:]) == 1
}

// HMAC webhook signatures: hmac.Equal is constant-time.
mac := hmac.New(sha256.New, webhookKey)
mac.Write(body) // hash.Hash.Write never returns an error
if !hmac.Equal(mac.Sum(nil), gotSig) {
	return ErrBadSignature
}
```

Store API keys hashed (SHA-256 is fine for high-entropy random keys), look up by a non-secret prefix, then compare.

### Authorization

Authentication says who; authorization must be checked on every operation, on the server, for the specific resource.

- Scope queries by the principal so a guessed ID returns nothing (IDOR): `UPDATE orders SET ... WHERE id = $1 AND owner_id = $2`, then treat 0 rows affected as not found.
- Put the check in the application/service layer, not only in route middleware - the same use case is often reachable from HTTP, a consumer, and a job.
- Deny by default: a new route without an explicit policy should be rejected in review.
- Returning 404 instead of 403 for other tenants' resources avoids confirming that they exist.

## 8. Cryptography

- **Tokens, IDs, nonces, reset codes:** `crypto/rand` only. `math/rand` and `math/rand/v2` are predictable. Go 1.24+: `rand.Text()` returns a 128-bit random base32 string, and `rand.Read` never returns an error.
- **Passwords:** `golang.org/x/crypto/bcrypt` (cost >= 12; it rejects inputs over 72 bytes with `ErrPasswordTooLong` - enforce a max length at validation) or `golang.org/x/crypto/argon2` `IDKey` with a random 16-byte salt (RFC 9106 second recommendation: time=3, memory=64 MiB, threads=4). Never plain SHA-256/MD5 for passwords - they are fast by design.
- **Encryption:** AES-256-GCM via `crypto/cipher` with a fresh random nonce per message, or `golang.org/x/crypto/nacl/secretbox`. Never reuse a nonce with the same key. Do not invent schemes.
- MD5/SHA-1 are acceptable only for non-security checksums and cache keys; gosec flags them, so annotate legitimate uses with `//nolint:gosec // non-security checksum`.

```go
func HashPassword(pw string) (string, error) {
	h, err := bcrypt.GenerateFromPassword([]byte(pw), 12)
	if err != nil {
		return "", fmt.Errorf("hash password: %w", err)
	}
	return string(h), nil
}

func CheckPassword(hash, pw string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(pw)) == nil
}
```

## 9. HTTP Server Hardening

A zero-value `http.Server` has no timeouts, so a client can hold connections open indefinitely (Slowloris). Always set them, and limit body size per handler.

```go
srv := &http.Server{
	Addr:              cfg.Addr,
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       15 * time.Second,
	WriteTimeout:      30 * time.Second,
	IdleTimeout:       120 * time.Second,
	MaxHeaderBytes:    1 << 20,
}
```

- `r.Body = http.MaxBytesReader(w, r.Body, 1<<20)` before decoding; map `*http.MaxBytesError` to 413.
- Do not import `net/http/pprof` in a binary that serves `http.DefaultServeMux` publicly - the import registers debug handlers as a side effect. Serve pprof and metrics on a separate, internal-only listener.
- CORS: allowlist exact origins; never reflect the request `Origin` while allowing credentials.
- Cookie-authenticated, state-changing endpoints need CSRF protection: Go 1.25+ `http.NewCrossOriginProtection()`; set cookies `HttpOnly`, `Secure`, `SameSite=Lax` or `Strict`. Bearer-token APIs are not exposed to CSRF.
- Security headers, rate limiting, trusted client IP, and TLS config: see `references/http-hardening.md`.

## 10. Errors That Do Not Leak

```go
func (h *Handler) writeError(w http.ResponseWriter, r *http.Request, err error) {
	switch {
	case errors.Is(err, domain.ErrNotFound):
		writeJSON(w, http.StatusNotFound, apiError{Code: "not_found"})
	case errors.Is(err, domain.ErrForbidden):
		writeJSON(w, http.StatusForbidden, apiError{Code: "forbidden"})
	default:
		// Full error (SQL, file paths, stack context) goes to logs only.
		h.log.ErrorContext(r.Context(), "request failed", "err", err, "path", r.URL.Path)
		writeJSON(w, http.StatusInternalServerError, apiError{Code: "internal"})
	}
}
```

Never return `err.Error()` from pgx, os, or upstream clients to the caller: it reveals table names, constraint names, file paths, and internal hostnames. Login failures should not distinguish "unknown user" from "wrong password". See the error-handling skill for the full error model.

## 11. Race Conditions Are Security Bugs

Check-then-act across two statements lets concurrent requests double-spend balances, redeem a coupon twice, or bypass a quota.

```go
// Wrong: two requests both pass the check.
// SELECT balance ...; if balance >= amount { UPDATE ... }

// Right: the condition and the write are one atomic statement.
tag, err := tx.Exec(ctx,
	`UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1`,
	amount, accountID)
if err != nil {
	return fmt.Errorf("debit: %w", err)
}
if tag.RowsAffected() == 0 {
	return ErrInsufficientFunds
}
```

- Alternatives: `SELECT ... FOR UPDATE` inside a transaction, unique constraints for one-time actions, idempotency keys for retried requests and redelivered messages.
- Data races on shared maps or auth caches can corrupt state or crash the process; run `go test -race` in CI.
- TOCTOU on files: check-then-open is racy; open first (via `os.Root`), then inspect the handle.

## 12. Brokers, Caches, and Containers

- **Message payloads are untrusted input.** Validate a NATS message exactly like an HTTP body: size, schema, types, enums. A compromised or buggy producer must not be able to corrupt this service's data. Identity comes from NATS-level auth and subject permissions, not from a `user_id` field inside the payload - unless that payload is signed and you verify the signature.
- **NATS:** authenticate each service with its own nkey or `.creds` file, restrict publish/subscribe permissions per subject, use TLS outside a single host.
- **Redis:** require AUTH with a per-service ACL user restricted to its key prefix and to non-dangerous commands; never expose the port publicly. Treat cached data as input when deserializing it.
- **Docker:** no secrets in `ENV`, `ARG`, or copied files - they persist in image layers. Use BuildKit secret mounts and runtime secrets; run as non-root on a minimal base image.

Details, configs, and examples: `references/infrastructure.md`. See also the nats-jetstream-patterns, redis-patterns, and docker-patterns skills.

## 13. Dependency and Static Scanning

```bash
govulncheck ./...        # known CVEs, only reports vulnerabilities in code paths you actually call
gosec ./...              # static analysis: injection, weak crypto, file perms, missing timeouts
go mod verify            # module cache matches go.sum hashes
go test -race ./...
```

- Install: `go install golang.org/x/vuln/cmd/govulncheck@latest` and `go install github.com/securego/gosec/v2/cmd/gosec@latest`. In golangci-lint v2, enable the `gosec` linter to run it with the rest of the lint suite.
- Run govulncheck in CI on every PR and on a schedule - new CVEs appear for code that has not changed. Also scan the toolchain: an old Go version ships vulnerable stdlib packages, and govulncheck reports those too.
- Commit `go.sum`; build with `-mod=readonly` (the default) in CI so builds cannot silently change dependencies.
- Suppress a gosec finding only with a reason: `//nolint:gosec // G304: path comes from os.Root-confined name`.

## Security Tests

Cover the negative paths, not just the happy path - these are the regressions that matter:

- Every protected route returns 401 without a token and 403/404 for another tenant's resource (table-driven over the route list).
- Tokens with `alg: none`, wrong `aud`/`iss`, expired `exp`, or signed with an unknown key are rejected.
- Oversized bodies return 413; invalid input returns 400 with no internal details.
- Path traversal names (`../etc/passwd`, absolute paths) and SSRF targets (`127.0.0.1`, `169.254.169.254`, `[::1]`) are rejected.
- Concurrent debit/redeem requests cannot exceed the limit (run with `-race`).

See the golang-testing skill for test structure.
