# Infrastructure Security for Go Services

Secrets in containers, NATS and Redis access control, database connections, and CI supply chain. The idea throughout: each service gets its own identity with the smallest set of permissions it needs, so a compromise of one service does not become a compromise of all of them.

## Contents

- Secrets in Docker images and compose
- PostgreSQL connections
- NATS authentication and permissions
- Validating broker messages
- Redis AUTH and ACLs
- CI and supply chain
- Pre-deploy infrastructure checklist

## Secrets in Docker Images and Compose

Anything in `ENV`, `ARG`, or a `COPY`-ed file is stored in an image layer and readable by anyone who can pull the image, even if a later layer deletes it. `docker history` shows `ARG` values.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
# Private modules: the credential is mounted only for this RUN step and never written to a layer.
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    --mount=type=cache,target=/go/pkg/mod \
    go mod download && go mod verify
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

# Minimal runtime: no shell, no package manager, runs as non-root (uid 65532).
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Build with `docker build --secret id=netrc,src=$HOME/.netrc .`.

- `.dockerignore` must exclude `.env`, `*.pem`, `*.key`, `secrets/`, `.git` - `COPY . .` otherwise ships them.
- Provide runtime secrets as files, not environment variables where possible: env vars leak via `docker inspect`, crash dumps, `/proc/<pid>/environ`, and child processes. The `requireSecret` helper in the main skill reads `<NAME>_FILE` for this reason.

```yaml
services:
  orders:
    build: .
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]

  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    ports:
      - "127.0.0.1:5432:5432" # local dev only; a bare "5432:5432" listens on all interfaces

secrets:
  db_password:
    file: ./secrets/db_password.txt # git-ignored
```

Docker-published ports bypass host firewalls such as ufw, so binding to `127.0.0.1` is the only reliable way to keep dev databases, Redis, and NATS off the network. Scan built images (e.g., Trivy or Grype) in CI. See the docker-patterns skill for build and compose structure.

## PostgreSQL Connections

- Never expose the database publicly; reach it over a private network.
- Use `sslmode=verify-full` (with the CA configured) outside a single host; `require` encrypts but does not verify the server, so it does not stop impersonation.
- One role per service, owning only its schema; runtime role has DML only. Migrations run with a separate role that has DDL rights.
- Never log the DSN - it contains the password. Log host and database name separately if needed.
- Set `statement_timeout` for the runtime role so one hostile query cannot pin a connection indefinitely.

## NATS Authentication and Permissions

Give each service its own user (nkey or decentralized JWT `.creds` file) and restrict which subjects it can publish and subscribe to. Without permissions, any service - or anyone holding any credential - can read every event and publish forged ones.

```go
// github.com/nats-io/nats.go
nc, err := nats.Connect(cfg.NATSURL,
	nats.Name("orders-service"),
	nats.UserCredentials(cfg.NATSCredsFile),     // JWT + nkey seed issued for this service only
	nats.RootCAs(cfg.NATSCAFile),                // verify the server certificate
	nats.CustomInboxPrefix("_INBOX_orders"),     // lets permissions isolate this service's replies
)
if err != nil {
	return fmt.Errorf("connect nats: %w", err)
}
```

For a static server config with nkeys (operator/JWT mode expresses the same permissions in the account and user JWTs):

```
authorization {
  users = [
    {
      nkey: "U..." # public nkey of the orders service; the seed stays with the service
      permissions: {
        publish: {
          allow: [
            "orders.events.>",
            "$JS.API.CONSUMER.INFO.PAYMENTS.orders-payments",
            "$JS.API.CONSUMER.MSG.NEXT.PAYMENTS.orders-payments",
            "$JS.ACK.PAYMENTS.orders-payments.>"
          ]
        }
        subscribe: {
          allow: ["_INBOX_orders.>"]
        }
      }
    }
  ]
}
```

- Create streams and consumers at deploy time with a separate admin identity; runtime services should not hold `$JS.API.>` (that allows deleting every stream).
- The exact JetStream API subjects a client needs depend on which calls it makes. Start strict, watch the server log for permission violations, and add only what is needed.
- Keep `.creds` files and nkey seeds out of images; mount them as secrets. A seed is a private key.
- Enable TLS whenever traffic leaves a single host.

See the nats-jetstream-patterns skill for stream and consumer design.

## Validating Broker Messages

A message from the broker is input from another process that may be buggy, outdated, or compromised. Validate it exactly like an HTTP body.

```go
func (c *PaymentConsumer) handle(ctx context.Context, msg jetstream.Msg) {
	var evt PaymentSettled
	// Do not use DisallowUnknownFields for events: producers add fields over time,
	// and consumers must tolerate them.
	if err := json.Unmarshal(msg.Data(), &evt); err != nil {
		c.terminate(ctx, msg, "malformed payload", err)
		return
	}
	if err := evt.Validate(); err != nil { // IDs parse, amounts in range, enums known
		c.terminate(ctx, msg, "invalid payload", err)
		return
	}
	// Business authorization still applies: does this order exist, is the state transition legal?
	if err := c.orders.MarkPaid(ctx, evt.OrderID, evt.Amount); err != nil {
		c.log.WarnContext(ctx, "mark paid failed, will redeliver", "err", err)
		if nakErr := msg.Nak(); nakErr != nil {
			c.log.ErrorContext(ctx, "nak failed", "err", nakErr)
		}
		return
	}
	if err := msg.Ack(); err != nil {
		c.log.ErrorContext(ctx, "ack failed", "err", err)
	}
}

// terminate stops redelivery of a message that can never succeed (poison message).
func (c *PaymentConsumer) terminate(ctx context.Context, msg jetstream.Msg, reason string, cause error) {
	c.log.ErrorContext(ctx, reason, "err", cause, "subject", msg.Subject())
	if err := msg.Term(); err != nil {
		c.log.ErrorContext(ctx, "term failed", "err", err)
	}
}
```

- Do not take identity or privilege from payload fields (`user_id`, `role`, `is_admin`). Trust comes from which identity could publish on that subject (enforced by NATS permissions), or from a signature you verify.
- Do not log full payloads at info level; they often contain personal data.
- Handle redelivery idempotently (see the race-conditions section of the main skill); an attacker or a bug can replay messages.
- The server's `max_payload` (default 1 MB) bounds message size; keep it as small as your largest legitimate message.

## Redis AUTH and ACLs

Redis without AUTH on a reachable port is a remote-code-execution risk (e.g., via `CONFIG SET` / `MODULE LOAD`), not just a data leak.

```go
// github.com/redis/go-redis/v9
rdb := redis.NewClient(&redis.Options{
	Addr:      cfg.RedisAddr,
	Username:  "orders-svc",
	Password:  cfg.RedisPassword.Reveal(),
	TLSConfig: &tls.Config{MinVersion: tls.VersionTLS12}, // when traffic leaves the host
})
```

Per-service ACL user (in an `aclfile` or via `ACL SETUSER`):

```
user default off
user orders-svc on >long-random-password ~orders:* resetchannels &orders:* -@all +@read +@write +@connection +@transaction -@dangerous
```

- Disable the `default` user, and give each service a key prefix (`~orders:*`) and channel pattern.
- `-@dangerous` removes `FLUSHALL`, `CONFIG`, `KEYS`, `DEBUG`, `MODULE` and similar. If the service uses Lua scripts, add `+@scripting` deliberately.
- Build keys from validated, typed values (`fmt.Sprintf("orders:session:%s", sessionID)` where `sessionID` is a parsed UUID). Raw user strings containing `:` can collide with other key namespaces.
- Data read back from Redis is input: validate it when deserializing, and never cache authorization decisions longer than the permission itself is valid.
- Do not store raw session tokens as keys; store a hash, so a dump of Redis does not hand out live sessions.

See the redis-patterns skill for cache and locking patterns.

## CI and Supply Chain

- Run `govulncheck ./...`, `gosec ./...` (or golangci-lint with `gosec` enabled), `go mod verify`, and `go test -race ./...` on every PR; run govulncheck on a schedule too.
- Scan for committed secrets (e.g., gitleaks or trufflehog) and fail the build on findings.
- Give CI tokens minimal permissions (GitHub Actions: `permissions: contents: read` by default) and prefer OIDC federation over long-lived cloud keys stored as CI secrets.
- Pin third-party CI actions to a commit SHA, not a mutable tag.
- Require review and passing checks before merge; protect the main branch.
- Keep the Go toolchain current: the `go` / `toolchain` lines in `go.mod` decide which stdlib (with its security fixes) gets compiled in.
- Review new dependencies before adding them: maintenance, transitive dependency count, and whether a few lines of code would do instead.

## Pre-Deploy Infrastructure Checklist

- [ ] No secrets in images, `ENV`/`ARG`, compose files, or git history
- [ ] Container runs as non-root on a minimal base, read-only root filesystem, all capabilities dropped
- [ ] Postgres, Redis, NATS not reachable from the public internet; dev ports bound to `127.0.0.1`
- [ ] Per-service credentials for Postgres, NATS, Redis with least privilege; `default` Redis user disabled
- [ ] TLS with verification for traffic leaving a host (`sslmode=verify-full`, NATS/Redis TLS)
- [ ] Image and dependency scans clean in CI; secret scanning enabled
- [ ] Backups automated, encrypted, and a restore has actually been tested
- [ ] Security-relevant events (login failures, permission denials, admin actions) logged with request IDs and alerting in place
