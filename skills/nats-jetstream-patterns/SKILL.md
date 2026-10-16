---
name: nats-jetstream-patterns
description: Patterns for Go services that use NATS and JetStream through github.com/nats-io/nats.go and its jetstream package (jetstream.New, CreateOrUpdateStream, CreateOrUpdateConsumer, Consume/Messages/Fetch). Covers connection setup and shutdown, subject naming, stream and consumer design, Ack/Nak/Term semantics, dead-letter handling, publishing with dedup and optimistic concurrency, idempotent consumers, transactional outbox, KV/Object Store, testing with an embedded server, and debugging with the nats CLI. Use whenever Go code publishes or consumes NATS messages, defines streams or consumers, or handles events between services, even if the user only says "message broker", "events", "queue" or "pub/sub". Also use when you see the legacy nc.JetStream()/js.Subscribe API so you can migrate it.
---

# NATS and JetStream Patterns for Go

Recommended defaults for Go services on NATS. They favour at-least-once delivery with idempotent handlers, explicit acks, and infrastructure that is declared once and reviewed like code.

## When to Activate

- Writing or reviewing code that imports `github.com/nats-io/nats.go` or `github.com/nats-io/nats.go/jetstream`
- Designing subjects, streams, consumers, or event payloads
- Debugging stuck consumers, redelivery storms, lost or duplicated messages
- Adding tests for publishers or consumers

## Use the new `jetstream` package, not the legacy API

Most blog posts and older examples use the legacy JetStream API on `nats.JetStreamContext`: `nc.JetStream()`, `js.Subscribe`, `js.QueueSubscribe`, `js.PullSubscribe`, `js.AddStream`, `nats.Durable(...)` subscribe options. That API is still shipped but is superseded by `github.com/nats-io/nats.go/jetstream`, which separates stream/consumer management from consumption and makes pull consumers the default.

| Legacy (avoid in new code) | New `jetstream` package |
|---|---|
| `js, _ := nc.JetStream()` | `js, err := jetstream.New(nc)` |
| `js.AddStream(&nats.StreamConfig{...})` | `js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{...})` |
| `js.PullSubscribe(subj, durable)` + `sub.Fetch` | `js.CreateOrUpdateConsumer(ctx, stream, cfg)` then `cons.Fetch` / `cons.Consume` / `cons.Messages` |
| `js.Subscribe` / `js.QueueSubscribe` (push) | Durable pull consumer shared by all instances |
| `js.Publish(subj, data, nats.MsgId(id))` | `js.Publish(ctx, subj, data, jetstream.WithMsgID(id))` |

When you find legacy code, do not mix both APIs for the same consumer; migrate the whole flow. Every `jetstream` call takes a `context.Context`, so carry deadlines through.

## Connection setup

One `*nats.Conn` per process, created at startup and shared. Give it a name (shows up in `nats server report connections` and server logs), reconnect forever with jitter, and log every state change; a silent disconnect is the most common "messages stopped" incident.

```go
func Connect(ctx context.Context, url, credsFile string, log *slog.Logger) (*nats.Conn, <-chan struct{}, error) {
	closed := make(chan struct{})
	opts := []nats.Option{
		nats.Name("orders-service"),
		nats.MaxReconnects(-1),                       // never give up; the orchestrator restarts us if needed
		nats.ReconnectWait(2 * time.Second),
		nats.ReconnectJitter(500*time.Millisecond, 2*time.Second),
		nats.RetryOnFailedConnect(true),              // tolerate NATS starting after us (compose, k8s)
		nats.DrainTimeout(20 * time.Second),
		nats.DisconnectErrHandler(func(_ *nats.Conn, err error) {
			log.Warn("nats disconnected", "err", err)
		}),
		nats.ReconnectHandler(func(c *nats.Conn) {
			log.Info("nats reconnected", "url", c.ConnectedUrl())
		}),
		nats.ErrorHandler(func(_ *nats.Conn, sub *nats.Subscription, err error) {
			subj := ""
			if sub != nil {
				subj = sub.Subject
			}
			log.Error("nats async error", "subject", subj, "err", err) // slow consumers land here
		}),
		nats.ClosedHandler(func(*nats.Conn) { close(closed) }),
	}
	if credsFile != "" {
		opts = append(opts, nats.UserCredentials(credsFile)) // JWT + nkey seed in one .creds file
	}
	nc, err := nats.Connect(url, opts...)
	if err != nil {
		return nil, nil, fmt.Errorf("nats connect: %w", err)
	}
	return nc, closed, nil
}
```

Credentials: prefer a `.creds` file (`nats.UserCredentials`) or an nkey seed (`nats.NkeyOptionFromSeed(path)`, which returns `(nats.Option, error)`) mounted as a secret. Never put user/password in the URL that ends up in logs. Enable TLS (`nats.RootCAs`, `nats.ClientCert`) outside local dev.

### Shutdown: Drain with a timeout

`nc.Drain()` stops new deliveries, lets in-flight handlers finish, flushes pending publishes, then closes the connection. It returns immediately; completion is signalled by the `ClosedHandler`. Bound the wait so a stuck handler cannot block shutdown forever.

```go
// Stop consumers first so no new work starts, then drain the connection.
consumeCtx.Drain()
if err := nc.Drain(); err != nil {
	log.Error("nats drain", "err", err)
}
select {
case <-closed:
case <-time.After(25 * time.Second): // slightly above DrainTimeout
	nc.Close()
}
```

## Subjects

Use `<domain>.<entity>.<event>.<version>`, lowercase, dot-separated, no spaces:

```
orders.order.created.v1
orders.order.cancelled.v1
billing.invoice.paid.v1
```

- Put the version in the subject so a v2 can run side by side and consumers opt in with a filter.
- Wildcards: `*` matches exactly one token (`orders.order.*.v1`), `>` matches one or more trailing tokens (`orders.>`). Streams usually capture `orders.>`; consumers filter narrower.
- Do not put high-cardinality IDs in the middle of event subjects unless you need per-entity ordering or `WithExpectLastSequencePerSubject`; if you do, put the ID last (`orders.order.events.<order_id>`) so wildcards still work.
- Commands/requests use a separate namespace from events (`orders.cmd.cancel`, `orders.svc.get`), because they have different delivery semantics.

### Core NATS or JetStream?

| Need | Use |
|---|---|
| Synchronous request/reply between services, caller waits and retries itself | Core NATS `nc.Request` / `nc.QueueSubscribe` (queue group for load balancing) |
| Fire-and-forget telemetry where loss is acceptable | Core NATS publish |
| Event that must survive a consumer being down, be replayed, or be processed at least once | JetStream |
| Work queue with retries and backoff | JetStream with a durable pull consumer |

Core NATS is at-most-once: if no subscriber is connected, the message is gone. Do not "upgrade" it to reliable delivery with retries in the publisher; use JetStream.

## Stream design

```go
// Lives in Bootstrap, run by `migrate-nats` (see "Who creates streams and consumers?").
_, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
	Name:       "ORDERS",
	Subjects:   []string{"orders.>"},
	Retention:  jetstream.LimitsPolicy,
	Storage:    jetstream.FileStorage,
	Replicas:   3,                 // 1 in local dev; 3 in a clustered production setup
	MaxAge:     7 * 24 * time.Hour,
	MaxBytes:   10 << 30,
	Discard:    jetstream.DiscardOld,
	Duplicates: 2 * time.Minute,   // dedup window for Nats-Msg-Id
})
```

Retention choice:

| Policy | Messages removed when | Use for |
|---|---|---|
| `LimitsPolicy` (default) | Limits (MaxAge/MaxMsgs/MaxBytes) hit | Domain events with several independent consumers and replay needs. The safe default. |
| `InterestPolicy` | Every consumer that existed at publish time has acked | Fan-out where storage matters and replay is not needed. Messages published with no consumers are dropped. |
| `WorkQueuePolicy` | The (single) consumer acks | Job queues. Consumers' filter subjects must not overlap; one logical consumer per subject. |

Other knobs:
- `Storage`: `FileStorage` for anything that matters; `MemoryStorage` only for ephemeral, rebuildable data.
- `MaxMsgs` / `MaxBytes` / `MaxMsgsPerSubject`: always set at least one size limit so a stuck consumer cannot fill the disk. `MaxMsgsPerSubject: 1` turns a stream into "latest value per subject".
- `Duplicates`: the dedup window for `Nats-Msg-Id`. It must be longer than your longest publisher retry (including outbox relay retries).
- One stream per bounded context/domain is a good default; subjects of two streams must not overlap.

### Who creates streams and consumers?

| Option | Pros | Cons |
|---|---|---|
| Infra step (Terraform NATS provider, `nats stream add --config`, a migration-like job in CI/CD) | Reviewed, one owner, no races between replicas, service needs no admin permissions | Extra pipeline step; local dev needs a bootstrap script |
| Service startup calls `CreateOrUpdateStream`/`CreateOrUpdateConsumer` | Self-contained, easy local dev and tests | Every replica races to "update"; two services can fight over a shared stream's config; service credentials need stream-admin rights |

The rule: streams and consumers are created by a dedicated `migrate-nats` subcommand of the service binary, run before rollout in every environment, the same way database migrations run. Do not create them on service startup.

- The owning service (publisher) declares its stream; each consuming service declares only its own durable consumers.
- `migrate-nats` calls one idempotent `Bootstrap(ctx, js, topology)` function. Local dev (`go run ./cmd/svc migrate-nats` or a one-shot compose service), integration tests and production all use it.
- Before each rollout, whoever deploys runs it from the same image/version; a non-zero exit means do not roll out. The deployment mechanism is outside the service's scope: document the command's contract, do not write pipeline or manifest code.
- Only the migrator gets stream/consumer admin credentials. The running service binds with `js.Stream(ctx, name)` / `js.Consumer(ctx, stream, name)` and fails fast on `jetstream.ErrStreamNotFound` / `jetstream.ErrConsumerNotFound` with a message saying to run `migrate-nats`.
- Per-environment values (`Replicas`, limits) come from config, not separate definitions.
- `Bootstrap` never deletes. Retiring a consumer or stream, and changing immutable fields (storage, consumer `DeliverPolicy`/`AckPolicy`), are explicit expand-contract changes.

Code for the topology, the subcommand, credentials split, the handoff contract for deployment, a safe-change table, and retirement: read `references/provisioning.md` whenever you add or change a stream or consumer, or set up deployment for a service that uses JetStream.

## Consumers

Default to a **durable pull consumer** with explicit acks. Pull consumers give flow control (the client asks for work) and scale horizontally: every replica that consumes from the same durable shares the load like a queue group.

```go
// Declared in the consuming service's Bootstrap; at runtime bind with js.Consumer(ctx, "ORDERS", name).
cons, err := js.CreateOrUpdateConsumer(ctx, "ORDERS", jetstream.ConsumerConfig{
	Durable:       "billing-order-created",          // <consumer-service>-<purpose>
	FilterSubject: "orders.order.created.v1",       // or FilterSubjects: []string{...}
	AckPolicy:     jetstream.AckExplicitPolicy,
	AckWait:       30 * time.Second,                 // > p99 handler time, or call InProgress
	MaxDeliver:    8,
	BackOff:       []time.Duration{time.Second, 5 * time.Second, 30 * time.Second, 2 * time.Minute},
	MaxAckPending: 256,                              // in-flight cap across all replicas
	DeliverPolicy: jetstream.DeliverAllPolicy,
})
```

- `AckExplicitPolicy`: each message must be acked individually. `AckNone` loses messages on crash; `AckAll` acks everything up to a sequence and hides failures.
- `AckWait`: how long the server waits for an ack before redelivering. Too short causes duplicate concurrent processing; too long delays retries after a crash.
- `BackOff`: redelivery delays for ack timeouts, overriding `AckWait` per attempt. The server requires `MaxDeliver` to be greater than the number of `BackOff` entries. Note that BackOff applies to ack-timeout redeliveries; an explicit `Nak()` redelivers immediately, which is why handlers use `NakWithDelay`.
- `MaxDeliver`: always set it (the default is unlimited). Without it a poison message is redelivered forever and blocks progress up to `MaxAckPending`.
- `MaxAckPending`: back-pressure. Size it from downstream capacity (DB pool size times replicas), not "as high as possible".
- `DeliverPolicy`: `DeliverAllPolicy` for a new consumer that must see history; `DeliverNewPolicy` when only future events matter; `DeliverByStartTimePolicy` (with `OptStartTime`) to backfill from a point. It only applies when the consumer is created; changing it later does not rewind an existing durable.
- Name durables after the consuming service and purpose. Two different purposes in the same service get two durables, so one slow handler does not hold back the other.

### Consuming

`cons.Consume` is the default: it keeps pull requests flowing, handles heartbeats, and calls your handler per message. Stop it on shutdown.

```go
cc, err := cons.Consume(h.handle,
	jetstream.PullMaxMessages(64),
	jetstream.ConsumeErrHandler(func(_ jetstream.ConsumeContext, err error) {
		log.Warn("consume error", "err", err) // missed heartbeats, consumer deleted, etc.
	}),
)
if err != nil {
	return fmt.Errorf("consume: %w", err)
}
defer cc.Stop()
```

- The handler has no `context.Context` parameter; derive one from your service's root context with a timeout below `AckWait`.
- Messages from one `Consume` are handled one at a time. For concurrency, start N `Consume` calls on the same consumer or use `cons.Messages()` (an iterator with `Next()`) feeding a bounded worker pool. Concurrency across all replicas is still capped by `MaxAckPending`.
- `cons.Fetch(n, jetstream.FetchMaxWait(d))` returns a batch (`batch.Messages()` channel, then check `batch.Error()`). Use it for cron-style batch jobs, not long-running workers.
- **Ordered consumers** (`js.OrderedConsumer(ctx, stream, jetstream.OrderedConsumerConfig{...})`) are ephemeral, unacked, single-instance, and recreate themselves on gaps. Use them for replay, projections rebuilt from scratch, and tooling, never for work that needs acks.

Full worker with error mapping and graceful stop: read `references/consumer-worker.md` when implementing a consumer.

## Ack semantics

| Call | Effect | Use when |
|---|---|---|
| `msg.Ack()` | Done, never redelivered to this consumer | Processed successfully, or a duplicate you already processed |
| `msg.DoubleAck(ctx)` | Ack and wait for server confirmation | The side effect must not run twice and a lost ack is expensive |
| `msg.Nak()` | Redeliver immediately | Rarely; it causes hot loops |
| `msg.NakWithDelay(d)` | Redeliver after `d` | Transient failure: DB down, timeout, 5xx from a dependency, lock conflict |
| `msg.InProgress()` | Reset the AckWait timer | Long handler that is still making progress; call periodically |
| `msg.Term()` / `msg.TermWithReason(r)` | Never redeliver | Poison message: cannot decode, fails validation, references an entity that will never exist |

Map error kinds, not individual errors, to these. Mark poison errors with a `Permanent(err)` wrapper and treat everything else as transient:

```go
switch {
case err == nil:
	ackErr = msg.Ack()
case IsPermanent(err): // decode/validation; retrying will never help
	h.deadLetter(ctx, msg, err)
	ackErr = msg.TermWithReason(err.Error())
default: // assume transient
	ackErr = msg.NakWithDelay(retryDelay(meta.NumDelivered))
}
```

Use `msg.Metadata()` for `NumDelivered` (attempt count), stream sequence, and `NumPending` (consumer lag at delivery time). Ack calls return errors; log them, because a failed ack means the message will come back.

### Dead letters

JetStream has no built-in DLQ. Two complementary patterns:

1. **Explicit DLQ subject.** On a terminal error, publish the original payload, headers, and the error to `dlq.<consumer>` (captured by a `DLQ` stream with long `MaxAge`), then `Term`. Also do this when `meta.NumDelivered >= MaxDeliver` in the handler, so the last failed attempt is captured with its error.
2. **MaxDeliver advisory.** When a message exhausts `MaxDeliver`, the server publishes an advisory on `$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES.<STREAM>.<CONSUMER>` containing the stream sequence. A small listener reads the original with `stream.GetMsg(ctx, seq)` and copies it to the DLQ. Advisories are core NATS messages, so capture them in a stream (subjects `$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES.>`) if the listener can be down. With `LimitsPolicy` the original stays in the stream until limits remove it; replay from the DLQ is a deliberate operation (`nats` CLI or an admin endpoint), not automatic.

Code for both is in `references/consumer-worker.md`.

## Publishing

Always publish to JetStream with `js.Publish` / `js.PublishMsg` and check the `PubAck`; a core `nc.Publish` to a stream subject gives no confirmation that the message was stored.

```go
msg := &nats.Msg{
	Subject: "orders.order.created.v1",
	Data:    body,
	Header:  nats.Header{},
}
msg.Header.Set("Content-Type", "application/json")
msg.Header.Set("X-Correlation-Id", correlationID)
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(msg.Header)) // traceparent

ack, err := js.PublishMsg(ctx, msg, jetstream.WithMsgID(event.ID)) // sets Nats-Msg-Id
if err != nil {
	return fmt.Errorf("publish order created: %w", err)
}
if ack.Duplicate {
	log.Debug("duplicate publish ignored by server", "msg_id", event.ID)
}
```

- **Dedup:** `jetstream.WithMsgID(id)` sets `Nats-Msg-Id`; the server drops repeats within the stream's `Duplicates` window and returns `PubAck.Duplicate == true`. Use a stable ID generated once per event (the outbox row ID), never a fresh UUID per retry.
- **Optimistic concurrency:** `jetstream.WithExpectLastSequencePerSubject(seq)` (per entity subject) or `jetstream.WithExpectLastSequence(seq)` (whole stream) make the publish fail if someone else wrote first. The error is a `*jetstream.APIError`; check its `ErrorCode` against `jetstream.JSErrCodeStreamWrongLastSequence` (10071) and `jetstream.JSErrCodeStreamWrongLastSequenceConstant` (10164, returned for replicated R>1 streams), reload, and retry. `jetstream.WithExpectStream(name)` guards against publishing into the wrong stream after a subject change.
- **Async publish** for throughput: `js.PublishAsync` returns a `PubAckFuture` (`Ok()` / `Err()` channels). Bound outstanding publishes with `jetstream.WithPublishAsyncMaxPending(n)` on `jetstream.New`, and wait with `<-js.PublishAsyncComplete()` (under a timeout) before reporting success or shutting down. Check every future's error; async publish errors are otherwise silent.

```go
futures := make([]jetstream.PubAckFuture, 0, len(events))
for _, ev := range events {
	f, err := js.PublishAsync(ev.Subject, ev.Data, jetstream.WithMsgID(ev.ID))
	if err != nil {
		return fmt.Errorf("publish async: %w", err)
	}
	futures = append(futures, f)
}
select {
case <-js.PublishAsyncComplete():
case <-ctx.Done():
	return ctx.Err()
}
var errs []error
for _, f := range futures {
	select {
	case <-f.Ok():
	case err := <-f.Err():
		errs = append(errs, fmt.Errorf("publish %s: %w", f.Msg().Subject, err))
	}
}
return errors.Join(errs...)
```

### Payload envelope and versioning

- Body: the event data only, JSON (or protobuf when the contract is defined there). Required fields in every event: `id` (also the `Nats-Msg-Id`), `occurred_at` (RFC 3339 UTC), and the entity ID.
- Metadata in headers, not the body: `Nats-Msg-Id`, `Content-Type`, `traceparent`, `X-Correlation-Id`, optionally `X-Event-Type`. Headers can be inspected and filtered without decoding the payload.
- Version in the subject (`.v1`). Within a version, only add optional fields; consumers ignore unknown fields. A breaking change is a new subject (`.v2`) published in parallel until every consumer has moved.
- Keep payload schemas in a contract repo or package, not duplicated per service (see contract-first).

## Exactly-once-ish processing

JetStream gives at-least-once delivery plus publish dedup. End-to-end "exactly once" effect comes from two pieces you build:

1. **Transactional outbox (publisher side).** Write the business change and an `outbox` row in the same Postgres transaction. A relay reads unpublished rows (`FOR UPDATE SKIP LOCKED`), publishes each with `WithMsgID(row.id)`, and marks it published. A crash between publish and mark causes a republish that the dedup window absorbs. Never publish to NATS inside the DB transaction; the message can escape while the transaction rolls back.
2. **Idempotent consumer (consumer side).** In the same transaction as the side effect, `INSERT INTO processed_messages (consumer, message_id) ... ON CONFLICT DO NOTHING`; if no row was inserted, the message was already handled, so ack and skip. The primary key `(consumer, message_id)` does the dedup.

Schema, relay loop, and consumer transaction code: read `references/outbox-and-idempotency.md` when implementing either side.

## KV and Object Store

- **KV** (`js.CreateOrUpdateKeyValue(ctx, jetstream.KeyValueConfig{Bucket: ..., History: ..., TTL: ...})`): `Get`, `Put`, `Create` (only if absent), `Update(ctx, key, value, revision)` for compare-and-set, and `Watch` for change feeds. Missing keys return `jetstream.ErrKeyNotFound`.
- **Object Store** (`js.CreateObjectStore`): chunked blobs (config bundles, small artifacts). Not a replacement for S3-class storage of user files.

KV vs Redis: choose NATS KV for low-volume, replicated config, feature flags, leader leases, and anything where watching changes matters and you want no extra infrastructure. Choose Redis (see redis-patterns) for high-throughput caching, rate limiting, rich data structures, and sub-millisecond hot paths.

## Testing

- **Unit tests:** hide NATS behind small ports (`EventPublisher`, a handler func that takes decoded events) and test handlers with fakes. Test the error-to-ack mapping by passing a fake `jetstream.Msg` (it is an interface).
- **Integration tests:** run an embedded server with JetStream from `github.com/nats-io/nats-server/v2/server`, storage in `t.TempDir()`, random port. Fast and needs no Docker.
- **Alternative:** testcontainers-go's NATS module, when you need the exact server version from production.
- **Local dev:** docker compose:

```yaml
services:
  nats:
    image: nats:2.14-alpine
    command: ["-js", "-sd", "/data", "-m", "8222"]  # JetStream, store dir, monitoring port
    ports: ["4222:4222", "8222:8222"]
    volumes: ["nats-data:/data"]
volumes:
  nats-data:
```

Embedded server helper, testcontainers variant, and a consumer round-trip test: read `references/testing.md` when writing tests.

## Observability and debugging

Metrics worth exporting (poll `cons.Info(ctx)` every 15-30s from one replica or an exporter such as the NATS Prometheus exporter):

- `ConsumerInfo.NumPending`: messages not yet delivered (lag). Alert on sustained growth.
- `ConsumerInfo.NumAckPending`: delivered but unacked; pinned at `MaxAckPending` means handlers are slow or stuck.
- `ConsumerInfo.NumRedelivered`: rising means transient failures or `AckWait` too short.
- Handler counters by outcome (ack / nak / term) and latency histograms, labelled by consumer.

`nats` CLI commands:

```bash
nats context add local --server nats://localhost:4222 --select
nats stream ls
nats stream info ORDERS
nats stream view ORDERS --subject orders.order.created.v1   # read messages without consuming
nats consumer ls ORDERS
nats consumer info ORDERS billing-order-created               # pending, ack pending, redelivered
nats sub 'orders.>'                                           # watch live traffic (core subscription)
nats sub '$JS.EVENT.ADVISORY.>'                               # watch advisories (max deliveries, terms)
nats pub orders.order.created.v1 '{"id":"..."}' -H Nats-Msg-Id:test-1
nats server report jetstream                                  # needs system account
```

## Checklist

- [ ] New `jetstream` package, not `nc.JetStream()`
- [ ] Connection named, infinite reconnect with jitter, handlers log disconnect/reconnect/errors
- [ ] Shutdown stops consumers, drains the connection, and bounds the wait
- [ ] Streams have size limits and a `Duplicates` window longer than publisher retries
- [ ] Streams/consumers created only by `migrate-nats` before rollout; the service binds and fails fast, never creates
- [ ] Durable pull consumer, `AckExplicit`, `MaxDeliver` set, `AckWait` above p99 handler time
- [ ] Transient errors `NakWithDelay`, poison messages `Term` plus DLQ, ack errors logged
- [ ] Publishes check `PubAck`, use a stable `Nats-Msg-Id`, carry trace and correlation headers
- [ ] Publisher uses an outbox when the event must match a DB change; consumer dedupes in the same transaction as its side effect
- [ ] Lag, ack-pending, and redelivery metrics exported
