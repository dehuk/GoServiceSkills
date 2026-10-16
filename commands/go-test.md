---
description: Implement a Go feature or bug fix test-first (TDD) - write failing tests, make them pass with minimal code, refactor, then check race-free coverage. Pass the feature or bug description as the argument.
argument-hint: "<feature or bug to implement test-first>"
allowed-tools: Bash(go test:*), Bash(go build:*), Bash(go vet:*), Bash(go tool cover:*)
---

# Go TDD Workflow

Implement the following test-first: **$ARGUMENTS**

If `$ARGUMENTS` is empty, ask the user what feature or bug to work on. This command drives a TDD cycle for one change; it is not a generic "run the tests" command.

Use the **golang-testing** skill for test patterns (table-driven tests, subtests, helpers, fakes, fuzzing) and for coverage targets. Do not reinvent them here.

## Steps

1. **Locate and scope.** Find the package where the change belongs. Read the existing code and tests there. State in 2-4 bullets what behavior you will add or fix, including edge and error cases.

2. **Define the surface.** Add or adjust the function signature, type, or interface with a stub body that compiles (return zero values plus `errors.New("not implemented")`, not `panic`, so the failing tests report cleanly instead of aborting the run).

3. **RED: write the tests first.** Cover the happy path, boundaries, invalid input, and error propagation. For a bug fix, write a test that reproduces the bug. Use interface fakes for dependencies in unit tests; leave real Postgres/NATS/Redis to integration tests.

4. **Run and confirm the failure:** `go test -race -run '<TestName>' ./<pkg>/`. The tests must fail because the behavior is missing, not because of a compile error or a broken test. If they pass already, the test is not testing the new behavior; fix the test.

5. **GREEN: implement the minimum** code to make the tests pass. Handle every error. Re-run the same command until it passes.

6. **REFACTOR** with tests green: remove duplication, improve names, wrap errors with context. Re-run after each change.

7. **Verify the package:**
   - `go test -race -cover ./<pkg>/...`
   - `go vet ./<pkg>/...`

   Always use `-race`: concurrency bugs that only show up under the race detector are cheap to fix now and expensive in production. Compare coverage to the golang-testing targets; if it is below, add tests for the uncovered branches (use `go test -coverprofile` and `go tool cover -func` to find them) rather than padding with trivial tests.

## Rules

- Do not write implementation code before a failing test exists for it; the point of the RED step is proving the test can fail.
- Do not weaken or delete an existing test to make it pass. If an existing test conflicts with the requested behavior, stop and ask.
- Do not use `time.Sleep` for synchronization in tests; it makes them slow and flaky. Use channels, contexts, or fakes with controllable clocks.

## Report

```text
Feature: <one line>
Tests added: TestOrderTotal (7 cases), TestOrderTotal_InvalidCurrency
RED: failed as expected (not implemented)
GREEN: pass
go test -race -cover ./internal/orders/...  ok  coverage: 91.3%
Files: internal/orders/total.go, internal/orders/total_test.go
```

Suggest `/go-review` as the next step.
