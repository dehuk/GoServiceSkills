# Transactional Outbox and Idempotent Consumers

JetStream delivers at least once and deduplicates publishes by `Nats-Msg-Id` within a window. These two patterns close the remaining gaps so a DB change and its event are never out of sync, and a redelivered event never applies its side effect twice.

Uses `pgx/v5` (`pgxpool`) and the `jetstream` package. Schema changes go through golang-migrate like any other migration (see database-migrations).

## Schema

```sql
-- publisher service
CREATE TABLE outbox (
    id           uuid        PRIMARY KEY,           -- becomes Nats-Msg-Id
    subject      text        NOT NULL,
    payload      bytea       NOT NULL,
    headers      jsonb       NOT NULL DEFAULT '{}',
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);
CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;

-- consumer service
CREATE TABLE processed_messages (
    consumer     text        NOT NULL,
    message_id   text        NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer, message_id)
);
```

The partial index keeps the relay query cheap no matter how many published rows accumulate.

## Writing the event in the business transaction

```go
type OutboxEvent struct {
	ID      string
	Subject string
	Payload []byte
	Headers map[string]string
}

func enqueue(ctx context.Context, tx pgx.Tx, ev OutboxEvent) error {
	_, err := tx.Exec(ctx,
		`INSERT INTO outbox (id, subject, payload, headers) VALUES ($1, $2, $3, $4)`,
		ev.ID, ev.Subject, ev.Payload, ev.Headers)
	if err != nil {
		return fmt.Errorf("enqueue outbox %s: %w", ev.Subject, err)
	}
	return nil
}

func (r *OrderRepo) Create(ctx context.Context, o Order, correlationID string) error {
	payload, err := json.Marshal(OrderCreatedV1{
		ID:         o.EventID, // generated once, e.g. uuid.Must(uuid.NewV7()).String()
		OrderID:    o.ID,
		TotalCents: o.TotalCents,
		OccurredAt: o.CreatedAt.UTC(),
	})
	if err != nil {
		return fmt.Errorf("marshal event: %w", err)
	}
	return pgx.BeginFunc(ctx, r.pool, func(tx pgx.Tx) error {
		if _, err := tx.Exec(ctx,
			`INSERT INTO orders (id, customer_id, total_cents) VALUES ($1, $2, $3)`,
			o.ID, o.CustomerID, o.TotalCents); err != nil {
			return fmt.Errorf("insert order: %w", err)
		}
		return enqueue(ctx, tx, OutboxEvent{
			ID:      o.EventID,
			Subject: "orders.order.created.v1",
			Payload: payload,
			Headers: map[string]string{
				"Content-Type":     "application/json",
				"X-Correlation-Id": correlationID,
			},
		})
	})
}
```

Store the trace context (`traceparent`) in `headers` too if you want the consumer span linked to the request that created the event; inject it with the OpenTelemetry propagator into a `map[string]string` carrier (`propagation.MapCarrier`).

## Relay

```go
type Relay struct {
	Pool     *pgxpool.Pool
	JS       jetstream.JetStream
	Batch    int
	Interval time.Duration
	Log      *slog.Logger
}

func (r *Relay) Run(ctx context.Context) error {
	t := time.NewTicker(r.Interval)
	defer t.Stop()
	for {
		// Drain the backlog quickly, then fall back to polling.
		for {
			n, err := r.publishBatch(ctx)
			if err != nil {
				r.Log.Error("outbox relay", "err", err)
				break
			}
			if n < r.Batch {
				break
			}
		}
		select {
		case <-ctx.Done():
			return nil
		case <-t.C:
		}
	}
}

type outboxRow struct {
	ID      string
	Subject string
	Payload []byte
	Headers map[string]string
}

func (r *Relay) publishBatch(ctx context.Context) (int, error) {
	published := 0
	err := pgx.BeginFunc(ctx, r.Pool, func(tx pgx.Tx) error {
		rows, err := tx.Query(ctx, `
			SELECT id::text, subject, payload, headers
			FROM outbox
			WHERE published_at IS NULL
			ORDER BY created_at
			LIMIT $1
			FOR UPDATE SKIP LOCKED`, r.Batch)
		if err != nil {
			return fmt.Errorf("select outbox: %w", err)
		}
		batch, err := pgx.CollectRows(rows, pgx.RowToStructByPos[outboxRow])
		if err != nil {
			return fmt.Errorf("scan outbox: %w", err)
		}

		ids := make([]string, 0, len(batch))
		for _, row := range batch {
			msg := nats.NewMsg(row.Subject)
			msg.Data = row.Payload
			for k, v := range row.Headers {
				msg.Header.Set(k, v)
			}
			if _, err := r.JS.PublishMsg(ctx, msg, jetstream.WithMsgID(row.ID)); err != nil {
				// Stop at the first failure to preserve order; rows already
				// published below are still marked when the tx commits.
				r.Log.Warn("outbox publish failed", "id", row.ID, "subject", row.Subject, "err", err)
				break
			}
			ids = append(ids, row.ID)
		}
		if len(ids) == 0 {
			return nil
		}
		if _, err := tx.Exec(ctx,
			`UPDATE outbox SET published_at = now() WHERE id = ANY($1::uuid[])`, ids); err != nil {
			return fmt.Errorf("mark published: %w", err)
		}
		published = len(ids)
		return nil
	})
	return published, err
}
```

Why it is safe:
- If the process dies after publishing but before commit, the rows are picked up again and republished with the same `Nats-Msg-Id`; the stream drops them as duplicates as long as the stream's `Duplicates` window is longer than the time to recover. Size that window with relay downtime in mind, or accept rare duplicates and rely on idempotent consumers (you need them anyway).
- `FOR UPDATE SKIP LOCKED` lets several relay replicas run without publishing the same row concurrently.

Trade-offs to decide explicitly:
- **Ordering.** Multiple relay replicas with `SKIP LOCKED` can publish out of `created_at` order. If consumers need per-entity order, run one active relay (Postgres advisory lock `pg_try_advisory_lock` as leader election) or include an entity version in the payload and let consumers reject stale versions.
- **Transaction length.** The DB transaction stays open while publishing; keep `Batch` small (50-200) and the publish context bounded.
- **Latency.** Polling adds up to `Interval` delay. If that matters, `LISTEN/NOTIFY` on insert can wake the relay early; keep the ticker as a fallback.
- **Cleanup.** Delete rows `WHERE published_at < now() - interval '7 days'` in a periodic job, in small batches.

## Idempotent consumer

Record the message ID in the same transaction as the side effect. The unique key does the deduplication, so it is correct even with concurrent redeliveries on different replicas.

```go
const consumerName = "billing-order-created"

func (h *BillingHandler) Handle(ctx context.Context, msg jetstream.Msg) error {
	var ev OrderCreatedV1
	if err := json.Unmarshal(msg.Data(), &ev); err != nil {
		return events.Permanent(fmt.Errorf("decode order created: %w", err))
	}
	if err := ev.Validate(); err != nil {
		return events.Permanent(err)
	}

	return pgx.BeginFunc(ctx, h.pool, func(tx pgx.Tx) error {
		tag, err := tx.Exec(ctx,
			`INSERT INTO processed_messages (consumer, message_id) VALUES ($1, $2)
			 ON CONFLICT DO NOTHING`, consumerName, ev.ID)
		if err != nil {
			return fmt.Errorf("record message: %w", err)
		}
		if tag.RowsAffected() == 0 {
			return nil // already processed; commit (no-op) and ack
		}
		if _, err := tx.Exec(ctx,
			`INSERT INTO invoices (order_id, amount_cents) VALUES ($1, $2)`,
			ev.OrderID, ev.TotalCents); err != nil {
			return fmt.Errorf("insert invoice: %w", err)
		}
		return nil
	})
}
```

Notes:
- Dedupe on the event's own ID (body `id`, equal to `Nats-Msg-Id`), not the stream sequence: a republish from the outbox gets a new sequence but keeps the ID.
- Keep `processed_messages` rows longer than the stream's `MaxAge` plus the longest redelivery window; older duplicates cannot arrive.
- Side effects outside your database (HTTP calls, emails) cannot join the transaction. Pass the event ID as the downstream idempotency key, or record intent in the DB and perform the call from your own outbox.
- Natural idempotency is even better when available: `INSERT ... ON CONFLICT (order_id) DO NOTHING` on the business table, or `UPDATE ... WHERE version < $n`.
- Map database errors deliberately: connection and serialization failures are transient (retry); constraint violations that will never pass are permanent (see error-handling).
