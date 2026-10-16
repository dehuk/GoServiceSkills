---
name: codebase-onboarding
description: Explore an unfamiliar Go service repository and produce a structured onboarding guide (module layout, entry points, dependency wiring, HTTP routes, NATS subjects, database migrations, config/env, local infrastructure, build/test commands) plus a starter CLAUDE.md. Use whenever the user opens a Go repo for the first time, asks "how does this service work", "where is X handled", "onboard me", "walk me through this repo", or wants a CLAUDE.md generated -- even if they do not explicitly ask for an onboarding guide.
metadata:
  origin: ECC
---

# Go Service Onboarding

Systematically map an unfamiliar Go service and produce a guide a developer can scan in two minutes. Gather signals with cheap commands and targeted searches first; read files only where signals are ambiguous. Reading every file wastes context and still misses the structure.

## Phase 1: Reconnaissance

Run these in parallel where possible.

**Module and packages**
```bash
cat go.mod                                   # module path, Go version, toolchain, key deps
ls go.work 2>/dev/null                       # multi-module workspace?
go list ./...                                # every package
go list -f '{{.ImportPath}} {{.Name}}' ./... | grep ' main$'   # binaries (package main)
go list -deps -f '{{if not .Standard}}{{.ImportPath}}{{end}}' ./cmd/... | grep -v "$(go list -m)" | sort -u | head -50
```

`go list` requires dependencies to be resolvable; if it fails offline, fall back to `find . -name '*.go' -not -path './vendor/*'` and `grep -r '^package main' --include=*.go`.

**Directory snapshot** (top two levels, skip `vendor/`, `.git/`, `bin/`, generated output)

Common Go service layout signals:

| Path | Usually means |
|---|---|
| `cmd/<name>/main.go` | one binary per directory (API server, worker, migrator, CLI) |
| `internal/` | private application code; its subdirectories reveal the architecture |
| `internal/domain`, `internal/app`, `internal/adapter`, `internal/port` | hexagonal / clean architecture (see **hexagonal-architecture**) |
| `internal/handler`, `internal/service`, `internal/repository` | layered architecture |
| `pkg/` | code intended for import by other modules (or legacy habit) |
| `api/`, `openapi/`, `contracts/`, `proto/` | API or event contracts; check if code is generated from them |
| `migrations/`, `db/migrations/` | SQL migrations (golang-migrate: `NNNN_name.up.sql` / `.down.sql`) |
| `deploy/`, `k8s/`, `helm/`, `.github/workflows/` | deployment and CI |

**Build, run, and tooling**
- `Makefile`, `Taskfile.yml`, `justfile`, `magefile.go` -- list targets; these are the real commands, prefer them over guessing `go` invocations
- `docker-compose*.yml` / `compose*.yaml` -- local Postgres, Redis, NATS, and their ports
- `Dockerfile` -- build flags, which binary ships, base image
- `.golangci.yml` -- enabled linters (note `version: "2"` for golangci-lint v2)
- `//go:generate` directives: `grep -rn 'go:generate' --include=*.go .` -- sqlc, mockgen, oapi-codegen, stringer; generated files must not be edited by hand
- `.env.example`, `config/*.yaml` -- configuration surface

**Tests**
- `find . -name '*_test.go' | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head` -- where tests concentrate
- Build tags for integration tests: `grep -rln '//go:build integration' --include=*_test.go .`
- testcontainers usage: `grep -rl 'testcontainers' --include=*.go .`
- `testdata/` directories and golden files

## Phase 2: Architecture Mapping

### Entry points and wiring

Read each `cmd/*/main.go` (and any `run`, `app`, or `server` package it delegates to). `main` is the composition root: it shows every dependency the service has and how they connect. Capture:

- Config loading (env vars, flags, files; library such as `envconfig`, `caarlos0/env`, `viper`)
- Infrastructure clients: `pgxpool.New`, `redis.NewClient`, `nats.Connect` + `jetstream.New`, HTTP clients
- Construction order: repositories -> use cases/services -> handlers/consumers
- Servers and workers started, and how they shut down (signal handling, `context` cancellation, `errgroup`, `http.Server.Shutdown`)

### HTTP surface

```bash
grep -rnE 'HandleFunc\(|Handle\(|\.(Get|Post|Put|Patch|Delete)\(' --include=*.go internal cmd | grep -v _test.go
```

Go 1.22+ `net/http` patterns look like `mux.HandleFunc("GET /users/{id}", h)`; chi/echo/gin use method helpers. Note the middleware chain (auth, logging, recovery, request ID, timeouts) and where errors are mapped to status codes. If an OpenAPI spec exists, check whether handlers are generated from it.

### Messaging surface

```bash
grep -rnE 'Publish(Async)?\(|Subscribe\(|CreateOrUpdateStream|CreateOrUpdateConsumer|Consume\(|Fetch\(' --include=*.go internal cmd | grep -v _test.go
grep -rnE '"[a-z0-9_-]+(\.[a-z0-9_*>-]+)+"' --include=*.go internal | grep -iE 'subject|stream|nats' | head -40
```

List streams, subjects published, consumers (durable names, filter subjects, ack policy), and which handler processes each. Note whether an outbox is used between DB writes and publishes.

### Data layer

- Migrations: count, latest version, which tables matter most
- Query style: hand-written SQL with pgx, sqlc-generated code, or an ORM
- Transaction helper and where transactions begin (use-case layer vs repository)
- Redis usage: caching, rate limiting, locks, sessions (`grep -rn 'redis\.' --include=*.go internal`)

### Request lifecycle

Trace one representative request end to end and write it down with file references:

```text
GET /orders/{id}
  cmd/api/main.go:48              route registration
  internal/transport/http/mw.go   auth, request ID, timeout
  internal/transport/http/order_handler.go:31   decode, validate
  internal/app/order/get.go:20    use case
  internal/adapter/postgres/order_repo.go:55    SQL query
  internal/transport/http/errors.go:12          error -> status mapping
```

Do the same for one event consumer if the service consumes messages.

### Package docs

`go doc ./internal/<pkg>` and `go doc -all ./internal/<pkg>` give a fast overview of exported API without opening files. Package doc comments (`doc.go`) often explain intent.

## Phase 3: Convention Detection

- **Errors**: sentinel errors (`var ErrNotFound = errors.New(...)`), custom types, wrapping with `%w`, where errors are logged vs returned (see **error-handling**)
- **Logging**: `log/slog`, zap, or zerolog; structured keys; logger passed via context or struct field
- **Context**: passed as first parameter everywhere? Timeouts set where?
- **Interfaces**: defined by consumer (port) or by implementer; size
- **Testing**: table-driven, `t.Run` naming, fakes vs mockgen/gomock, testify vs stdlib assertions, integration test tags
- **Naming**: package names, file naming (`order_handler.go` vs `handler/order.go`), constructor style (`NewX` returning struct or interface)
- **Git**: branch naming and commit style from `git log --oneline -20`; if history is empty or shallow, say "Git history unavailable or too shallow to detect conventions"

When the code contradicts a README or config, trust the code and note the discrepancy.

## Phase 4: Generate Artifacts

### Output 1: Onboarding Guide (print to conversation)

````markdown
# Onboarding Guide: [module path]

## Overview
[2-3 sentences: what the service does, what it owns, who calls it]

## Tech Stack
| Layer | Technology | Version |
|---|---|---|
| Language | Go | [from go.mod] |
| HTTP | [net/http ServeMux / chi / ...] | |
| Database | PostgreSQL via pgx/v5 | |
| Migrations | golang-migrate | |
| Cache | Redis via go-redis/v9 | |
| Messaging | NATS JetStream | |
| Testing | [stdlib / testify, testcontainers] | |

## Binaries
| Binary | Path | Purpose |
|---|---|---|

## Architecture
[Layers and how they connect; one short diagram]

## HTTP Routes
| Method + path | Handler | Notes |
|---|---|---|

## Events
| Subject | Direction | Stream / consumer | Handler |
|---|---|---|---|

## Request Lifecycle
[One traced request with file:line references]

## Configuration
| Env var | Purpose | Default |
|---|---|---|

## Local Development
- Start infra: `docker compose up -d`
- Migrate: [detected command]
- Run: [detected command]
- Test: [unit and integration commands]
- Lint: [detected command]

## Conventions
- [errors, logging, tests, naming]

## Where to Look
| I want to... | Look at... |
|---|---|
| Add an endpoint | |
| Add a use case | |
| Add a table | |
| Publish / consume an event | |
| Add a config value | |

## Unknowns
- [Anything that could not be determined confidently]
````

### Output 2: Starter CLAUDE.md

If `CLAUDE.md` exists, read it first and enhance it: preserve existing instructions and state clearly what you added or changed. Keep it under ~100 lines; it is loaded into every session, so every line costs context.

````markdown
# Project Instructions

## Stack
[One-line summary: Go version, pgx, NATS JetStream, Redis, ...]

## Commands
- Build: `[cmd]`
- Test: `go test -race ./...` [plus integration command]
- Lint: `golangci-lint run`
- Local infra: `docker compose up -d`
- Migrations: `[cmd]`
- Codegen: `go generate ./...` [if used; never edit generated files]

## Layout
[Key directory -> purpose]

## Conventions
- [Error handling pattern]
- [Logging pattern]
- [Test style and where tests live]
- [Commit style if detectable]
````

Ask before writing CLAUDE.md to disk unless the user already asked for it.

## Best Practices

1. **Commands over reading**: `go list`, `go doc`, and `grep` reveal structure faster and cheaper than opening files.
2. **Start at `main`**: the composition root is the most information-dense file in a Go service.
3. **Prefer the Makefile/Taskfile**: they encode flags, build tags, and env that ad-hoc `go` commands miss.
4. **Verify, do not guess**: a dependency in `go.mod` may be unused or used only in tests; confirm with `grep` before listing it as part of the stack.
5. **Flag unknowns**: "Could not determine how migrations run in CI" beats a confident wrong answer.
6. **Stay concise**: highlight dependencies that shape how code is written, not every module in `go.sum`.

## Examples

- **"Onboard me to this repo"**: run all four phases; print the guide; offer to write CLAUDE.md.
- **"Generate a CLAUDE.md"**: run phases 1-3; write only CLAUDE.md.
- **"Where are orders cancelled?"**: run the relevant parts of phase 1-2 (routes, subjects, grep for the use case) and answer with a traced path rather than a full guide.
