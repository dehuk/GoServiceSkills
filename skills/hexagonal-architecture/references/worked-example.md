# Worked example: one vertical slice

A complete orders slice wired the hexagonal way: domain, use cases, consumer-owned ports, a pgx adapter with a unit of work and transactional outbox, an HTTP adapter, the composition root, and a use-case test with fakes. The code compiles and its test passes as shown (pgx v5, Go 1.24+). Read it when you need concrete code to copy from; SKILL.md explains the decisions.

```text
cmd/orders/main.go
internal/domain/order.go
internal/app/ports.go
internal/app/orders.go
internal/app/orders_test.go
internal/adapters/postgres/orders.go
internal/adapters/http/orders.go
```

## Domain

Entities, invariants and domain errors. Standard library only.

```go
// internal/domain/order.go
package domain

import (
	"errors"
	"fmt"
	"time"
)

var (
	ErrNotFound      = errors.New("not found")
	ErrConflict      = errors.New("conflict")
	ErrInvalid       = errors.New("invalid")
	ErrInvalidStatus = fmt.Errorf("%w: illegal status transition", ErrInvalid)
)

type OrderStatus string

const (
	StatusPending   OrderStatus = "pending"
	StatusPaid      OrderStatus = "paid"
	StatusCancelled OrderStatus = "cancelled"
)

type OrderID string

type Order struct {
	ID         OrderID
	CustomerID string
	TotalCents int64
	Status     OrderStatus
	CreatedAt  time.Time
}

// NewOrder enforces invariants that must hold no matter which adapter created the order.
func NewOrder(id OrderID, customerID string, totalCents int64, now time.Time) (Order, error) {
	if customerID == "" {
		return Order{}, fmt.Errorf("%w: customer id is required", ErrInvalid)
	}
	if totalCents <= 0 {
		return Order{}, fmt.Errorf("%w: total must be positive, got %d", ErrInvalid, totalCents)
	}
	return Order{ID: id, CustomerID: customerID, TotalCents: totalCents, Status: StatusPending, CreatedAt: now}, nil
}

func (o *Order) MarkPaid() error {
	if o.Status != StatusPending {
		return fmt.Errorf("%w: %s -> %s", ErrInvalidStatus, o.Status, StatusPaid)
	}
	o.Status = StatusPaid
	return nil
}
```

## Ports

Declared by the consumer (the app package), with only the methods the use cases call. `Tx` hands the callback repositories bound to one transaction, so the app layer never sees `pgx.Tx`.

```go
// internal/app/ports.go
package app

import (
	"context"
	"time"

	"example.com/orders/internal/domain"
)

// Ports are declared here, next to the code that consumes them, and only
// contain the methods the use cases actually call.

type OrderRepository interface {
	Get(ctx context.Context, id domain.OrderID) (domain.Order, error)
	Insert(ctx context.Context, o domain.Order) error
	Update(ctx context.Context, o domain.Order) error
}

type Outbox interface {
	Add(ctx context.Context, subject string, payload any) error
}

// Tx is the set of repositories bound to one database transaction.
type Tx interface {
	Orders() OrderRepository
	Outbox() Outbox
}

// TxRunner runs fn inside a transaction: commit if fn returns nil, rollback otherwise.
type TxRunner interface {
	InTx(ctx context.Context, fn func(ctx context.Context, tx Tx) error) error
}

type Clock interface{ Now() time.Time }

type IDGenerator interface{ NewOrderID() domain.OrderID }
```

## Use cases

Orchestration only: build/load domain objects, call domain methods, persist through ports, record events. No SQL, no JSON, no HTTP.

```go
// internal/app/orders.go
package app

import (
	"context"
	"fmt"

	"example.com/orders/internal/domain"
)

type CreateOrderInput struct {
	CustomerID string
	TotalCents int64
}

type OrderService struct {
	orders OrderRepository
	tx     TxRunner
	clock  Clock
	ids    IDGenerator
}

func NewOrderService(orders OrderRepository, tx TxRunner, clock Clock, ids IDGenerator) *OrderService {
	return &OrderService{orders: orders, tx: tx, clock: clock, ids: ids}
}

func (s *OrderService) Get(ctx context.Context, id domain.OrderID) (domain.Order, error) {
	return s.orders.Get(ctx, id)
}

func (s *OrderService) Create(ctx context.Context, in CreateOrderInput) (domain.Order, error) {
	o, err := domain.NewOrder(s.ids.NewOrderID(), in.CustomerID, in.TotalCents, s.clock.Now())
	if err != nil {
		return domain.Order{}, err
	}
	err = s.tx.InTx(ctx, func(ctx context.Context, tx Tx) error {
		if err := tx.Orders().Insert(ctx, o); err != nil {
			return fmt.Errorf("insert order: %w", err)
		}
		return tx.Outbox().Add(ctx, "orders.created", OrderCreated{ID: string(o.ID), TotalCents: o.TotalCents})
	})
	if err != nil {
		return domain.Order{}, fmt.Errorf("create order: %w", err)
	}
	return o, nil
}

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

// Event payloads are application types; the outbox adapter serializes them.
type OrderCreated struct {
	ID         string `json:"id"`
	TotalCents int64  `json:"totalCents"`
}

type OrderPaid struct {
	ID string `json:"id"`
}
```

## Postgres adapter

Storage model, row mapping, pgx error translation, and the unit of work. The same repository type works on the pool or on a transaction because both satisfy `querier`. Events go to an outbox table in the same transaction; a separate relay publishes them to NATS (see the nats-jetstream-patterns skill).

```go
// internal/adapters/postgres/orders.go
package postgres

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jackc/pgx/v5/pgxpool"

	"example.com/orders/internal/app"
	"example.com/orders/internal/domain"
)

// querier is satisfied by both *pgxpool.Pool and pgx.Tx, so one repository
// implementation works inside and outside a transaction.
type querier interface {
	Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
	QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}

type OrderRepo struct{ q querier }

func NewOrderRepo(q querier) *OrderRepo { return &OrderRepo{q: q} }

// orderRow is the storage model; it may diverge from domain.Order (column
// names, nullable columns, denormalized fields) without touching the domain.
type orderRow struct {
	ID         string
	CustomerID string
	TotalCents int64
	Status     string
	CreatedAt  time.Time
}

func (r orderRow) toDomain() domain.Order {
	return domain.Order{
		ID: domain.OrderID(r.ID), CustomerID: r.CustomerID, TotalCents: r.TotalCents,
		Status: domain.OrderStatus(r.Status), CreatedAt: r.CreatedAt,
	}
}

func (r *OrderRepo) Get(ctx context.Context, id domain.OrderID) (domain.Order, error) {
	var row orderRow
	err := r.q.QueryRow(ctx,
		`SELECT id, customer_id, total_cents, status, created_at FROM orders WHERE id = $1`, id,
	).Scan(&row.ID, &row.CustomerID, &row.TotalCents, &row.Status, &row.CreatedAt)
	if errors.Is(err, pgx.ErrNoRows) {
		return domain.Order{}, fmt.Errorf("order %s: %w", id, domain.ErrNotFound)
	}
	if err != nil {
		return domain.Order{}, fmt.Errorf("select order %s: %w", id, err)
	}
	return row.toDomain(), nil
}

func (r *OrderRepo) Insert(ctx context.Context, o domain.Order) error {
	_, err := r.q.Exec(ctx,
		`INSERT INTO orders (id, customer_id, total_cents, status, created_at) VALUES ($1, $2, $3, $4, $5)`,
		o.ID, o.CustomerID, o.TotalCents, o.Status, o.CreatedAt)
	if isUniqueViolation(err) {
		return fmt.Errorf("order %s: %w", o.ID, domain.ErrConflict)
	}
	return err
}

func (r *OrderRepo) Update(ctx context.Context, o domain.Order) error {
	tag, err := r.q.Exec(ctx, `UPDATE orders SET status = $2 WHERE id = $1`, o.ID, o.Status)
	if err != nil {
		return err
	}
	if tag.RowsAffected() == 0 {
		return fmt.Errorf("order %s: %w", o.ID, domain.ErrNotFound)
	}
	return nil
}

func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505" // unique_violation
}

type OutboxRepo struct{ q querier }

func (r *OutboxRepo) Add(ctx context.Context, subject string, payload any) error {
	b, err := json.Marshal(payload)
	if err != nil {
		return fmt.Errorf("marshal outbox payload for %s: %w", subject, err)
	}
	_, err = r.q.Exec(ctx, `INSERT INTO outbox (subject, payload) VALUES ($1, $2)`, subject, b)
	return err
}

// TxRunner implements app.TxRunner on top of a pgx pool.
type TxRunner struct{ pool *pgxpool.Pool }

func NewTxRunner(pool *pgxpool.Pool) *TxRunner { return &TxRunner{pool: pool} }

type txRepos struct{ tx pgx.Tx }

func (t txRepos) Orders() app.OrderRepository { return &OrderRepo{q: t.tx} }
func (t txRepos) Outbox() app.Outbox          { return &OutboxRepo{q: t.tx} }

func (r *TxRunner) InTx(ctx context.Context, fn func(ctx context.Context, tx app.Tx) error) error {
	// BeginFunc commits when fn returns nil and rolls back on error or panic.
	return pgx.BeginFunc(ctx, r.pool, func(tx pgx.Tx) error {
		return fn(ctx, txRepos{tx: tx})
	})
}
```

## HTTP adapter

Transport model, decoding, mapping to use-case input and back, and error-to-status translation. The adapter declares its own narrow interface for the service. For the full problem-details error mapping, see the error-handling skill.

```go
// internal/adapters/http/orders.go
package http

import (
	"context"
	"encoding/json"
	"errors"
	"log/slog"
	"net/http"
	"time"

	"example.com/orders/internal/app"
	"example.com/orders/internal/domain"
)

// OrderService is the inbound port as seen by this adapter: only what the
// handlers call. *app.OrderService satisfies it implicitly.
type OrderService interface {
	Get(ctx context.Context, id domain.OrderID) (domain.Order, error)
	Create(ctx context.Context, in app.CreateOrderInput) (domain.Order, error)
}

type Handler struct {
	orders OrderService
	log    *slog.Logger
}

func NewHandler(orders OrderService, log *slog.Logger) *Handler {
	return &Handler{orders: orders, log: log}
}

func (h *Handler) Routes(mux *http.ServeMux) {
	mux.HandleFunc("GET /orders/{id}", h.getOrder)
	mux.HandleFunc("POST /orders", h.createOrder)
}

// Transport models: JSON names and formats belong to the API contract, not the domain.
type createOrderRequest struct {
	CustomerID string `json:"customerId"`
	TotalCents int64  `json:"totalCents"`
}

type orderResponse struct {
	ID         string    `json:"id"`
	CustomerID string    `json:"customerId"`
	TotalCents int64     `json:"totalCents"`
	Status     string    `json:"status"`
	CreatedAt  time.Time `json:"createdAt"`
}

func toResponse(o domain.Order) orderResponse {
	return orderResponse{
		ID: string(o.ID), CustomerID: o.CustomerID, TotalCents: o.TotalCents,
		Status: string(o.Status), CreatedAt: o.CreatedAt,
	}
}

func (h *Handler) getOrder(w http.ResponseWriter, r *http.Request) {
	o, err := h.orders.Get(r.Context(), domain.OrderID(r.PathValue("id")))
	if err != nil {
		h.writeError(w, r, err)
		return
	}
	h.writeJSON(w, http.StatusOK, toResponse(o))
}

func (h *Handler) createOrder(w http.ResponseWriter, r *http.Request) {
	var req createOrderRequest
	dec := json.NewDecoder(http.MaxBytesReader(w, r.Body, 1<<20))
	dec.DisallowUnknownFields()
	if err := dec.Decode(&req); err != nil {
		// Syntax-level validation lives in the adapter: the use case never sees malformed input.
		http.Error(w, "malformed JSON body", http.StatusBadRequest)
		return
	}
	o, err := h.orders.Create(r.Context(), app.CreateOrderInput{CustomerID: req.CustomerID, TotalCents: req.TotalCents})
	if err != nil {
		h.writeError(w, r, err)
		return
	}
	h.writeJSON(w, http.StatusCreated, toResponse(o))
}

func (h *Handler) writeError(w http.ResponseWriter, r *http.Request, err error) {
	switch {
	case errors.Is(err, domain.ErrNotFound):
		http.Error(w, "not found", http.StatusNotFound)
	case errors.Is(err, domain.ErrInvalid):
		http.Error(w, err.Error(), http.StatusUnprocessableEntity)
	case errors.Is(err, domain.ErrConflict):
		http.Error(w, "conflict", http.StatusConflict)
	default:
		h.log.ErrorContext(r.Context(), "request failed", "method", r.Method, "path", r.URL.Path, "err", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}

func (h *Handler) writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		// Headers are already sent; the client most likely went away.
		h.log.Debug("write response", "err", err)
	}
}
```

## Composition root

Every dependency is constructed and wired explicitly in `run`. Nothing is registered globally; reading this function tells you the whole graph.

```go
// cmd/orders/main.go
package main

import (
	"context"
	"crypto/rand"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	httpadapter "example.com/orders/internal/adapters/http"
	"example.com/orders/internal/adapters/postgres"
	"example.com/orders/internal/app"
	"example.com/orders/internal/domain"
)

type systemClock struct{}

func (systemClock) Now() time.Time { return time.Now().UTC() }

type randomIDs struct{}

func (randomIDs) NewOrderID() domain.OrderID { return domain.OrderID(rand.Text()) }

func main() {
	if err := run(); err != nil {
		slog.Error("service stopped", "err", err)
		os.Exit(1)
	}
}

// run is the composition root: every concrete adapter is built and wired here,
// in plain code, so the whole dependency graph can be read top to bottom.
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

	srv := &http.Server{Addr: ":8080", Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	errCh := make(chan error, 1)
	go func() { errCh <- srv.ListenAndServe() }()

	select {
	case err := <-errCh:
		return fmt.Errorf("http server: %w", err)
	case <-ctx.Done():
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil && !errors.Is(err, http.ErrServerClosed) {
		return fmt.Errorf("http shutdown: %w", err)
	}
	return nil
}
```

## Use-case test with fakes

Fakes are tiny hand-written structs. The fake `InTx` just calls the function; real commit/rollback behavior is tested against Postgres in the adapter's integration tests.

```go
// internal/app/orders_test.go
package app_test

import (
	"context"
	"errors"
	"testing"
	"time"

	"example.com/orders/internal/app"
	"example.com/orders/internal/domain"
)

type fakeOrders struct {
	byID map[domain.OrderID]domain.Order
}

func (f *fakeOrders) Get(_ context.Context, id domain.OrderID) (domain.Order, error) {
	o, ok := f.byID[id]
	if !ok {
		return domain.Order{}, domain.ErrNotFound
	}
	return o, nil
}
func (f *fakeOrders) Insert(_ context.Context, o domain.Order) error {
	if _, ok := f.byID[o.ID]; ok {
		return domain.ErrConflict
	}
	f.byID[o.ID] = o
	return nil
}
func (f *fakeOrders) Update(_ context.Context, o domain.Order) error { f.byID[o.ID] = o; return nil }

type fakeOutbox struct{ subjects []string }

func (f *fakeOutbox) Add(_ context.Context, subject string, _ any) error {
	f.subjects = append(f.subjects, subject)
	return nil
}

// fakeTx runs fn directly; enough for use-case tests. Commit/rollback
// semantics are covered by the postgres adapter's integration tests.
type fakeTx struct {
	orders *fakeOrders
	outbox *fakeOutbox
}

func (f *fakeTx) Orders() app.OrderRepository { return f.orders }
func (f *fakeTx) Outbox() app.Outbox          { return f.outbox }
func (f *fakeTx) InTx(ctx context.Context, fn func(context.Context, app.Tx) error) error {
	return fn(ctx, f)
}

type fixedClock time.Time

func (c fixedClock) Now() time.Time { return time.Time(c) }

type fixedID domain.OrderID

func (id fixedID) NewOrderID() domain.OrderID { return domain.OrderID(id) }

func TestCreateOrder(t *testing.T) {
	tests := []struct {
		name        string
		in          app.CreateOrderInput
		wantErr     error
		wantSubject []string
	}{
		{name: "valid", in: app.CreateOrderInput{CustomerID: "c1", TotalCents: 100}, wantSubject: []string{"orders.created"}},
		{name: "zero total", in: app.CreateOrderInput{CustomerID: "c1"}, wantErr: domain.ErrInvalid},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			orders := &fakeOrders{byID: map[domain.OrderID]domain.Order{}}
			tx := &fakeTx{orders: orders, outbox: &fakeOutbox{}}
			svc := app.NewOrderService(orders, tx, fixedClock(time.Unix(0, 0)), fixedID("o1"))

			_, err := svc.Create(context.Background(), tt.in)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("Create() err = %v, want %v", err, tt.wantErr)
			}
			if len(tx.outbox.subjects) != len(tt.wantSubject) {
				t.Fatalf("published %v, want %v", tx.outbox.subjects, tt.wantSubject)
			}
		})
	}
}
```
