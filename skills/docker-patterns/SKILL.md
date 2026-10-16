---
name: docker-patterns
description: Dockerfile and docker compose patterns for Go services - multi-stage builds with BuildKit cache mounts, static CGO-free binaries, distroless/scratch images, non-root users, shell-free healthchecks, .dockerignore, and a local development stack of Postgres, NATS JetStream and Redis with healthchecks, one-shot migrations, env files and profiles. Use whenever you create or review a Dockerfile, .dockerignore or compose file for a Go service, set up local infrastructure, wire migrations into compose, or debug why a containerized Go service will not start or become healthy - even if the user only says "containerize this" or "make it run locally".
metadata:
  origin: ECC
---

# Docker Patterns for Go Services

Go compiles to a single static binary, so the production image needs almost nothing besides that binary. Most Docker mistakes in Go services come from carrying build-time habits from other ecosystems (full OS images, shells, package managers) into the runtime image, or from compose stacks that start services before their dependencies are ready.

## When to Use

- Writing or reviewing a `Dockerfile` or `.dockerignore` for a Go service
- Setting up `compose.yaml` with Postgres, NATS JetStream and Redis for local development
- Adding healthchecks, migrations or startup ordering to compose
- Deciding whether to run the service on the host or inside compose
- Debugging slow image builds, failing healthchecks or containers that cannot reach each other

## Production Dockerfile

````dockerfile
# syntax=docker/dockerfile:1

ARG GO_VERSION=1.25

# Build on the native platform and cross-compile; Go does not need QEMU emulation.
FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS build
WORKDIR /src

# Dependency layer: only invalidated when go.mod/go.sum change.
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

# Declared late so changing VERSION does not invalidate the download step.
ARG TARGETOS TARGETARCH
ARG SERVICE=api
ARG VERSION=dev
ARG COMMIT=unknown
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -trimpath \
      -ldflags "-s -w -X main.version=${VERSION} -X main.commit=${COMMIT}" \
      -o /out/server ./cmd/${SERVICE}

# distroless/static: CA certificates, tzdata, /etc/passwd with a nonroot user, no shell.
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /app/server
USER nonroot:nonroot
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD ["/app/server", "healthcheck"]
ENTRYPOINT ["/app/server"]
````

Build it with:

```bash
docker build \
  --build-arg SERVICE=api \
  --build-arg VERSION="$(git describe --tags --always --dirty)" \
  --build-arg COMMIT="$(git rev-parse --short HEAD)" \
  -t orders-api:dev .
```

### Why each piece is there

| Choice | Reason |
|---|---|
| `# syntax=docker/dockerfile:1` | Enables BuildKit features (`--mount=type=cache`, `--mount=type=secret`) on any recent Docker. |
| Cache mounts for `/go/pkg/mod` and `/root/.cache/go-build` | Modules and compiled packages survive between builds even when the layer is invalidated. Rebuilds after a one-line change drop from minutes to seconds. These paths are the defaults (`GOMODCACHE`, `GOCACHE`) for root in the official golang images. |
| `COPY go.mod go.sum` before `COPY .` | Source edits do not re-trigger `go mod download`. |
| `--platform=$BUILDPLATFORM` + `GOOS/GOARCH` from `TARGETOS/TARGETARCH` | `docker buildx build --platform linux/amd64,linux/arm64` compiles natively for both instead of emulating the compiler, which is 5-20x faster. |
| `CGO_ENABLED=0` | Produces a fully static binary that runs on `distroless/static` or `scratch`. With cgo the binary links glibc/musl and fails at start with a confusing "no such file or directory". If you genuinely need cgo (e.g. a SQLite driver with cgo), use `gcr.io/distroless/base-debian12` and build on a glibc image. |
| `-trimpath` | Removes local filesystem paths from the binary: reproducible builds and no leakage of build-machine paths in stack traces. |
| `-ldflags "-s -w"` | Strips symbol table and DWARF, typically 25-30% smaller. Panics still print full stack traces; you lose delve/core-dump debugging, so drop `-s -w` in a debug build target if you need it. |
| `-X main.version=...` | Version and commit are injected at build time because `.git` is excluded from the build context, so Go's automatic VCS stamping (`debug.ReadBuildInfo`) is empty. Declare `var version, commit = "dev", "unknown"` in `package main`; `-X` only works on package-level string variables, and silently does nothing if the path is wrong. |
| `distroless/static:nonroot` | No shell or package manager for an attacker to use, smallest CVE surface, but still includes CA roots (needed for TLS to managed Postgres/NATS) and tzdata. |
| `USER nonroot:nonroot` | The `:nonroot` tag already defaults to UID 65532; stating it keeps the intent visible in review and survives a base-image swap. |
| Exec-form `ENTRYPOINT` | The binary is PID 1 and receives SIGTERM directly, so graceful shutdown (draining HTTP, NATS consumers, pool close) actually runs. Shell form would wrap it in `/bin/sh`, which does not exist here and would swallow signals anyway. |

Pin base images by digest in CI-produced images (`gcr.io/distroless/static-debian12:nonroot@sha256:...`) so a rebuild cannot silently change the runtime. Tags alone are fine for local development.

For the `scratch` variant, private modules via build secrets, a debug image, and building several binaries from one Dockerfile, read `references/dockerfile-variants.md`.

### Never put secrets in layers

Every `ARG`, `ENV` and `COPY` ends up in the image history (`docker history --no-trunc` shows build args). Things that look temporary - `RUN echo $TOKEN > ~/.netrc && go mod download && rm ~/.netrc` - are still in the layer. Use `RUN --mount=type=secret` for build-time credentials (see the variants reference) and inject runtime secrets via environment or mounted files at `docker run`/compose time.

## .dockerignore

Keep the build context small (faster builds) and keep secrets and local state out of `COPY . .`.

```
.git
.github
.idea
.vscode
.env
.env.*
!.env.example
*.local
compose*.yaml
docker-compose*.yml
Dockerfile*
bin/
dist/
tmp/
coverage.*
*.out
*.test
**/testdata/golden/*.tmp
vendor/
```

Remove `vendor/` from the list if the repository vendors dependencies and builds with `-mod=vendor`. Do not ignore `go.sum`, `migrations/` or embedded assets (`//go:embed` targets must be in the context).

## Healthchecks Without a Shell

Distroless and scratch images have no `curl`, `wget` or `sh`, so `HEALTHCHECK CMD curl ...` fails forever and the container is reported unhealthy. Two working approaches:

1. **Healthcheck subcommand in the same binary** (shown below). The binary calls its own `/healthz` endpoint over loopback and exits 0 or 1.
2. **Separate admin listener** (e.g. `:8081`) serving `/healthz`, `/readyz` and metrics. Use this for workers that have no public HTTP server; orchestrators like Kubernetes probe it directly and ignore Docker `HEALTHCHECK` entirely.

```go
package main

import (
	"context"
	"net/http"
	"os"
	"time"
)

var version, commit = "dev", "unknown"

func main() {
	if len(os.Args) > 1 && os.Args[1] == "healthcheck" {
		os.Exit(healthcheck())
	}
	// ... normal startup
}

// healthcheck probes the local server so Docker can check a shell-less image.
func healthcheck() int {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	port := os.Getenv("HTTP_PORT")
	if port == "" {
		port = "8080"
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://127.0.0.1:"+port+"/healthz", nil)
	if err != nil {
		return 1
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return 1
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return 1
	}
	return 0
}
```

Keep liveness and readiness separate:

- `/healthz` (liveness): the process is alive and its HTTP loop responds. Do not check Postgres/NATS/Redis here; a database blip would otherwise restart every replica at once and make the outage worse.
- `/readyz` (readiness): dependencies reachable (`pool.Ping`, `nc.Status() == nats.CONNECTED`, `rdb.Ping`), migrations applied, consumers started. Return 503 while shutting down so load balancers stop routing before the server closes.

## Local Development Stack (docker compose)

The default workflow: compose runs the infrastructure, the Go service runs on the host with `go run` (fast rebuilds, debugger, no image builds). The same compose file can also run the services in containers behind a profile, which is useful for end-to-end checks and for teammates who do not work on that service.

```yaml
# compose.yaml
name: orders

x-go-service: &go-service
  env_file: compose.env          # container-to-container addresses (postgres:5432, ...)
  restart: unless-stopped
  profiles: ["app"]              # not started by a plain `docker compose up`
  read_only: true                # a static Go binary needs no writable root filesystem
  security_opt: ["no-new-privileges:true"]
  cap_drop: [ALL]
  depends_on:
    migrate:
      condition: service_completed_successfully
    nats-migrate:
      condition: service_completed_successfully
    nats:
      condition: service_healthy
    redis:
      condition: service_healthy

services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: ${POSTGRES_DB:-orders}
    ports:
      - "127.0.0.1:${POSTGRES_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./deploy/postgres/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      # -h 127.0.0.1 forces TCP: during init scripts Postgres listens only on the
      # unix socket, so a socket check would report "ready" before init finishes.
      test: ["CMD-SHELL", "pg_isready -h 127.0.0.1 -U \"$${POSTGRES_USER}\" -d \"$${POSTGRES_DB}\""]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  migrate:
    image: migrate/migrate:v4.18.3
    volumes:
      - ./migrations:/migrations:ro
    command:
      - -path=/migrations
      - -database=postgres://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB:-orders}?sslmode=disable
      - up
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"

  nats:
    image: nats:2.14-alpine        # the default nats tag is scratch-based: no wget for the healthcheck
    command: ["-js", "-sd", "/data", "-m", "8222"]
    ports:
      - "127.0.0.1:${NATS_PORT:-4222}:4222"      # clients
      - "127.0.0.1:${NATS_MONITOR_PORT:-8222}:8222"   # monitoring: /varz, /jsz, /healthz
    volumes:
      - natsdata:/data             # JetStream store survives restarts
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:8222/healthz?js-enabled-only=true"]
      interval: 5s
      timeout: 3s
      retries: 10

  nats-migrate:                    # one-shot: creates/updates JetStream streams and consumers
    build:
      context: .
      args: { SERVICE: api, VERSION: "${VERSION:-dev}" }
    command: ["migrate-nats"]      # subcommand of the service binary, see nats-jetstream-patterns
    env_file: compose.env
    profiles: ["app"]              # on the host, run `go run ./cmd/api migrate-nats` instead
    depends_on:
      nats:
        condition: service_healthy
    restart: "no"

  redis:
    image: redis:7.4-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD:?set REDIS_PASSWORD in .env}", "--appendonly", "yes"]
    environment:
      REDISCLI_AUTH: ${REDIS_PASSWORD}   # lets redis-cli authenticate without -a on the command line
    ports:
      - "127.0.0.1:${REDIS_PORT:-6379}:6379"
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep -q PONG"]
      interval: 5s
      timeout: 3s
      retries: 10

  api:
    <<: *go-service
    build:
      context: .
      args: { SERVICE: api, VERSION: "${VERSION:-dev}" }
    ports:
      - "127.0.0.1:8080:8080"

  worker:
    <<: *go-service
    build:
      context: .
      args: { SERVICE: worker, VERSION: "${VERSION:-dev}" }

volumes:
  pgdata:
  natsdata:
  redisdata:
```

Points that matter:

- **`depends_on` with conditions.** A bare `depends_on` only orders container start, not readiness; the service then races Postgres and crashes on the first connect. `service_healthy` waits for the healthcheck, `service_completed_successfully` waits for the one-shot migration to exit 0. The service should still retry its initial connections with backoff, because in production nothing orders startup for you.
- **Migrations as a one-shot service.** Running `migrate up` from compose (and from a deploy job in production) keeps schema changes out of service startup, where several replicas would race to migrate. See the database-migrations skill for writing the migrations themselves.
- **Init scripts** in `/docker-entrypoint-initdb.d` run only when the data volume is empty. Use them for things migrations cannot do as the app user (extra databases, roles, extensions such as `pgcrypto`), not for schema. After changing them, `docker compose down -v` to re-run.
- **Postgres 18+** images moved the data directory; mount the volume at `/var/lib/postgresql` instead of `/var/lib/postgresql/data` when you upgrade, and do not reuse a volume across major versions (use `pg_upgrade` or dump/restore).
- **Ports bound to `127.0.0.1`.** `"5432:5432"` publishes on every interface, so anyone on the same Wi-Fi can reach a database with a dev password. Docker's published ports also bypass host firewalls such as ufw.
- **Redis password and AOF.** A password-protected Redis matches production client config, so missing `Password` in the Go client is caught locally. `appendonly yes` keeps data across restarts when Redis holds rate-limit counters or locks you are debugging.
- **NATS JetStream.** `-js` enables JetStream, `-sd /data` puts the store on the named volume, `-m 8222` exposes monitoring. Streams and consumers are created by the service's `migrate-nats` subcommand, run as the one-shot `nats-migrate` service (or `go run ./cmd/api migrate-nats` on the host) before the app starts, and by a pre-deploy Job in production. Never by hand and never on service startup; see the nats-jetstream-patterns skill.

### Env files: three different things

Compose has two unrelated mechanisms with similar names, and the Go service needs a third:

| File | Read by | Purpose |
|---|---|---|
| `.env` (project dir, gitignored) | Compose itself | Variable interpolation in `compose.yaml` (`${POSTGRES_PASSWORD}`). Not passed into containers. |
| `compose.env` (`env_file:`) | Containers | Service config with in-network hostnames: `DATABASE_URL=postgres://app:...@postgres:5432/orders`, `NATS_URL=nats://nats:4222`, `REDIS_ADDR=redis:6379`. |
| `.env.local` (gitignored) | The service on your host | Same keys, but `localhost:5432`, `localhost:4222`, `localhost:6379`. |

Commit a `.env.example` with every key and dummy values so a new checkout starts with `cp .env.example .env`. Keep real credentials out of all committed files.

### Running against the stack

```bash
# Infrastructure only, wait until healthy, then migrate.
docker compose up -d --wait postgres nats redis
docker compose run --rm migrate

# Service on the host (debugger, fast rebuilds).
set -a; . ./.env.local; set +a
go run ./cmd/api migrate-nats      # JetStream streams and consumers
go run ./cmd/api

# Everything in containers, including the Go services.
docker compose --profile app up --build

# Reset all local data (destructive: deletes the named volumes).
docker compose down -v
```

Hot reload with `air` (on the host or in a dev-stage container), compose overrides for debugging, and test-only stacks are in `references/dev-workflow.md`. Read it when the user asks for live reload, a containerized dev environment, or running integration tests against compose.

## Debugging

```bash
docker compose ps                               # state and health of each service
docker compose logs -f --tail=100 api           # follow one service
docker inspect --format '{{json .State.Health}}' orders-api-1   # healthcheck output and failures
docker compose exec postgres psql -U app -d orders
docker compose exec redis redis-cli             # REDISCLI_AUTH already set
curl -s localhost:8222/jsz?streams=true         # JetStream streams and consumers
docker run --rm -it --network orders_default natsio/nats-box nats -s nats://nats:4222 stream ls
```

A distroless container has no shell to `exec` into. To inspect it, attach a debug container to its namespaces: `docker debug <container>` (Docker Desktop) or `docker run --rm -it --pid=container:<id> --network=container:<id> busybox sh`. Alternatively build a `:debug` variant (see `references/dockerfile-variants.md`).

Common failures:

| Symptom | Cause |
|---|---|
| `exec /app/server: no such file or directory` | Binary built with cgo (dynamic linking) on a static base image, or built for the wrong `GOARCH`. |
| `x509: certificate signed by unknown authority` | `scratch` image without CA certificates. |
| `unknown time zone Europe/...` | `scratch` without tzdata; add `import _ "time/tzdata"` or copy zoneinfo. |
| `connection refused` to `localhost:5432` from a container | Inside a container `localhost` is the container itself; use the service name `postgres`. |
| Container unhealthy forever, app works | `HEALTHCHECK` uses `curl`/`wget`/`sh` that do not exist in the image. |
| Graceful shutdown never runs, `docker stop` takes 10s | Shell-form `ENTRYPOINT`/`CMD`, or the app ignores SIGTERM. |
| Init SQL changes have no effect | Data volume already initialized; `docker compose down -v`. |

## Anti-Patterns

- Shipping the `golang` image as the runtime: ~800 MB, a full toolchain and shell for an attacker, hundreds of scanner findings.
- `:latest` tags for Postgres/NATS/Redis: a teammate's pull silently upgrades a major version and the existing volume no longer starts.
- Running migrations inside service startup with several replicas: concurrent migrators and slow cold starts.
- Secrets in `ENV`, `ARG` or committed `compose.yaml`; put them in gitignored `.env` locally and a secret manager in production.
- Using compose as the production orchestrator for multi-host deployments; it has no rescheduling or rolling updates.
- Bind-mounting the source into a container and running `go build` there on macOS when `go run` on the host would do; file-sync overhead makes every rebuild slower.
- Running as root "because it works locally": the same image goes to production.
