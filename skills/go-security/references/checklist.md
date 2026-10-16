# Go Service Security Review Checklist

Use this when reviewing a diff or auditing a service. Work through the areas the change touches; the severity table at the end says how to rank what you find.

## Secrets and Configuration

- [ ] No hardcoded keys, passwords, tokens, DSNs with credentials, or private keys in code, test fixtures that look real, compose files, or Dockerfiles
- [ ] Secrets loaded from env or mounted files at startup; service refuses to start if one is missing or empty
- [ ] Secret values wrapped in a type with `LogValue`/`String` redaction, or otherwise never passed to loggers
- [ ] No logging of `Authorization`/`Cookie` headers, full DSNs, request bodies on auth endpoints, or full broker payloads with personal data
- [ ] `.env` and key files are in `.gitignore` and `.dockerignore`

## Input Validation

- [ ] Every boundary (HTTP, NATS consumer, CLI, file import, upstream API response) decodes into a typed struct and validates it
- [ ] Body size limited (`http.MaxBytesReader`); strings, slices, page sizes, and numbers bounded
- [ ] IDs parsed into real types (UUID, int) at the boundary
- [ ] Privilege-bearing fields (`role`, `owner_id`, `tenant_id`) come from the authenticated principal, never from the body
- [ ] Integer conversions from input are range-checked
- [ ] HTML rendered with `html/template`

## Injection

- [ ] All SQL values passed as pgx parameters (`$1`, `ANY($1)`); no `fmt.Sprintf`/`+` with input in SQL
- [ ] Dynamic identifiers (ORDER BY, column, table) mapped through an allowlist to constants; `pgx.Identifier` for configured identifiers
- [ ] `LIKE` patterns escape `%` and `_`
- [ ] No `sh -c`/`bash -c` with input; `exec.CommandContext` with fixed program, discrete args, `--` before user args, timeout
- [ ] Filesystem access with input-derived names goes through `os.Root` (Go 1.24+) or is checked with `filepath.IsLocal`; archive entries checked too
- [ ] No `encoding/gob` on untrusted data

## Outbound Requests (SSRF)

- [ ] URLs from input go through an allowlist, or a client whose dialer `Control` rejects private/loopback/link-local/metadata IPs
- [ ] That client has `Proxy: nil`, timeouts, limited redirects, http/https only, capped response size
- [ ] Internal credentials and headers are never forwarded to user-supplied URLs

## Authentication

- [ ] JWT parsing pins algorithms (`WithValidMethods`), requires `exp`, checks `iss` and `aud`; keys come from configuration/JWKS, never from token headers
- [ ] API keys, webhook signatures, and tokens compared with `crypto/subtle` / `hmac.Equal`
- [ ] API keys and session tokens stored hashed
- [ ] Passwords hashed with bcrypt (cost >= 12, max length enforced) or argon2id; never fast hashes
- [ ] Login, reset, and OTP endpoints rate limited per IP and per account; responses do not reveal whether an account exists
- [ ] Cookies `HttpOnly`, `Secure`, `SameSite`; CSRF protection on cookie-authenticated state changes

## Authorization

- [ ] Every operation checks the principal's right to the specific resource, in the service layer (not only route middleware)
- [ ] Queries scoped by owner/tenant (`WHERE id = $1 AND tenant_id = $2`); 0 rows treated as not found
- [ ] New routes and consumers deny by default
- [ ] Admin-only operations are not reachable through a generic update endpoint

## Cryptography and Randomness

- [ ] Tokens, nonces, IDs, codes from `crypto/rand` (`rand.Text`, `rand.Read`); no `math/rand` for anything security-relevant
- [ ] Encryption uses AES-GCM or secretbox with unique nonces; no custom schemes, no ECB, no static IVs
- [ ] `InsecureSkipVerify` absent from production paths; `MinVersion` set to TLS 1.2 or higher
- [ ] MD5/SHA-1 only for non-security purposes, annotated

## HTTP Server

- [ ] `http.Server` sets `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
- [ ] pprof/metrics on a separate internal listener; `net/http/pprof` not imported into a public `DefaultServeMux`
- [ ] CORS allowlists exact origins; never reflects arbitrary `Origin` with credentials
- [ ] Security headers set (`nosniff`, frame denial, CSP, `no-store` for personal data)
- [ ] Client IP taken from `X-Forwarded-For` only behind trusted proxies
- [ ] Rate limits on expensive and auth endpoints; in-memory limiter maps bounded

## Errors and Logging

- [ ] Clients receive stable error codes, never `err.Error()` from DB, filesystem, or upstreams; no stack traces
- [ ] Unexpected errors logged server-side with request ID
- [ ] Security events (auth failures, permission denials, admin actions) logged for auditing

## Concurrency

- [ ] Check-then-act on balances, quotas, coupons, and one-time tokens done atomically (conditional UPDATE, `FOR UPDATE`, unique constraint)
- [ ] Retried requests and redelivered messages are idempotent
- [ ] Shared maps/caches guarded; `go test -race` passes

## Messaging, Cache, Containers

- [ ] Broker payloads validated like HTTP input; poison messages terminated, not retried forever
- [ ] Identity not taken from payload fields
- [ ] Per-service NATS credentials with subject permissions; no `$JS.API.>` for runtime services
- [ ] Redis AUTH with per-service ACL user, key prefix, `-@dangerous`; `default` user disabled
- [ ] No secrets in image layers; non-root, minimal base image; dev ports bound to `127.0.0.1`

## Dependencies

- [ ] `govulncheck ./...` clean (or findings triaged with justification)
- [ ] `gosec ./...` clean, suppressions carry a reason
- [ ] `go mod verify` passes; `go.sum` committed
- [ ] New dependencies justified and maintained; Go toolchain on a supported, patched release

## Severity Guide

| Finding | Severity |
|---------|----------|
| Hardcoded production secret, or secret committed to git | CRITICAL |
| SQL built with `fmt.Sprintf`/concatenation of input | CRITICAL |
| Shell invocation with input (`sh -c`) | CRITICAL |
| Missing authentication on a non-public route | CRITICAL |
| Missing per-resource authorization (IDOR), tenant data reachable cross-tenant | CRITICAL |
| JWT verified without algorithm pinning, or `exp`/`aud` not checked | CRITICAL |
| Plaintext or fast-hash password storage | CRITICAL |
| Check-then-act race on money, quota, or one-time tokens | CRITICAL |
| Path traversal with input-derived file names | HIGH |
| SSRF: fetching user-supplied URLs without IP/host restrictions | HIGH |
| `math/rand` for tokens, IDs, or codes | HIGH |
| `InsecureSkipVerify: true` reachable in production | HIGH |
| Non-constant-time comparison of secrets | HIGH |
| CORS reflecting any origin with credentials | HIGH |
| Known-exploitable CVE in a called code path (govulncheck) | HIGH |
| Broker payload used without validation, or identity taken from payload | HIGH |
| `http.Server` without timeouts; no body size limit | MEDIUM |
| No rate limiting on login/reset/OTP | MEDIUM |
| Internal error details returned to clients | MEDIUM |
| Secrets or personal data in logs | MEDIUM |
| pprof exposed on public listener | MEDIUM |
| Redis/NATS without per-service credentials or ACLs | MEDIUM |
| Missing security headers; container running as root | LOW |
| gosec suppression without a reason | LOW |

Raise severity when the code is internet-facing or handles payments, health, or authentication data; lower it when the code is internal-only and defense-in-depth exists - and say so in the finding.

## Common False Positives

- Placeholder values in `.env.example` or clearly fake credentials in tests
- SHA-256/MD5 used for checksums, ETags, or cache keys
- `fmt.Sprintf` in SQL where every interpolated value is a compile-time constant or an allowlisted mapping
- gosec G304 (file inclusion) when the path comes from `os.Root` or from configuration, not request input
- `math/rand` used for jitter, sampling, or load balancing

Verify the data flow before flagging: the question is always whether attacker-controlled input reaches the sink.
