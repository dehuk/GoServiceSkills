# HTTP contracts: OpenAPI, oapi-codegen strict server, contract tests

Concrete setup for a provider generated from OpenAPI with oapi-codegen v2. Read this when wiring generation into a service, implementing generated handlers, or writing the HTTP contract test. The generated identifiers below follow oapi-codegen's naming for the sample spec (operation `getOrder`, path parameter `orderId`); check the generated file for the exact names in your spec, since they derive from `operationId`, parameter names and content types.

## Layout

```text
api/
  orders.openapi.yaml          # copied from the spec repo at a pinned revision
  SPEC_REVISION                # the spec repo commit the copy came from
internal/adapters/http/api/
  oapi-codegen.yaml
  generate.go                  # //go:generate line
  api.gen.go                   # generated, committed
internal/adapters/http/
  server.go                    # implements api.StrictServerInterface
```

## Minimal spec excerpt

```yaml
openapi: 3.0.3
info: { title: Orders, version: 1.0.0 }
paths:
  /orders/{orderId}:
    get:
      operationId: getOrder
      parameters:
        - { name: orderId, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: The order
          content:
            application/json:
              schema: { $ref: "#/components/schemas/OrderSummary" }
        "404":
          description: Order not found
          content:
            application/problem+json:
              schema: { $ref: "#/components/schemas/Problem" }
components:
  schemas:
    OrderSummary:
      type: object
      required: [id, status, totalCents]
      properties:
        id: { type: string, description: "Opaque identifier; never parse it." }
        status: { type: string, enum: [pending, paid, cancelled] }
        totalCents: { type: integer, format: int64, minimum: 0 }
        cancellationReason: { type: string, nullable: true }
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
```

Make semantics explicit where types cannot: say in `description` that `cancellationReason` is non-null only when `status` is `cancelled`. Identifiers are strings so no consumer rounds a large integer.

## Generation

Pin the generator as a tool dependency so every developer and CI use the same version:

```bash
go get -tool github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@<pinned version>
```

```yaml
# internal/adapters/http/api/oapi-codegen.yaml
package: api
output: api.gen.go
generate:
  models: true
  std-http-server: true
  strict-server: true
  embedded-spec: true   # GetSwagger() for request validation middleware
```

```go
// internal/adapters/http/api/generate.go
package api

//go:generate go tool oapi-codegen -config oapi-codegen.yaml ../../../../api/orders.openapi.yaml
```

Run `go generate ./...` and commit `api.gen.go`. In CI, regenerate and fail on `git diff --exit-code` so nobody edits generated code by hand or forgets to regenerate after a spec sync.

## Implementing the strict server

Handlers receive decoded request objects and return typed response objects. Map to and from domain types here; the use case never sees `api.*` types.

```go
package http

import (
	"context"

	"example.com/orders/internal/adapters/http/api"
	"example.com/orders/internal/domain"
)

type OrderService interface {
	Get(ctx context.Context, id domain.OrderID) (domain.Order, error)
}

type Server struct{ orders OrderService }

var _ api.StrictServerInterface = (*Server)(nil) // compile error if the spec gains an operation we do not implement

func (s *Server) GetOrder(ctx context.Context, req api.GetOrderRequestObject) (api.GetOrderResponseObject, error) {
	o, err := s.orders.Get(ctx, domain.OrderID(req.OrderId))
	if err != nil {
		return nil, err // mapped centrally by ResponseErrorHandlerFunc
	}
	return api.GetOrder200JSONResponse(toAPIOrder(o)), nil
}

func toAPIOrder(o domain.Order) api.OrderSummary {
	return api.OrderSummary{
		Id:         string(o.ID),
		Status:     api.OrderSummaryStatus(o.Status),
		TotalCents: o.TotalCents,
	}
}
```

Returning domain errors and letting one handler map them keeps the status mapping in one place (see the error-handling skill). The alternative, returning the generated `GetOrder404...Response` types, makes each documented error a compile-checked branch but duplicates the mapping in every handler. Pick one style per service. Either way, every error status the mapper can produce must be declared in the spec, which the contract test enforces.

## Wiring

```go
strict := api.NewStrictHandlerWithOptions(srv, nil, api.StrictHTTPServerOptions{
	// Body could not be decoded: a client error, never a 500.
	RequestErrorHandlerFunc: func(w http.ResponseWriter, r *http.Request, err error) {
		httpx.WriteProblem(w, http.StatusBadRequest, "Malformed request body")
	},
	ResponseErrorHandlerFunc: func(w http.ResponseWriter, r *http.Request, err error) {
		httpx.WriteError(w, r, log, err)
	},
})

handler := api.HandlerWithOptions(strict, api.StdHTTPServerOptions{
	BaseRouter: mux,
	// Path or query parameter could not be bound to its declared type.
	ErrorHandlerFunc: func(w http.ResponseWriter, r *http.Request, err error) {
		httpx.WriteProblem(w, http.StatusBadRequest, "Invalid request parameter")
	},
})
```

`httpx.WriteError` is the error mapper from the error-handling skill; `WriteProblem` is a small helper that writes a fixed problem body. Do not echo `err.Error()` from the binding/decoding errors to clients unless the project's error spec asks for parameter-level detail; they can contain internal type names.

For constraints the Go types do not enforce (`minimum`, `pattern`, `enum` on input), add request validation from `github.com/oapi-codegen/nethttp-middleware`, built on kin-openapi:

```go
swagger, err := api.GetSwagger()
if err != nil {
	return fmt.Errorf("load embedded spec: %w", err)
}
swagger.Servers = nil // match on path only, not on the server URL in the spec
validated := nethttpmiddleware.OapiRequestValidator(swagger)(handler)
```

Configure its error handler option so validation failures use the same problem format as everything else.

## Provider contract test

Run the real handler stack with a fake service and validate every response against the document. This catches undeclared statuses, missing required fields, `omitempty` on required fields, wrong enum strings, and `null` where the spec does not allow it.

```go
import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/getkin/kin-openapi/openapi3"
	"github.com/getkin/kin-openapi/openapi3filter"
	"github.com/getkin/kin-openapi/routers/gorillamux"
)

func TestResponsesMatchContract(t *testing.T) {
	ctx := context.Background()
	loader := openapi3.NewLoader()
	doc, err := loader.LoadFromFile("../../../api/orders.openapi.yaml")
	if err != nil {
		t.Fatalf("load spec: %v", err)
	}
	if err := doc.Validate(ctx); err != nil {
		t.Fatalf("invalid spec: %v", err)
	}
	doc.Servers = nil
	router, err := gorillamux.NewRouter(doc)
	if err != nil {
		t.Fatal(err)
	}

	h := newTestHandler(t) // generated handler + Server with a fake OrderService

	tests := []struct {
		name, method, path string
		wantStatus         int
	}{
		{"found", http.MethodGet, "/orders/o-paid", http.StatusOK},
		{"cancelled with reason", http.MethodGet, "/orders/o-cancelled", http.StatusOK},
		{"not found", http.MethodGet, "/orders/missing", http.StatusNotFound},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			req := httptest.NewRequest(tt.method, tt.path, nil)
			rec := httptest.NewRecorder()
			h.ServeHTTP(rec, req)
			if rec.Code != tt.wantStatus {
				t.Fatalf("status = %d, want %d; body: %s", rec.Code, tt.wantStatus, rec.Body)
			}

			route, pathParams, err := router.FindRoute(req)
			if err != nil {
				t.Fatalf("route not in spec: %v", err)
			}
			in := &openapi3filter.ResponseValidationInput{
				RequestValidationInput: &openapi3filter.RequestValidationInput{
					Request: req, PathParams: pathParams, Route: route,
				},
				Status:  rec.Code,
				Header:  rec.Header(),
				Options: &openapi3filter.Options{IncludeResponseStatus: true}, // undeclared status = failure
			}
			in.SetBodyBytes(rec.Body.Bytes())
			if err := openapi3filter.ValidateResponse(ctx, in); err != nil {
				t.Fatalf("response violates contract: %v", err)
			}
		})
	}
}
```

Add a case for every documented status and every materially different success shape (nullable fields set and unset, empty lists). Errors produced by the central mapper (503, 500) belong in the table too, which forces the spec to declare them.

## Breaking-change check

In the spec repo, diff each changed document against the last released revision:

```bash
oasdiff breaking <previous>/http/orders.openapi.yaml http/orders.openapi.yaml --fail-on ERR
```

A failure means the change needs a new API version or a deprecation path, not a quiet merge.
