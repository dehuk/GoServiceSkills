---
name: hexagonal-architecture
description: Structure Go services as ports and adapters - cmd/<svc>/main.go composition root, internal/domain, internal/app use cases, internal/adapters/{http,postgres,nats,redis}, small consumer-owned interfaces, explicit wiring without a DI framework, a func(ctx, tx) unit of work across repositories, and mapping between transport, domain and storage models. Use whenever creating a new Go service, adding a feature that touches HTTP/Postgres/NATS/Redis, deciding where code, validation or a transaction belongs, or untangling handlers that run SQL directly - even if the user never says "hexagonal" or "clean architecture".
metadata:
  origin: ECC
---

# Hexagonal Architecture for Go Services

Ports and adapters keep business rules independent of HTTP, Postgres, NATS and Redis. The domain and use cases depend on small interfaces; adapters at the edge implement or call them; `main` wires everything together. The payoff is that use cases are testable with in-memory fakes in milliseconds, and infrastructure can change without touching business rules.

For a complete, compiling slice (domain, ports, pgx adapter with unit of work, HTTP adapter, `main`, use-case test), read `references/worked-example.md` when you are about to write or scaffold real code.

## Layout

```text
cmd/
  orders/
    main.go              # composition root: config, adapters, wiring, lifecycle
internal/
  domain/                # entities, value objects, invariants, domain errors; stdlib only
  app/                   # use cases + the port interfaces they consume
  adapters/
    http/                # inbound: handlers, request/response models, error -> status
    nats/                # inbound consumers and outbound publishers / outbox relay
    postgres/            # outbound: repositories, row models, unit of work
    redis/               # outbound: cache, rate limiter, locks
migrations/              # golang-migrate SQL files
```

Why this shape:

- `internal/` stops other modules from importing your adapters or domain, so the service's only public surface is its API and events.
- One `domain` and one `app` package per service is the right default. Split by subdomain (`internal/billing/{domain,app}`) only when a service genuinely holds several bounded contexts; premature feature folders create import cycles and empty packages.
- Adapter packages are named by technology because that is what they encapsulate. Alias on import when a name collides with the standard library (`httpadapter "…/internal/adapters/http"`).

Dependency direction is strictly inward:

```text
adapters/*  ->  app  ->  domain
cmd/<svc>   ->  everything (wiring only)
```

`domain` imports nothing from the service. `app` imports `domain` only. Adapters import `app` and `domain`, never each other. If an HTTP handler needs something a Postgres adapter does, route it through a use case. Enforce this with a `depguard` rule in golangci-lint so violations fail CI rather than code review.

## Ports: small interfaces owned by the consumer

Go interfaces are satisfied implicitly, so declare them where they are used, not where they are implemented:

- Outbound ports (`OrderRepository`, `Outbox`, `Clock`, `PaymentGateway`) live in `app`, next to the use cases that call them.
- Inbound ports are whatever the adapter needs: the HTTP package declares `type OrderService interface { Get(...); Create(...) }` with only the methods its handlers call, and `*app.OrderService` satisfies it.
- The postgres package exports concrete types (`*OrderRepo`) and never mentions the interface. Return structs, accept interfaces.

Keep interfaces narrow (one to four methods). A 20-method `Repository` forces every fake to implement 20 methods and hides which use case depends on what. Name ports by capability, not technology: `OrderRepository`, not `PostgresStore`.

Do not create an interface for something with one implementation and no test seam need (e.g. a pure domain helper). Interfaces earn their place at I/O boundaries.

## The layers

**Domain** (`internal/domain`): entities with methods that enforce invariants (`order.MarkPaid()` refuses illegal transitions), value objects, constructors that validate (`NewOrder`), and domain errors (`ErrNotFound`, `ErrConflict`, `ErrInvalid`). No `context`, no struct tags, no `pgx`, no `json`. If you feel the need for `json:"..."` tags on a domain type, you need a transport model instead.

**Application** (`internal/app`): one method or type per use case. A use case loads state through ports, calls domain methods, persists, and records events. It takes `context.Context`, returns domain types or small output structs, and returns errors wrapped with `%w` so adapters can classify them. It contains no SQL, no HTTP status codes, no NATS subjects beyond logical event names.

**Adapters** (`internal/adapters/*`): translation and I/O. Decode requests, call a use case, encode responses; scan rows into storage models and map them to domain types; translate driver errors into domain errors (`pgx.ErrNoRows` -> `domain.ErrNotFound`, unique violation -> `domain.ErrConflict`). The error-handling skill covers the full error translation chain.

## Mapping between models

Keep three models per aggregate when they diverge, and map explicitly with plain functions:

| Model | Lives in | Shaped by |
|---|---|---|
| Transport (`createOrderRequest`, `orderResponse`, generated OpenAPI types) | `adapters/http`, `adapters/nats` | the API/event contract |
| Domain (`domain.Order`) | `domain` | business rules |
| Storage (`orderRow`) | `adapters/postgres` | the schema |

Why not one struct with `json` and `db` tags: a column rename then silently changes the public API, a nullable column forces pointer fields into business code, and an internal field leaks into responses the first time someone forgets `json:"-"`. Mapping functions are boring code, which is the point; they are where contract drift becomes a compile error.

When the three really are identical and the service is small, a shared struct is acceptable. Split the moment any one of them needs to change independently.

## Where validation lives

Validate at each layer for what that layer owns, once:

1. **Adapter, syntactic**: malformed JSON, unknown fields, wrong types, missing required fields, body size limits, path/query parsing. Reject with 400 before calling the use case. If you generate from OpenAPI, request validation middleware covers much of this (see the contract-first skill).
2. **Domain, invariants**: rules that must hold regardless of entry point (total > 0, legal status transitions). Enforced in constructors and methods, so an invalid `Order` cannot exist. Returns errors wrapping `domain.ErrInvalid`.
3. **Use case, contextual**: rules needing other state (customer exists, stock available, caller owns the resource). Done through ports.

Do not re-validate the same rule in the handler "just in case": duplicated rules drift, and the domain is the one place every entry point (HTTP, NATS consumer, CLI, backfill job) goes through.

## Transactions across repositories: unit of work

Use cases often need several writes to commit atomically (update order + insert outbox row). Keep `pgx.Tx` out of `app` by exposing a transaction runner that hands the callback tx-bound repositories:

```go
// internal/app/ports.go
type Tx interface {
	Orders() OrderRepository
	Outbox() Outbox
}

type TxRunner interface {
	InTx(ctx context.Context, fn func(ctx context.Context, tx Tx) error) error
}
```

```go
// internal/app/orders.go
func (s *OrderService) MarkPaid(ctx context.Context, id domain.OrderID) error {
	return s.tx.InTx(ctx, func(ctx context.Context, tx Tx) error {
		o, err := tx.Orders().Get(ctx, id)
		if err != nil {
			return err
		}
		if err := o.MarkPaid(); err != nil {
			return err
		}
		if err := tx.Orders().Update(ctx, o); err != nil {
			return fmt.Errorf("update order %s: %w", id, err)
		}
		return tx.Outbox().Add(ctx, "orders.paid", OrderPaid{ID: string(id)})
	})
}
```

The postgres adapter implements `InTx` with `pgx.BeginFunc`, which commits on nil and rolls back on error or panic. Repositories accept a small `querier` interface satisfied by both `*pgxpool.Pool` and `pgx.Tx`, so one implementation serves both cases. Full code is in the worked example.

Rules that keep this safe:

- Only the use case decides transaction boundaries. Repositories never begin or commit; handlers never see a transaction.
- Inside `fn`, use only `tx`'s repositories. Calling a pool-backed repository from inside the callback silently escapes the transaction and can deadlock against your own row locks.
- No network calls (HTTP, NATS publish, Redis) inside `fn`. They hold the transaction and a pool connection open for their duration, and they cannot be rolled back. Publish events through a transactional outbox written in the same transaction, relayed to NATS afterwards (see the nats-jetstream-patterns skill).
- Retry the whole `InTx` on serialization failures (`40001`) or deadlocks (`40P01`) if you use `REPEATABLE READ`/`SERIALIZABLE`; `fn` must therefore be safe to re-run.
- For row-level locking (`SELECT ... FOR UPDATE`), add an explicit repository method such as `GetForUpdate`; do not hide locking inside `Get`.

## Composition root

`main` builds every concrete dependency and passes it through constructors. No DI framework, no `init()` registration, no package-level singletons:

```go
func run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	if err != nil {
		return fmt.Errorf("create pg pool: %w", err)
	}
	defer pool.Close()

	orders := app.NewOrderService(
		postgres.NewOrderRepo(pool),
		postgres.NewTxRunner(pool),
		systemClock{},
		randomIDs{},
	)

	mux := http.NewServeMux()
	httpadapter.NewHandler(orders, log).Routes(mux)

	// Start the HTTP server, NATS consumers and outbox relay; on ctx.Done()
	// shut them down in reverse order. Full version in the worked example.
	return serve(ctx, mux)
}
```

Why explicit wiring: the dependency graph is readable top to bottom, missing dependencies are compile errors instead of runtime panics, and there is no reflection or codegen step to debug. Keep `main()` a two-liner that calls `run() error` so deferred cleanup runs and errors are logged once.

When `run` grows past ~150 lines, extract `newApp(cfg Config) (*App, error)` in `cmd/<svc>` or an `internal/wire` package; still plain constructor calls. Wire (`google/wire`) or fx are not needed for a single service and add indirection.

Inject time, randomness and IDs (`Clock`, `IDGenerator`) as ports so use-case tests are deterministic.

## Testing each layer

| Layer | Test style | Speed |
|---|---|---|
| domain | Plain table-driven unit tests, no fakes | microseconds |
| app | Unit tests with hand-written fakes for ports; fake `InTx` just calls `fn` | milliseconds |
| adapters/postgres | Integration tests against real Postgres (testcontainers-go or docker compose), migrations applied; cover commit and rollback paths, constraint-to-error mapping | seconds |
| adapters/http | `httptest` with a fake service: decoding, status codes, error mapping, response shape against the contract | milliseconds |
| adapters/nats | Integration test against a real NATS server in Docker: ack/nak/term behavior, redelivery | seconds |
| end to end | A few critical flows through the real binary and infrastructure | slow, few |

Write fakes by hand as small structs with maps; they are shorter than mock-framework setup and they assert outcomes rather than call sequences. When the same port has several implementations (Postgres repo and an in-memory one), run one shared contract test suite against each. The golang-testing skill covers table-driven tests and testcontainers setup.

## Refactoring an existing service toward this

1. Pick one high-churn endpoint or consumer.
2. Add characterization tests at the HTTP/NATS level so behavior is pinned.
3. Extract a use case method; move the logic out of the handler, leaving the handler to decode, call, encode.
4. Introduce the outbound port the use case needs and make the existing DB code implement it.
5. Move wiring into `main`. Repeat slice by slice; do not rewrite the service in one change.

## When hexagonal is overkill

Skip or flatten it when:

- The service is a thin CRUD proxy over one table with no business rules. A handler plus a repository (two layers) is enough; add `app` when the first real rule shows up.
- It is a short-lived script, migration job, or prototype.
- The "domain" is a pass-through of another API (a BFF or adapter service). Model the external contract, not a domain.

Signals you have overdone it: interfaces with exactly one implementation and no fake, mapping functions that copy identical fields across three identical structs, use cases that only call one repository method, packages with a single file of type aliases. Collapse those layers; the structure should pay for itself in tests and changeability.

## Anti-patterns

- Domain types with `json`/`db` tags or imports of `pgx`, `net/http`, `nats`.
- Handlers that run SQL or publish to NATS directly.
- `*http.Request`, `pgx.Tx`, or `jetstream.Msg` passed into a use case.
- Adapters importing each other (`http` calling `postgres` directly).
- A giant `Repository` interface declared in the postgres package.
- Package-level `var db *pgxpool.Pool` or a service locator.
- Publishing to NATS inside a DB transaction and assuming they are atomic.
- Returning storage rows from use cases.

## Checklist

- [ ] `domain` imports only the standard library; `app` imports only `domain`.
- [ ] Every I/O dependency of a use case is a small interface declared in `app`.
- [ ] Transport, domain and storage models are mapped explicitly where they differ.
- [ ] Syntactic validation in adapters, invariants in domain, contextual rules in use cases.
- [ ] Multi-write use cases go through `InTx`; no network calls inside the callback.
- [ ] Events are written to an outbox in the same transaction, not published inline.
- [ ] Driver errors are translated to domain errors in the adapter.
- [ ] `main` wires everything explicitly and handles graceful shutdown.
- [ ] Use cases have fast fake-based tests; adapters have integration tests against real infrastructure.
