# Provisioning streams and consumers

Streams and consumers are created by a dedicated `migrate-nats` subcommand of the service binary, run before rollout in every environment, exactly like database migrations. This file has the code, how to run it locally and in CI/CD, and how to change topology safely.

## Contents

- Why a separate step
- Topology in code
- The `migrate-nats` subcommand
- Binding at runtime (fail fast)
- Credentials: migrator vs runtime
- Running it: local, compose, tests, handoff to deployment
- Changing topology safely
- Removing streams and consumers

## Why a separate step

- **Least privilege.** Only the migrator holds stream/consumer admin rights, and only for the minute it runs. Running service instances can publish and consume but cannot delete or reconfigure a stream, so a leaked runtime credential cannot wipe data.
- **No config flapping.** If services create streams on startup, an old replica restarting during a rolling deploy re-applies the old config.
- **Failures stop the deploy, not the service.** An invalid change (for example an immutable field) fails the step before the new version replaces the old one.
- **One code path everywhere.** Local dev, integration tests and production all call the same `Bootstrap` function, so what you test is what you deploy.

## Topology in code

Keep the definitions next to the code that publishes and consumes, in one place per service. Environment differences (replicas, limits) come from config, not from separate definitions.

```go
// internal/adapters/natsx/topology.go
package natsx

import (
	"context"
	"fmt"
	"time"

	"github.com/nats-io/nats.go/jetstream"
)

// Topology holds the per-environment knobs; everything else is fixed in code.
type Topology struct {
	Replicas int           // 1 on a single local server, 3 in a clustered production setup
	MaxAge   time.Duration // retention for the owned stream
}

type consumerDef struct {
	stream string
	cfg    jetstream.ConsumerConfig
}

// Streams owned by this service (it publishes to them).
func (t Topology) streams() []jetstream.StreamConfig {
	return []jetstream.StreamConfig{{
		Name:       "ORDERS",
		Subjects:   []string{"orders.>"},
		Retention:  jetstream.LimitsPolicy,
		Storage:    jetstream.FileStorage,
		Replicas:   t.Replicas,
		MaxAge:     t.MaxAge,
		MaxBytes:   10 << 30,
		Duplicates: 2 * time.Minute,
	}}
}

// Durable consumers owned by this service (it consumes through them),
// possibly on streams owned by other services.
func (t Topology) consumers() []consumerDef {
	return []consumerDef{{
		stream: "PAYMENTS",
		cfg: jetstream.ConsumerConfig{
			Durable:       "orders-payment-captured",
			FilterSubject: "payments.payment.captured.v1",
			AckPolicy:     jetstream.AckExplicitPolicy,
			AckWait:       30 * time.Second,
			MaxDeliver:    10,
			BackOff:       []time.Duration{time.Second, 5 * time.Second, 30 * time.Second},
			Replicas:      t.Replicas,
		},
	}}
}

// Bootstrap creates or updates every stream and consumer this service owns.
// It is idempotent: running it twice, or from two deploys at once, is safe.
// It never deletes anything; see "Removing streams and consumers".
func Bootstrap(ctx context.Context, js jetstream.JetStream, t Topology) error {
	for _, cfg := range t.streams() {
		if _, err := js.CreateOrUpdateStream(ctx, cfg); err != nil {
			return fmt.Errorf("stream %s: %w", cfg.Name, err)
		}
	}
	for _, c := range t.consumers() {
		if _, err := js.CreateOrUpdateConsumer(ctx, c.stream, c.cfg); err != nil {
			return fmt.Errorf("consumer %s on %s: %w", c.cfg.Durable, c.stream, err)
		}
	}
	return nil
}
```

The owner rule keeps teams from overwriting each other: the publishing service declares its stream; each consuming service declares only its own durables, even on someone else's stream. Never declare another service's stream "just in case": two definitions of one stream fight on every deploy.

## The `migrate-nats` subcommand

Ship it in the same binary and image as the service, so the topology always matches the code version being deployed.

```go
// cmd/orders/main.go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	cfg, err := config.Load()
	if err != nil {
		slog.Error("load config", "err", err)
		os.Exit(1)
	}

	cmd := "serve"
	if len(os.Args) > 1 {
		cmd = os.Args[1]
	}
	switch cmd {
	case "serve":
		err = serve(ctx, cfg)
	case "migrate-nats":
		err = migrateNATS(ctx, cfg)
	default:
		err = fmt.Errorf("unknown command %q (want serve or migrate-nats)", cmd)
	}
	if err != nil {
		slog.Error(cmd+" failed", "err", err)
		os.Exit(1) // non-zero exit stops whatever runs it (compose dependency, deploy step)
	}
}

func migrateNATS(ctx context.Context, cfg config.Config) error {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Minute)
	defer cancel()

	opts := []nats.Option{nats.Name(cfg.ServiceName + "-migrate-nats")}
	if cfg.NATSMigratorCredsFile != "" { // empty locally, where the dev server has no auth
		opts = append(opts, nats.UserCredentials(cfg.NATSMigratorCredsFile))
	}
	nc, err := nats.Connect(cfg.NATSURL, opts...)
	if err != nil {
		return fmt.Errorf("connect nats: %w", err)
	}
	defer nc.Close()

	js, err := jetstream.New(nc)
	if err != nil {
		return fmt.Errorf("jetstream: %w", err)
	}
	if err := natsx.Bootstrap(ctx, js, natsx.Topology{
		Replicas: cfg.NATSReplicas,
		MaxAge:   cfg.NATSStreamMaxAge,
	}); err != nil {
		return err
	}
	slog.Info("nats topology applied")
	return nil
}
```

## Binding at runtime (fail fast)

The running service only looks streams and consumers up. If the migrate step did not run, it fails at startup with a message that says so, instead of silently creating something with default settings.

```go
cons, err := js.Consumer(ctx, "PAYMENTS", "orders-payment-captured")
if errors.Is(err, jetstream.ErrConsumerNotFound) || errors.Is(err, jetstream.ErrStreamNotFound) {
	return fmt.Errorf("nats consumer missing, run `migrate-nats` before deploying: %w", err)
}
if err != nil {
	return fmt.Errorf("bind consumer: %w", err)
}
```

Publishers need nothing extra: `js.Publish` to a subject that no stream captures returns an error (no responders), which surfaces the same problem on the first publish. Check it in the readiness probe if you want it earlier: `js.Stream(ctx, "ORDERS")`.

## Credentials: migrator vs runtime

Issue two NATS users per service:

| User | Used by | Can do |
|---|---|---|
| `<svc>-migrator` | `migrate-nats` only | Create/update/info on its own stream (`$JS.API.STREAM.CREATE.ORDERS`, `$JS.API.STREAM.UPDATE.ORDERS`, `$JS.API.STREAM.INFO.ORDERS`) and create/info on its own durables (`$JS.API.CONSUMER.CREATE.PAYMENTS.orders-payment-captured.>`, `$JS.API.CONSUMER.INFO.PAYMENTS.orders-payment-captured`) |
| `<svc>` | the running service | Publish to its subjects, `INFO`/`MSG.NEXT`/`ACK` on its own consumers; no `CREATE`, `UPDATE` or `DELETE` |

Provide the migrator credentials only to the `migrate-nats` run, never to the running service. Exact runtime permission lists and the server config syntax are in the go-security skill (infrastructure reference). If a permission is missing, the server logs a permissions violation naming the subject; tune from that.

## Running it

### Local, service on the host

```bash
docker compose up -d --wait nats
set -a; . ./.env.local; set +a
go run ./cmd/orders migrate-nats
go run ./cmd/orders            # serve
```

Add a `make nats-migrate` (or Taskfile) target so nobody has to remember the order.

### Local, everything in compose

A one-shot service built from the same image, which the app services wait for. See the docker-patterns skill for the full compose file.

```yaml
  nats-migrate:
    build:
      context: .
      args: { SERVICE: orders, VERSION: "${VERSION:-dev}" }
    command: ["migrate-nats"]
    env_file: compose.env
    profiles: ["app"]
    depends_on:
      nats:
        condition: service_healthy
    restart: "no"

  orders:
    # ...
    depends_on:
      nats-migrate:
        condition: service_completed_successfully
```

### Integration tests

Call `natsx.Bootstrap` directly against the embedded or containerized server (see `testing.md`), with `Replicas: 1`.

### Handoff to deployment

How and where the deploy pipeline runs `migrate-nats` is owned by whoever operates deployments, not by the service code. The service's job is to make the step easy to run correctly and to document its contract, for example in the service README:

| | Contract |
|---|---|
| Command | `<binary> migrate-nats`, from the same image/version that is about to be deployed |
| When | Before the new version starts serving; after the producer's stream exists (see below) |
| Config | `NATS_URL`, `NATS_REPLICAS`, `NATS_STREAM_MAX_AGE`, `NATS_MIGRATOR_CREDS_FILE` |
| Credentials | The migrator user, not the runtime one |
| Result | Exit 0 = topology applied; non-zero = do not roll out (the log says which stream/consumer failed and why) |
| Safety | Idempotent and safe to retry or run concurrently; never deletes; finishes within 2 minutes or fails |

Do not write deployment manifests, pipeline steps or hooks as part of a service change unless the user explicitly asks; describe the contract instead.

If database migrations also run as a separate step, they are independent of `migrate-nats`; both must succeed before the new version starts.

### Deploy order between services

A consumer service's `migrate-nats` fails with "stream not found" if the producer's stream does not exist yet. That is the correct signal: deploy the producer (or at least its migrate step) first. For a brand-new event flow, ship the producer's stream before the consumer that depends on it.

## Changing topology safely

`CreateOrUpdate*` applies most edits in place, but some fields cannot change after creation, and the server rejects the update. That failure stops the deploy, which is what you want.

| Change | In place? | How |
|---|---|---|
| `MaxAge`, `MaxBytes`, `MaxMsgs`, `Duplicates`, `Replicas` (within cluster size), adding subjects | Yes | Edit the definition, deploy |
| Removing a subject from a stream | Yes, but messages on it are no longer captured | Only after no publisher uses it (expand-contract) |
| `Storage` (file/memory), stream name | No | New stream + migrate data + switch publishers and consumers, then retire the old one |
| `Retention` policy | Restricted (depends on server version) | Treat as "new stream" |
| Consumer `AckWait`, `MaxDeliver`, `BackOff`, `MaxAckPending`, filter subjects (server 2.10+) | Yes | Edit and deploy |
| Consumer `DeliverPolicy`, `AckPolicy`, durable name | No | New durable with a new name; drain the old one; retire it |

Renaming a subject is expand-contract: add the new subject to the stream, make publishers send to the new one, move consumers' filters, then remove the old subject once nothing is pending on it.

Keep `Replicas` from config: setting 3 against a single local server fails with "insufficient resources".

## Removing streams and consumers

`Bootstrap` never deletes. Deleting a definition from code leaves it on the server, and a forgotten durable is not harmless: on an `InterestPolicy` or `WorkQueuePolicy` stream it keeps messages from being removed, and it shows up as growing lag.

Retire explicitly, as its own reviewed change:

- Keep a short `retired` list in the topology and delete those in `Bootstrap` (`js.DeleteConsumer(ctx, stream, name)`, ignoring `ErrConsumerNotFound`), then drop the entry a release later; or
- Delete once by hand with the nats CLI (`nats consumer rm PAYMENTS old-durable`) as part of the release checklist.

Deleting a stream deletes its messages. Do it by hand, after a backup (`nats stream backup`) if the data matters, never automatically from a deploy.
