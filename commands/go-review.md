---
description: Review changed Go code for correctness, error handling, concurrency safety, security, and idiomatic style. Delegates to the go-reviewer agent and, when relevant, silent-failure-hunter and security-reviewer.
argument-hint: "[paths or packages to review; defaults to the git diff]"
allowed-tools: Bash(git diff:*), Bash(git status:*), Bash(go build:*), Bash(go vet:*), Bash(golangci-lint run:*), Bash(govulncheck:*), Bash(gosec:*), Bash(command -v:*)
---

# Go Code Review

## 1. Determine scope

- If `$ARGUMENTS` is non-empty, review those paths or packages.
- Otherwise review the uncommitted changes: `git diff HEAD -- '*.go' go.mod go.sum` (plus untracked `.go` files from `git status --porcelain`). If there are none, review the diff of the current branch against `main` (`git diff main...HEAD`). If that is also empty, ask the user what to review instead of reviewing the whole repo.

Review the changed code and the code it directly touches; do not report pre-existing issues in unrelated files, because they bury the findings the author can act on now.

## 2. Run automated checks on the affected packages

- `go build` and `go vet` (always).
- `golangci-lint run` if installed.
- `govulncheck ./...` if installed and `go.mod`/`go.sum` changed.
- `gosec` on the affected packages if installed and the change touches input handling, SQL, exec, crypto, files, or auth.

Check installation with `command -v`. If a tool is missing, note it in the report and continue; do not install tools.

## 3. Delegate

- Always run the **go-reviewer** agent with the scope and the tool output.
- Also run **silent-failure-hunter** in parallel when the diff touches error paths: new `if err != nil` branches, `defer ... Close()`, goroutines/errgroup, retries, fallbacks, or logging of errors.
- Also run **security-reviewer** in parallel when the diff touches HTTP handlers, authn/authz, SQL, message consumers that act on external input, secrets/config, crypto, or file/exec handling.

Merge the results, drop duplicates, and keep the highest severity when agents disagree.

## 4. Report

Severity:

- **CRITICAL**: security holes, data races, data loss or corruption, ignored errors on a path that mutates state.
- **HIGH**: lost error context, goroutine/resource leaks, missing context propagation or cancellation, panics on recoverable errors, missing tests for new behavior.
- **MEDIUM**: non-idiomatic code, performance issues, readability, missing docs on exported identifiers.

For each finding give `file:line`, the problem, why it matters, and a concrete fix (short Go snippet when it helps).

End with a verdict:

- **Approve**: no CRITICAL or HIGH findings.
- **Approve with comments**: MEDIUM findings only.
- **Request changes**: any CRITICAL or HIGH finding.

```text
Scope: git diff HEAD (4 files)
Checks: build OK | vet OK | golangci-lint 1 issue | govulncheck not installed

[CRITICAL] internal/orders/cache.go:31  map written from multiple goroutines without a lock
  Fix: guard with sync.RWMutex or confine to one goroutine
[HIGH] internal/orders/service.go:58  `return err` loses which order failed
  Fix: return fmt.Errorf("load order %s: %w", id, err)

CRITICAL: 1 | HIGH: 1 | MEDIUM: 0
Verdict: Request changes
```

Do not edit code during the review; offer to apply fixes afterwards.
