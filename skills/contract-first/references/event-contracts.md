# Event contracts: schema, Go type, contract test

A complete example for one NATS event: the JSON Schema that lives in the spec repo, the Go type the producer and consumers use, and a provider-side contract test. The Go code compiles and the test passes against the schema shown (Go 1.24+, `github.com/santhosh-tekuri/jsonschema/v6`). Read this when defining a new event or adding tests for an existing one.

## Schema (spec repo: `events/orders.created.v1.schema.json`)

Note what is deliberately absent: no `additionalProperties: false`, so adding an optional field later stays compatible with old validators. `note` is optional and nullable; `totalCents` states its unit in the name. `type` is a constant, so a v2 payload can never validate as v1.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://contracts.example.com/events/orders.created.v1.schema.json",
  "title": "orders.created v1",
  "type": "object",
  "required": ["id", "type", "occurredAt", "data"],
  "properties": {
    "id": { "type": "string", "description": "Unique event ID; also used as Nats-Msg-Id for deduplication." },
    "type": { "const": "orders.created.v1" },
    "occurredAt": { "type": "string", "format": "date-time" },
    "data": {
      "type": "object",
      "required": ["orderId", "customerId", "totalCents", "currency"],
      "properties": {
        "orderId": { "type": "string" },
        "customerId": { "type": "string" },
        "totalCents": { "type": "integer", "minimum": 1 },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" },
        "note": { "type": ["string", "null"] }
      }
    }
  }
}
```

Keep a matching `orders.created.v1.example.json` next to it; the spec repo's CI validates every example against its schema, and consumers use the examples as test fixtures.

## Go type (producer and consumers)

A generic envelope plus one struct per event version. Old versions stay in the code as long as messages of that version can still be in the stream.

```go
package events

import "time"

const TypeOrderCreatedV1 = "orders.created.v1"

type Envelope[T any] struct {
	ID         string    `json:"id"`
	Type       string    `json:"type"`
	OccurredAt time.Time `json:"occurredAt"`
	Data       T         `json:"data"`
}

type OrderCreatedV1 struct {
	OrderID    string  `json:"orderId"`
	CustomerID string  `json:"customerId"`
	TotalCents int64   `json:"totalCents"`
	Currency   string  `json:"currency"`
	Note       *string `json:"note"`
}
```

Field tags are the contract's names; pointer fields mark nullable values. Do not add `omitempty` to required fields: a zero value would silently disappear from the payload and fail consumers. The contract test below catches that.

## Provider contract test

Sync the schema into `testdata/` at the pinned spec revision (do not edit it there). Build the event the same way production code does, marshal it, and validate.

```go
package events

import (
	"bytes"
	"encoding/json"
	"testing"
	"time"

	"github.com/santhosh-tekuri/jsonschema/v6"
)

// The schema is the contract from the spec repo, synced into testdata at a
// pinned revision. The test proves the Go type serializes to something the
// contract accepts, including the optional and null paths.
func TestOrderCreatedV1MatchesContract(t *testing.T) {
	sch, err := jsonschema.NewCompiler().Compile("testdata/orders.created.v1.schema.json")
	if err != nil {
		t.Fatalf("compile schema: %v", err)
	}

	note := "leave at the door"
	tests := []struct {
		name string
		data OrderCreatedV1
	}{
		{name: "minimal", data: OrderCreatedV1{OrderID: "o1", CustomerID: "c1", TotalCents: 100, Currency: "EUR"}},
		{name: "with note", data: OrderCreatedV1{OrderID: "o1", CustomerID: "c1", TotalCents: 100, Currency: "EUR", Note: &note}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ev := Envelope[OrderCreatedV1]{ID: "e1", Type: TypeOrderCreatedV1, OccurredAt: time.Now().UTC(), Data: tt.data}
			b, err := json.Marshal(ev)
			if err != nil {
				t.Fatal(err)
			}
			inst, err := jsonschema.UnmarshalJSON(bytes.NewReader(b))
			if err != nil {
				t.Fatal(err)
			}
			if err := sch.Validate(inst); err != nil {
				t.Fatalf("event does not match contract:\n%s\n%v", b, err)
			}
		})
	}
}
```

Extend the table with every materially different shape the producer can emit (optional fields set and unset, each enum value). If production builds events through a constructor such as `NewOrderCreated(order)`, call that instead of constructing the struct literally, so the test covers the real mapping.

## Consumer test against examples

Consumers do the opposite: decode the spec repo's example for every version still in retention with the production decoder, and run the handler on it.

```go
func TestHandleOrderCreatedExamples(t *testing.T) {
	files, err := filepath.Glob("testdata/orders.created.v*.example.json")
	if err != nil || len(files) == 0 {
		t.Fatalf("no examples found (err=%v)", err)
	}
	for _, f := range files {
		t.Run(filepath.Base(f), func(t *testing.T) {
			b, err := os.ReadFile(f)
			if err != nil {
				t.Fatal(err)
			}
			h := newTestHandler(t) // the real handler with fake ports
			if err := h.Process(context.Background(), b); err != nil {
				t.Fatalf("process %s: %v", f, err)
			}
		})
	}
}
```

The consumer's decoder must not use `DisallowUnknownFields`; examples from newer minor revisions may carry fields this consumer does not know yet.

## Compatibility check in the spec repo

For each event type, validate:

- every older example against the newest schema of the same major version (new consumers read old messages), and
- the newest example against every older schema of the same major version (old consumers read new messages).

Both directions passing means the change is additive. A failure means either the change is breaking (new major version) or the schema was too strict to begin with.

## Protobuf variant

If you choose protobuf instead: one `.proto` per event in the spec repo, `buf lint` and `buf breaking --against '<spec repo>#branch=main'` in the spec repo's CI, generate with `buf generate` in each service. Never reuse or renumber a field; mark removed fields `reserved` (both number and name). Unknown fields are preserved by default in Go protobuf, which gives you the "ignore unknown fields" consumer rule for free.
