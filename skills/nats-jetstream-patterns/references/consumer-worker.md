# Consumer Worker, Error Mapping, and Dead Letters

A complete JetStream consumer worker for a Go service: error classification, ack mapping, panic safety, DLQ publishing, graceful shutdown, a concurrent variant, and a MaxDeliver advisory listener.

## Error kinds

Handlers return plain errors. Only "permanent" (poison) errors are marked; everything else is treated as transient and retried. That default is safe: a misclassified transient error costs a few retries, a misclassified permanent error loses a message.

```go
package events

import "errors"

type permanentError struct{ err error }

func (e permanentError) Error() string { return e.err.Error() }
func (e permanentError) Unwrap() error { return e.err }

// Permanent marks err as non-retryable: decode failures, validation errors,
// references that will never resolve.
func Permanent(err error) error {
	if err == nil {
		return nil
	}
	return permanentError{err: err}
}

func IsPermanent(err error) bool {
	var p permanentError
	return errors.As(err, &p)
}
```

## Worker

```go
package events

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"math/rand/v2"
	"slices"
	"strconv"
	"time"

	"github.com/nats-io/nats.go"
	"github.com/nats-io/nats.go/jetstream"
)

type Handler func(ctx context.Context, msg jetstream.Msg) error

type Worker struct {
	JS         jetstream.JetStream
	Consumer   jetstream.Consumer
	Name       string        // durable name, used for DLQ subject and metrics
	MaxDeliver int           // same value as ConsumerConfig.MaxDeliver
	Timeout    time.Duration // per-message budget, below AckWait
	Handle     Handler
	Log        *slog.Logger
}

// Run consumes until ctx is cancelled, then drains in-flight messages.
func (w *Worker) Run(ctx context.Context) error {
	// Handlers must not see the shutdown cancellation, otherwise every
	// in-flight message fails during drain and gets redelivered.
	base := context.WithoutCancel(ctx)

	cc, err := w.Consumer.Consume(
		func(msg jetstream.Msg) { w.process(base, msg) },
		jetstream.PullMaxMessages(64),
		jetstream.ConsumeErrHandler(func(_ jetstream.ConsumeContext, err error) {
			w.Log.Warn("consume error", "consumer", w.Name, "err", err)
		}),
	)
	if err != nil {
		return fmt.Errorf("consume %s: %w", w.Name, err)
	}

	<-ctx.Done()
	cc.Drain() // stop pulling, let buffered/in-flight messages finish
	select {
	case <-cc.Closed(): // ConsumeContext.Closed requires nats.go v1.37+
	case <-time.After(w.Timeout + 5*time.Second):
		cc.Stop()
		w.Log.Warn("consumer drain timed out", "consumer", w.Name)
	}
	return nil
}

func (w *Worker) process(base context.Context, msg jetstream.Msg) {
	meta, err := msg.Metadata()
	if err != nil {
		// Not a JetStream message (no ack reply subject); nothing to ack.
		w.Log.Error("message metadata", "subject", msg.Subject(), "err", err)
		return
	}

	ctx, cancel := context.WithTimeout(base, w.Timeout)
	defer cancel()

	err = w.safeHandle(ctx, msg)
	log := w.Log.With("consumer", w.Name, "subject", msg.Subject(),
		"stream_seq", meta.Sequence.Stream, "attempt", meta.NumDelivered)

	var ackErr error
	switch {
	case err == nil:
		ackErr = msg.Ack()

	case IsPermanent(err) || meta.NumDelivered >= uint64(w.MaxDeliver):
		// Poison message, or the last allowed attempt failed: capture it with
		// the error, then stop redelivery. If the DLQ publish itself fails,
		// retry instead of terminating so the message is not lost.
		log.Error("dead-lettering message", "err", err)
		if dlqErr := w.deadLetter(ctx, msg, meta, err); dlqErr != nil {
			log.Error("dead-letter publish failed", "err", dlqErr)
			ackErr = msg.NakWithDelay(30 * time.Second)
			break
		}
		ackErr = msg.TermWithReason(truncate(err.Error(), 256))

	default:
		log.Warn("transient failure, retrying", "err", err)
		ackErr = msg.NakWithDelay(retryDelay(meta.NumDelivered))
	}
	if ackErr != nil {
		// The server will redeliver after AckWait; the handler must be idempotent.
		log.Error("ack failed", "err", ackErr)
	}
}

// safeHandle turns a panic into a transient error so one bad message cannot
// crash the process (and every other consumer in it).
func (w *Worker) safeHandle(ctx context.Context, msg jetstream.Msg) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("handler panic: %v", r)
		}
	}()
	return w.Handle(ctx, msg)
}

func (w *Worker) deadLetter(ctx context.Context, msg jetstream.Msg, meta *jetstream.MsgMetadata, cause error) error {
	dl := nats.NewMsg("dlq." + w.Name)
	dl.Data = msg.Data()
	for k, v := range msg.Headers() {
		dl.Header[k] = slices.Clone(v)
	}
	dl.Header.Del(jetstream.MsgIDHeader) // the original ID belongs to the source stream
	dl.Header.Set("X-Dlq-Original-Subject", msg.Subject())
	dl.Header.Set("X-Dlq-Stream", meta.Stream)
	dl.Header.Set("X-Dlq-Stream-Seq", strconv.FormatUint(meta.Sequence.Stream, 10))
	dl.Header.Set("X-Dlq-Deliveries", strconv.FormatUint(meta.NumDelivered, 10))
	if cause != nil {
		dl.Header.Set("X-Dlq-Error", truncate(cause.Error(), 1024))
	}
	// Same ID as the advisory listener uses, so both paths never double-DLQ.
	id := fmt.Sprintf("%s-%s-%d", meta.Stream, w.Name, meta.Sequence.Stream)
	_, err := w.JS.PublishMsg(ctx, dl, jetstream.WithMsgID(id))
	return err
}

// retryDelay: exponential from 1s, capped at 5m, with jitter to avoid
// synchronized retry waves after a dependency outage.
func retryDelay(attempt uint64) time.Duration {
	d := min(time.Second<<min(max(attempt, 1)-1, 9), 5*time.Minute)
	return d/2 + rand.N(d/2+1)
}

func truncate(s string, n int) string {
	if len(s) <= n {
		return s
	}
	return s[:n]
}
```

Notes:
- `jetstream.MsgIDHeader` is the `Nats-Msg-Id` header name constant.
- `TermWithReason` requires nats.go v1.33+; on older ones use `Term()`.
- `MaxDeliver` is duplicated in the worker so the handler can capture the last attempt with its error. Keep the two values from one config constant.

### DLQ stream

Create it in `Bootstrap` (run by `migrate-nats`, see `provisioning.md`), not when the worker starts.

```go
_, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
	Name:       "DLQ",
	Subjects:   []string{"dlq.>"},
	Storage:    jetstream.FileStorage,
	MaxAge:     30 * 24 * time.Hour,
	Duplicates: 10 * time.Minute,
})
```

Replay from the DLQ is a deliberate operator action: read `dlq.<consumer>`, republish to `X-Dlq-Original-Subject` once the bug is fixed. Do not auto-replay; a poison message will just loop.

## Concurrent variant with Messages()

One `Consume` handles messages sequentially. When the handler is I/O bound, fan out with the iterator and a bounded pool. `MaxAckPending` still caps total in-flight across replicas.

```go
func (w *Worker) RunConcurrent(ctx context.Context, workers int) error {
	it, err := w.Consumer.Messages(jetstream.PullMaxMessages(2 * workers))
	if err != nil {
		return fmt.Errorf("messages %s: %w", w.Name, err)
	}
	stop := context.AfterFunc(ctx, it.Drain) // Next returns ErrMsgIteratorClosed once drained
	defer stop()

	base := context.WithoutCancel(ctx)
	sem := make(chan struct{}, workers)
	var wg sync.WaitGroup
	defer wg.Wait()

	for {
		msg, err := it.Next()
		if errors.Is(err, jetstream.ErrMsgIteratorClosed) {
			return nil
		}
		if err != nil {
			w.Log.Warn("next message", "consumer", w.Name, "err", err)
			continue
		}
		sem <- struct{}{}
		wg.Add(1)
		go func() {
			defer wg.Done()
			defer func() { <-sem }()
			w.process(base, msg)
		}()
	}
}
```

(Add `"sync"` to the worker's imports.) Per-subject ordering is lost with concurrency. If order per entity matters, either keep one sequential `Consume` per consumer or partition subjects (`orders.order.events.<id>`) and use separate consumers per partition.

## MaxDeliver advisory listener

Needs `"encoding/json"` in addition to the worker's imports. Catches messages that exhausted `MaxDeliver` without the handler seeing the last attempt fail, typically because the process crashed or hung past `AckWait`. Capture advisories in a stream so the listener can be down without losing them.

```go
type maxDeliveriesAdvisory struct {
	Stream     string `json:"stream"`
	Consumer   string `json:"consumer"`
	StreamSeq  uint64 `json:"stream_seq"`
	Deliveries uint64 `json:"deliveries"`
}

// SetupAdvisoryCapture belongs in Bootstrap (run by `migrate-nats`); the running
// dlq-mover binds to the consumer with js.Consumer(ctx, "ADVISORIES", "dlq-mover").
func SetupAdvisoryCapture(ctx context.Context, js jetstream.JetStream) (jetstream.Consumer, error) {
	if _, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
		Name:     "ADVISORIES",
		Subjects: []string{"$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES.>"},
		Storage:  jetstream.FileStorage,
		MaxAge:   7 * 24 * time.Hour,
	}); err != nil {
		return nil, fmt.Errorf("advisory stream: %w", err)
	}
	return js.CreateOrUpdateConsumer(ctx, "ADVISORIES", jetstream.ConsumerConfig{
		Durable:    "dlq-mover",
		AckPolicy:  jetstream.AckExplicitPolicy,
		MaxDeliver: 10,
	})
}

func MoveToDLQ(ctx context.Context, js jetstream.JetStream, msg jetstream.Msg) error {
	var adv maxDeliveriesAdvisory
	if err := json.Unmarshal(msg.Data(), &adv); err != nil {
		return Permanent(fmt.Errorf("decode advisory: %w", err))
	}
	stream, err := js.Stream(ctx, adv.Stream)
	if err != nil {
		return fmt.Errorf("stream %s: %w", adv.Stream, err)
	}
	raw, err := stream.GetMsg(ctx, adv.StreamSeq)
	if errors.Is(err, jetstream.ErrMsgNotFound) {
		return nil // already removed by retention limits; nothing to move
	}
	if err != nil {
		return fmt.Errorf("get msg %d: %w", adv.StreamSeq, err)
	}

	dl := nats.NewMsg("dlq." + adv.Consumer)
	dl.Data = raw.Data
	for k, v := range raw.Header {
		dl.Header[k] = slices.Clone(v)
	}
	dl.Header.Del(jetstream.MsgIDHeader)
	dl.Header.Set("X-Dlq-Original-Subject", raw.Subject)
	dl.Header.Set("X-Dlq-Stream", adv.Stream)
	dl.Header.Set("X-Dlq-Stream-Seq", strconv.FormatUint(adv.StreamSeq, 10))
	dl.Header.Set("X-Dlq-Deliveries", strconv.FormatUint(adv.Deliveries, 10))
	dl.Header.Set("X-Dlq-Error", "max deliveries exceeded")
	id := fmt.Sprintf("%s-%s-%d", adv.Stream, adv.Consumer, adv.StreamSeq)
	_, err = js.PublishMsg(ctx, dl, jetstream.WithMsgID(id))
	return err
}
```

Run `MoveToDLQ` as the `Handle` of a `Worker` bound to the `dlq-mover` consumer. The shared `Nats-Msg-Id` scheme (`<stream>-<consumer>-<seq>`) makes the handler path and the advisory path idempotent with each other within the DLQ stream's dedup window. Reading via `stream.GetMsg` requires the runtime credentials to allow the direct/get API on the source stream.

A similar advisory exists for terminated messages (`$JS.EVENT.ADVISORY.CONSUMER.MSG_TERMINATED.<STREAM>.<CONSUMER>`); since the worker already dead-letters before `Term`, you rarely need it.
