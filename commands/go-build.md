---
description: Fix Go build errors, go vet findings, and golangci-lint issues with minimal, surgical changes. Delegates to the go-build-resolver agent.
argument-hint: "[package path, e.g. ./internal/orders/...]"
allowed-tools: Bash(go build:*), Bash(go vet:*), Bash(golangci-lint run:*), Bash(go test:*), Bash(go mod tidy:*), Bash(command -v:*)
---

# Go Build and Fix

Get the Go code compiling and vet/lint-clean with the smallest possible diff.

Target: `$ARGUMENTS` if given, otherwise `./...`.

## Steps

1. Run the diagnostics against the target, in this order, and capture the output:
   - `go build <target>`
   - `go vet <target>`
   - `golangci-lint run <target>` only if `command -v golangci-lint` succeeds. If it is not installed, say so in the report and skip it; do not install it.

   These commands are read-only. Do not run `go mod tidy` as a diagnostic: it rewrites `go.mod`/`go.sum`, so it belongs in the fix phase, and only when the error is actually a module problem (missing or unused dependency, `missing go.sum entry`).

2. If everything is clean, report that and stop.

3. Otherwise delegate to the **go-build-resolver** agent. Pass it the target and the full diagnostic output. Instruct it to fix in this order, because later categories are noisy or meaningless until earlier ones are resolved:
   1. Compile errors (code must build before vet/lint can run)
   2. `go vet` findings (likely bugs)
   3. golangci-lint findings (style and correctness)

   One fix at a time, re-running `go build` after each.

4. When the agent finishes, run `go build <target>` and `go vet <target>` yourself to confirm, then `go test <target>` to check nothing broke.

## Rules for fixes

- Fix the root cause; do not suppress. No `//nolint`, no blank identifiers to silence "declared and not used" on values that should be used, no deleted code just to make an error go away. Ask the user before adding a `//nolint` directive.
- Never discard errors to make types line up. A type mismatch from a parse call is fixed by handling the error:

  ```go
  count, err := strconv.Atoi(r.URL.Query().Get("count"))
  if err != nil {
      http.Error(w, "count must be an integer", http.StatusBadRequest)
      return
  }
  ```

- Do not change exported signatures or refactor unless the error cannot be fixed otherwise; if that is the case, stop and explain.

## Stop and report when

- The same error persists after 3 attempts.
- A fix introduces more errors than it removes.
- The fix requires an architectural change (for example breaking an import cycle across many packages) or a dependency that cannot be fetched.

## Report format

```text
Target: ./...
[FIXED] internal/orders/handler.go:42  undefined: OrderStore -> added import of internal/orders/store
[FIXED] internal/orders/service.go:88  vet: printf arg mismatch -> %d to %s
[OPEN]  internal/users/repo.go:17      import cycle users -> orders -> users (needs design decision)

Build: OK | Vet: OK | Lint: 2 remaining (or: not installed) | Tests: OK
Files modified: internal/orders/handler.go, internal/orders/service.go
```

After a clean build, suggest `/go-test` or `/go-review` as the next step.
