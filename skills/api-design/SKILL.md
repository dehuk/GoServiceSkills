---
name: api-design
description: HTTP/REST API design for Go services — resource naming, methods and status codes, cursor vs offset pagination, filtering and sorting, RFC 9457 problem+json errors, idempotency keys, versioning, request validation, timeouts, Go 1.22+ ServeMux handlers, strict JSON decoding, and OpenAPI. Use whenever designing, implementing, or reviewing an HTTP endpoint, handler, request/response struct, or API contract in a Go service, even if the user only asks to "add an endpoint" and does not mention API design.
metadata:
  origin: ECC
---

# API Design for Go HTTP Services

Conventions for consistent, predictable HTTP APIs in Go services. These are recommended defaults.

**If the project has its own API conventions document, it takes precedence — read it first.** Look for API, error-code, pagination, or filtering specs in the repository or a linked specification repo before applying anything below. Where a project spec and this skill disagree, follow the spec and mention the difference only if it looks like a mistake.

For the full, compile-checked helper code (JSON helpers, strict decoding, handler, cursor, server setup), read `references/go-http-handlers.md` when you write or review that code.

## Resources and URLs

```
GET    /v1/users                 list
POST   /v1/users                 create
GET    /v1/users/{id}            read
PUT    /v1/users/{id}            full replace
PATCH  /v1/users/{id}            partial update
DELETE /v1/users/{id}            delete
GET    /v1/users/{id}/orders     sub-resource (ownership)
POST   /v1/orders/{id}/cancel    action that is not CRUD (use sparingly)
```

- Plural nouns, lowercase, kebab-case for multi-word segments (`/team-members`). Verbs in paths (`/getUsers`) duplicate what the method already says.
- Nest only for true ownership, and at most one level; deeper nesting makes URLs brittle when relationships change.
- Use one JSON field naming style across all services (`snake_case` or `camelCase`) — consistency matters more than the choice.
- IDs in URLs are opaque strings to clients (UUIDs are a good default); do not expose sequential IDs on public APIs, they leak volume and invite enumeration.

## Methods and Status Codes

| Method | Safe | Idempotent | Typical success |
|--------|------|-----------|-----------------|
| GET | yes | yes | 200 |
| POST | no | no (make it so with an idempotency key) | 201 + `Location`, or 202 for async work |
| PUT | no | yes | 200 with body, or 204 |
| PATCH | no | not inherently | 200 with body, or 204 |
| DELETE | no | yes | 204 (also on repeated delete, or 404 — pick one and document it) |

Client errors:

| Code | Use |
|------|-----|
| 400 | Malformed request: bad JSON, wrong types, unknown fields, bad query params |
| 401 | Missing or invalid credentials |
| 403 | Authenticated but not allowed |
| 404 | Not found — also when the caller may not know the resource exists, so 403 does not leak existence |
| 405 | Method not allowed (ServeMux sends this with an `Allow` header automatically) |
| 409 | State conflict: duplicate unique key, version mismatch, idempotency key in flight |
| 412 | `If-Match` precondition failed (optimistic concurrency) |
| 413 | Body too large |
| 415 | Wrong `Content-Type` |
| 422 | Well-formed but semantically invalid (field validation) |
| 429 | Rate limited; include `Retry-After` |

Server errors: 500 for unexpected failures (log the cause, return a generic body), 502/504 for upstream failure/timeout, 503 with `Retry-After` for overload or shutdown.

The 400-vs-422 split is a common default; some specs use 400 for all validation errors. Follow the project spec if it picks one.

Mistakes to avoid: 200 with `{"success": false}`; 500 for validation errors; 200 for creation instead of 201 + `Location`; leaking SQL or Go error strings in 5xx bodies.

## Error Format

Default to RFC 9457 Problem Details (`application/problem+json`), which obsoletes RFC 7807. **A project-specific error spec overrides this** — if the project defines its own error envelope and error codes, use those exactly.

```json
{
  "type": "https://example.com/problems/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "2 fields are invalid",
  "instance": "/v1/users",
  "errors": [
    {"field": "email", "code": "invalid_format", "message": "must be a valid email address"},
    {"field": "name", "code": "out_of_range", "message": "must be 1-100 characters"}
  ]
}
```

- `type` is a stable identifier clients can switch on; `about:blank` is valid when the status code says it all. `title` is fixed per `type`; `detail` is occurrence-specific.
- Extra members (`errors`, a machine-readable `code`) are allowed as extensions.
- Return all field errors at once so clients fix everything in one round trip.
- Map domain errors to HTTP in the handler only. Services and repositories return domain errors (`ErrNotFound`, `ErrConflict`); see the **error-handling** skill.

## Handlers on the Go 1.22+ ServeMux

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /v1/users", h.list)
mux.HandleFunc("POST /v1/users", h.create)
mux.HandleFunc("GET /v1/users/{id}", h.get)        // r.PathValue("id")
mux.HandleFunc("GET /static/{path...}", h.static) // trailing wildcard matches the rest
mux.HandleFunc("GET /{$}", h.root)                // exact "/" only
```

- Method + wildcard patterns remove the need for a router library in most services. `GET` patterns also match `HEAD`.
- The most specific pattern wins; two patterns that overlap with neither more specific panic at registration, which is a feature — it catches ambiguous routes at startup.
- Keep handlers thin: decode, validate, call one service method, map the result. Business logic in handlers cannot be reused from event consumers or tested without HTTP. See the **hexagonal-architecture** skill.
- Apply cross-cutting concerns (request ID, logging, recovery, auth, metrics) as `func(http.Handler) http.Handler` middleware wrapped around the mux.

## Request Decoding and Validation

Decode strictly; the default `json.NewDecoder(r.Body).Decode(&v)` is too permissive for an API boundary:

- Wrap the body with `http.MaxBytesReader(w, r.Body, limit)` — without a limit one request can exhaust memory. Map `*http.MaxBytesError` to 413.
- Call `dec.DisallowUnknownFields()` so typos fail loudly instead of being silently ignored. (Relax this only for endpoints that must accept forward-compatible payloads, and document it.)
- Reject trailing data with a second `Decode` that must return `io.EOF`.
- Map `*json.SyntaxError`, `*json.UnmarshalTypeError`, and empty body to 400 with a safe message; never echo arbitrary `err.Error()` text.
- Validate after decoding, in a `validate()` method on the request struct (or a validation library), collecting all field errors. Validate query and path parameters too — parse `limit` with `strconv.Atoi` and bound it.
- Use separate request/response structs per endpoint rather than decoding into domain or database types. Decoding straight into a domain struct lets clients set fields they should not (`role`, `owner_id`) — mass assignment.

### JSON encoding pitfalls

- `omitempty` does not omit zero structs, so a zero `time.Time` serializes as `"0001-01-01T00:00:00Z"`. Go 1.24 adds `omitzero`, which omits zero values and respects an `IsZero() bool` method: ``DeletedAt time.Time `json:"deleted_at,omitzero"` ``. Prefer it for time and struct fields.
- Nil slices encode as `null`, empty slices as `[]`. For list responses initialize to an empty slice so clients always get an array.
- `int64` IDs above 2^53 lose precision in JavaScript clients; send them as strings (`json:",string"`) or use string IDs.
- Use pointers (or a nullable wrapper) only when the API must distinguish "absent" from "zero", typically in PATCH bodies.
- Set `Content-Type` before `WriteHeader`; headers written after the status line are silently dropped. Encoding errors after `WriteHeader` can only be logged.

## Pagination

Every list endpoint is paginated from day one with a default and a maximum `limit`; adding pagination later is a breaking change.

| | Offset (`?limit=20&offset=40` or `page`) | Cursor / keyset (`?limit=20&cursor=...`) |
|---|---|---|
| Jump to page N | yes | no |
| Cost on deep pages | grows with offset (DB scans skipped rows) | constant with a matching index |
| Concurrent inserts | rows skipped or duplicated | stable |
| Total count | natural, but `COUNT(*)` is expensive on big tables | usually omitted |
| Good for | small admin lists, search UIs with page numbers | feeds, sync, large or growing tables, public APIs |

Default to cursor pagination. Make the cursor opaque (base64url of the last row's sort key plus `id` as tiebreaker), validate it as untrusted input (400 on garbage), and fetch `limit+1` rows to compute `has_more`:

```json
{
  "data": [{"id": "u_123", "name": "Alice"}],
  "next_cursor": "eyJjIjoiMjAyNi0wMS0xNVQxMDozMDowMFoiLCJpIjoidV8xMjMifQ",
  "has_more": true
}
```

A cursor is only valid for the sort order it was created with; reject it if the client changes `sort`. Keyset SQL and cursor code are in the reference file.

## Filtering and Sorting

```
GET /v1/orders?status=active&customer_id=c_42
GET /v1/orders?status=active,pending              multiple values
GET /v1/orders?created_after=2026-01-01T00:00:00Z range bounds as explicit params
GET /v1/orders?sort=-created_at,id                "-" prefix for descending
GET /v1/products?q=wireless+headphones            full-text search
```

- Whitelist filter and sort fields and map each to a SQL column in code. Never interpolate a client-provided field name into SQL; placeholders cannot parameterize identifiers, so a whitelist map is the only safe option.
- Always end the `ORDER BY` with a unique column (`id`) so ordering — and pagination — is deterministic.
- Return 400 for unknown filter or sort fields rather than ignoring them; silently ignored filters return wrong data that looks right.
- Bracket operators (`price[gte]=10`) and other syntaxes are fine if the project spec defines them; pick one style for all services.

## Idempotency Keys

Network retries turn non-idempotent POSTs into duplicate orders and payments. Accept an `Idempotency-Key` header (client-generated UUID) on such endpoints:

1. On receipt, atomically claim `(key, caller)`: `INSERT ... ON CONFLICT DO NOTHING` in Postgres, or `SET key value NX EX ttl` in Redis.
2. If the key is new: process the request, then store the status code and response body with the key.
3. If the key exists with a stored response: return that response unchanged.
4. If the key exists but is still in progress: return 409 so the client retries later.
5. If the key is reused with a different request body (compare a hash of it): return 422 — it is a client bug.
6. Expire keys after a documented window (commonly 24 hours).

Scope keys per caller so one client cannot replay another's response. Storing the key in the same Postgres transaction as the business write gives the strongest guarantee. See the **postgres-patterns** and **redis-patterns** skills for the storage side.

For updates, use optimistic concurrency: return an `ETag` (row version) and require `If-Match` on PUT/PATCH, answering 412 on mismatch. This prevents lost updates between concurrent editors.

## Versioning

- Default to a major version in the path (`/v1/...`): explicit, easy to route and log. Header or media-type versioning works but is easier to forget in clients and tests.
- Additive changes are not breaking: new endpoints, new optional request fields, new response fields. Clients must ignore unknown response fields — say so in the API docs.
- Breaking changes need a new major version: removing or renaming fields, changing types or semantics, new required inputs, changed status codes or error codes.
- Run at most two major versions at once. Announce deprecation, send `Deprecation` and `Sunset` headers on the old version, and remove it after the sunset date.
- For internal service-to-service APIs, prefer evolving compatibly over versioning: deploy the tolerant reader first, then the new writer.

## Timeouts

Every HTTP server and client needs explicit timeouts; the zero values mean "wait forever".

- Server: set `ReadHeaderTimeout` (slowloris protection), `ReadTimeout`, `WriteTimeout` (longer than the slowest handler), and `IdleTimeout`. Shut down gracefully with `srv.Shutdown(ctx)` on SIGTERM so in-flight requests finish.
- Per request: derive `context.WithTimeout(r.Context(), budget)` and pass it to every pgx, Redis, NATS, and outbound HTTP call so the whole request aborts together. Map `context.DeadlineExceeded` from upstreams to 504 or 503.
- Clients: never use `http.DefaultClient` (no timeout). Build an `*http.Client` with `Timeout` and a tuned `Transport`, always close response bodies, and retry only idempotent requests with backoff and a cap.

## Security Basics

- Every endpoint is authenticated unless explicitly marked public. Use `Authorization: Bearer <token>`; never put credentials in query strings, which end up in logs.
- Authorize per resource in the service that owns the data (`order.CustomerID == caller.ID`), not only by role at the gateway.
- Rate-limit by caller and return 429 with `Retry-After`.
- Never return internal details (SQL errors, stack traces, Go type names) in responses. See the **go-security** skill.

## OpenAPI

- Keep an OpenAPI 3.x document per service in the repo as the contract. Write it first and generate server interfaces/types from it (for example with `oapi-codegen`), or keep hand-written handlers and validate the spec against them in CI — either way, the spec must not drift. The **contract-first** skill covers the workflow.
- Document every status code an endpoint can return, the error schema, pagination parameters, and the auth scheme.
- Run a breaking-change diff on the spec in CI so accidental breaking changes are caught before merge.

## Checklist

- [ ] Project API conventions document checked and followed
- [ ] Plural, kebab-case resource paths; no verbs except explicit actions
- [ ] Correct method and status codes; 201 + `Location` on create
- [ ] Body size limit, unknown-field rejection, trailing-data check
- [ ] All field errors returned together; error format matches the project spec or RFC 9457
- [ ] Separate request/response structs; no mass assignment
- [ ] List endpoints paginated with default and max `limit`; deterministic ordering
- [ ] Filter/sort fields whitelisted
- [ ] Retryable POSTs accept `Idempotency-Key`; updates use `If-Match` where races matter
- [ ] Server, per-request, and client timeouts set; context passed downstream
- [ ] Authentication and per-resource authorization enforced
- [ ] No internal error details in responses
- [ ] OpenAPI spec updated; breaking-change check passes
