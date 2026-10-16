# Testing NATS and JetStream Code

Three layers, cheapest first:

1. **Unit:** handlers and publishers behind small interfaces, tested with fakes. No NATS at all.
2. **Integration (in-process):** embedded `nats-server` with JetStream and a temp store. Starts in milliseconds, needs no Docker, safe for `go test -race` and `t.Parallel()` because every test gets its own server and port.
3. **Integration (container):** testcontainers-go, when you need the exact production server version or cluster behaviour.

## Unit: fake jetstream.Msg

`jetstream.Msg` is an interface, so the ack mapping can be tested by embedding it and overriding only what the worker calls. Calling an un-overridden method panics on the nil embedded interface, which flags unexpected calls.

```go
type fakeMsg struct {
	jetstream.Msg // nil; unexpected method calls panic
	data      []byte
	headers   nats.Header
	delivered uint64

	acked, termed bool
	nakDelay      time.Duration
}

func (m *fakeMsg) Data() []byte         { return m.data }
func (m *fakeMsg) Headers() nats.Header { return m.headers }
func (m *fakeMsg) Subject() string      { return "orders.order.created.v1" }
func (m *fakeMsg) Ack() error           { m.acked = true; return nil }
func (m *fakeMsg) TermWithReason(string) error {
	m.termed = true
	return nil
}
func (m *fakeMsg) NakWithDelay(d time.Duration) error { m.nakDelay = d; return nil }
func (m *fakeMsg) Metadata() (*jetstream.MsgMetadata, error) {
	return &jetstream.MsgMetadata{
		NumDelivered: m.delivered,
		Stream:       "ORDERS",
		Sequence:     jetstream.SequencePair{Stream: 42, Consumer: 7},
	}, nil
}
```

Table-test the worker's `process` with handlers returning `nil`, a transient error, `Permanent(err)`, and a transient error on the last attempt. For the dead-letter path, the worker also needs a `jetstream.JetStream`; use the embedded server below rather than faking that large interface.

## Integration: embedded server

```go
package natstest

import (
	"testing"
	"time"

	"github.com/nats-io/nats-server/v2/server"
	"github.com/nats-io/nats.go"
	"github.com/nats-io/nats.go/jetstream"
)

// Start runs an in-process NATS server with JetStream and returns a
// connected client. Everything is cleaned up when the test ends.
func Start(t *testing.T) (*nats.Conn, jetstream.JetStream) {
	t.Helper()

	ns, err := server.NewServer(&server.Options{
		Host:      "127.0.0.1",
		Port:      server.RANDOM_PORT,
		JetStream: true,
		StoreDir:  t.TempDir(),
		NoLog:     true,
		NoSigs:    true,
	})
	if err != nil {
		t.Fatalf("new nats server: %v", err)
	}
	ns.Start()
	if !ns.ReadyForConnections(10 * time.Second) {
		ns.Shutdown()
		t.Fatal("nats server not ready")
	}
	t.Cleanup(func() {
		ns.Shutdown()
		ns.WaitForShutdown()
	})

	nc, err := nats.Connect(ns.ClientURL())
	if err != nil {
		t.Fatalf("connect: %v", err)
	}
	t.Cleanup(nc.Close) // registered after the server cleanup, so it runs first

	js, err := jetstream.New(nc)
	if err != nil {
		t.Fatalf("jetstream: %v", err)
	}
	return nc, js
}
```

- `server.NewServer` and `ns.Start()` come from `github.com/nats-io/nats-server/v2/server`. `Start` launches the server's goroutines and returns; always gate on `ReadyForConnections`.
- The server module is large; keep this helper in an `internal/natstest` package imported only from `_test.go` files so it never lands in the production binary.
- The `nats-server/v2/test` package also has helpers (`RunServer`, `DefaultTestOptions`) used by nats.go's own tests; the explicit helper above is clearer about JetStream and storage.

### Round-trip test

Exercise the real bootstrap (stream + consumer creation), publish, and assert the side effect. Poll with a deadline instead of sleeping a fixed time.

```go
func TestOrderCreatedConsumer(t *testing.T) {
	t.Parallel()
	_, js := natstest.Start(t)
	ctx := t.Context() // Go 1.24+

	if err := events.Bootstrap(ctx, js); err != nil { // same function the migrate command uses
		t.Fatalf("bootstrap: %v", err)
	}
	cons, err := js.Consumer(ctx, "ORDERS", "billing-order-created")
	if err != nil {
		t.Fatalf("consumer: %v", err)
	}

	got := make(chan string, 1)
	cc, err := cons.Consume(func(msg jetstream.Msg) {
		got <- string(msg.Data())
		if err := msg.Ack(); err != nil {
			t.Errorf("ack: %v", err)
		}
	})
	if err != nil {
		t.Fatalf("consume: %v", err)
	}
	t.Cleanup(cc.Stop)

	ack, err := js.Publish(ctx, "orders.order.created.v1", []byte(`{"id":"e1"}`),
		jetstream.WithMsgID("e1"))
	if err != nil {
		t.Fatalf("publish: %v", err)
	}
	if ack.Stream != "ORDERS" {
		t.Fatalf("stored in %q, want ORDERS", ack.Stream)
	}

	// Duplicate publish within the window is dropped by the server.
	dup, err := js.Publish(ctx, "orders.order.created.v1", []byte(`{"id":"e1"}`),
		jetstream.WithMsgID("e1"))
	if err != nil {
		t.Fatalf("publish dup: %v", err)
	}
	if !dup.Duplicate {
		t.Fatal("expected duplicate publish to be detected")
	}

	select {
	case body := <-got:
		if body != `{"id":"e1"}` {
			t.Fatalf("got %s", body)
		}
	case <-time.After(5 * time.Second):
		t.Fatal("message not consumed")
	}
}
```

Useful additional cases:
- Handler returns a transient error: assert the message is redelivered (`NumDelivered == 2`). Use a short `NakWithDelay` or make the delay injectable so the test stays fast.
- Handler returns `Permanent`: assert a message appears on `dlq.<consumer>` (read it with an ordered consumer or `stream.GetLastMsgForSubject`).
- Consumer lag: publish N, assert `cons.Info(ctx)` reports `NumPending` decreasing to 0.

## Integration: testcontainers-go

```go
import (
	"github.com/testcontainers/testcontainers-go"
	tcnats "github.com/testcontainers/testcontainers-go/modules/nats"
)

func startNATSContainer(t *testing.T) string {
	t.Helper()
	ctx := t.Context()
	c, err := tcnats.Run(ctx, "nats:2.14-alpine")
	if err != nil {
		t.Fatalf("start nats container: %v", err)
	}
	t.Cleanup(func() {
		if err := testcontainers.TerminateContainer(c); err != nil {
			t.Logf("terminate nats container: %v", err)
		}
	})
	url, err := c.ConnectionString(ctx)
	if err != nil {
		t.Fatalf("connection string: %v", err)
	}
	return url
}
```

The NATS module starts the server with JetStream enabled by default (its default command is `-DV -js`); pass custom args through the module's options if you need others. Use a build tag (`//go:build integration`) or `testing.Short()` so container tests do not run in the fast unit loop.

## Local stack: docker compose

```yaml
services:
  nats:
    image: nats:2.14-alpine
    command: ["-js", "-sd", "/data", "-m", "8222"]
    ports:
      - "4222:4222"   # clients
      - "8222:8222"   # monitoring: /varz, /jsz, /connz
    volumes:
      - nats-data:/data
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8222/healthz?js-enabled-only=true"]
      interval: 5s
      timeout: 3s
      retries: 10
volumes:
  nats-data:
```

Services that depend on NATS use `depends_on: { nats: { condition: service_healthy } }`, and still set `nats.RetryOnFailedConnect(true)`, because healthy-at-start does not mean available forever.
