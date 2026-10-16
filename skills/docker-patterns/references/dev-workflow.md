# Local Development Workflow with Compose

Extends the compose stack in SKILL.md: hot reload, debugging overrides, and stacks for integration tests.

## Choose Where the Service Runs

| Run the Go service... | Good for | Cost |
|---|---|---|
| On the host (`go run`, IDE debugger) against compose infra | Daily development of the service you are changing | Host env file with `localhost` addresses |
| In compose via `--profile app` | End-to-end checks of the real image, running services you are not working on | Image rebuild per change (seconds with cache mounts) |
| In compose with `air` (dev stage) | Teams that want a single `docker compose up` for everything | Slower file watching on macOS/Windows bind mounts |

Default to the first. Containerized hot reload mostly adds a layer of indirection between you and the compiler.

## Hot Reload on the Host with air

```bash
go install github.com/air-verse/air@latest   # pin a version in a tools file for the team
air -c .air.toml
```

```toml
# .air.toml
root = "."
tmp_dir = "tmp"

[build]
  cmd = "go build -o ./tmp/api ./cmd/api"
  entrypoint = ["./tmp/api"]
  include_ext = ["go", "sql", "yaml", "tmpl"]
  exclude_dir = ["tmp", "vendor", "testdata", "node_modules"]
  exclude_regex = ["_test\\.go$"]
  delay = 300
  kill_delay = "2s"        # time for graceful shutdown before SIGKILL
  send_interrupt = true    # SIGINT first, so the service drains connections

[log]
  time = false
```

Load `.env.local` into the shell before starting `air` so the binary sees host addresses. `tmp/` must be in `.gitignore` and `.dockerignore`.

## Hot Reload Inside Compose

Add a `dev` stage to the Dockerfile. It keeps the full toolchain and is never used for release.

```dockerfile
FROM golang:1.25-alpine AS dev
WORKDIR /src
RUN go install github.com/air-verse/air@latest
ENV GOFLAGS=-buildvcs=false
CMD ["air", "-c", ".air.toml"]
```

`-buildvcs=false` avoids "error obtaining VCS status" when the bind-mounted checkout belongs to a different UID than the container user.

```yaml
# compose.dev.yaml - use with: docker compose -f compose.yaml -f compose.dev.yaml --profile app up
services:
  api:
    build:
      target: dev
    read_only: false            # air writes to ./tmp and the Go caches
    volumes:
      - .:/src
      - gomod:/go/pkg/mod       # named volumes keep caches across container restarts
      - gobuild:/root/.cache/go-build
    environment:
      HTTP_PORT: "8080"

volumes:
  gomod:
  gobuild:
```

On macOS and Windows, file events across bind mounts can be delayed or lost. If air misses changes, set `poll = true` and `poll_interval = 500` under `[build]` in `.air.toml`.

## Debugging Overrides

Expose delve from a container only when you need it:

```yaml
# compose.debug.yaml
services:
  api:
    build:
      target: dev
    command: ["dlv", "debug", "./cmd/api", "--headless", "--listen=:2345", "--api-version=2", "--accept-multiclient", "--continue"]
    ports:
      - "127.0.0.1:2345:2345"
    security_opt: ["seccomp:unconfined"]   # delve needs ptrace
    cap_add: [SYS_PTRACE]
```

The dev stage needs `go install github.com/go-delve/delve/cmd/dlv@latest`. Debugging a service on the host is usually simpler; use this only when the bug depends on the container environment.

## Stacks for Integration Tests

Two options, matching the golang-testing skill:

- **testcontainers-go** starts disposable Postgres/NATS/Redis per test package. Best for CI and for tests that need clean state; no shared compose state between runs.
- **A dedicated compose project** for tests that reuse long-lived infra locally. Give it its own project name and ports so it never touches development data:

```bash
docker compose -p orders-test -f compose.yaml -f compose.test.yaml up -d --wait postgres nats redis
docker compose -p orders-test -f compose.yaml -f compose.test.yaml run --rm migrate
go test -race -tags=integration ./...
docker compose -p orders-test -f compose.yaml -f compose.test.yaml down -v
```

When both dev and test stacks run at once, override published ports via `.env` variables (`POSTGRES_PORT=55432`) rather than editing `compose.yaml`.

The `compose.test.yaml` override makes the test database faster with `tmpfs` storage and durability off; never use these settings for data you care about:

```yaml
# compose.test.yaml
services:
  postgres:
    command: ["postgres", "-c", "fsync=off", "-c", "synchronous_commit=off", "-c", "full_page_writes=off"]
    volumes: !reset []          # drop the named volume (Compose 2.24+); tmpfs cannot share its mount point
    tmpfs:
      - /var/lib/postgresql/data
```

## Compose Profiles Summary

- Infrastructure services have no profile: `docker compose up` starts only them.
- Go services use `profiles: ["app"]`.
- Optional tools (pgAdmin, RedisInsight, nats-box, an OpenTelemetry collector + Jaeger) use `profiles: ["tools"]`, so they are opt-in with `--profile tools`.
- A service whose dependency sits in an inactive profile may fail to start or start without it, depending on how it was invoked; keeping infrastructure profile-free avoids that class of surprise.
