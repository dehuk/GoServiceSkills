---
description: Security review of Go code - runs govulncheck and gosec, then delegates a manual review of the given paths or the current diff to the security-reviewer agent and reports findings by severity.
argument-hint: "[paths or packages to review; defaults to the git diff]"
allowed-tools: Bash(govulncheck:*), Bash(gosec:*), Bash(git diff:*), Bash(git status:*), Bash(go mod verify:*), Bash(command -v:*)
---

# Go Security Review

## 1. Determine scope

- If `$ARGUMENTS` is non-empty, review those paths or packages.
- Otherwise review the uncommitted changes: `git diff HEAD` (plus untracked files from `git status --porcelain`). If there are none, use the current branch against `main` (`git diff main...HEAD`). If that is also empty, ask the user what to review rather than auditing the whole repository unprompted.

Include non-Go files in scope when they changed: Dockerfiles, compose files, CI config, NATS/Redis config, and `go.mod`/`go.sum` all carry security risk.

## 2. Run the scanners

Check each tool with `command -v`. If one is missing, do not install it; tell the user how and continue with the rest:

- `go install golang.org/x/vuln/cmd/govulncheck@latest`
- `go install github.com/securego/gosec/v2/cmd/gosec@latest`

Then run, from the module root:

1. `govulncheck ./...` - always run it on the whole module, since a vulnerable dependency or Go stdlib version affects every package regardless of the diff. Only findings in the "Your code is affected" section matter for severity; mention the "your code does not appear to call" findings briefly.
2. `gosec -quiet ./...` when reviewing the whole module, or `gosec -quiet` on the affected package directories (e.g. `gosec -quiet ./internal/orders/...`) when reviewing a diff.
3. `go mod verify` if `go.mod` or `go.sum` changed.

Keep the raw output for the next step. A non-zero exit code from a scanner usually means it found something, not that it failed; read the output before concluding.

## 3. Delegate the manual review

Run the **security-reviewer** agent with:

- the scope from step 1 (paths, or the diff command to use),
- the scanner output from step 2,
- a note on which tools were unavailable.

Ask it to confirm or dismiss each scanner finding against the code, and to review the logic that scanners miss: authorization and tenant scoping, trust boundaries (including broker messages and cache values), check-then-act races, fail-open error paths, and information leaks. The go-security skill is its reference.

## 4. Report

Merge scanner results and the agent's findings, drop duplicates, and group by severity: CRITICAL, HIGH, MEDIUM, LOW. For each finding give `file:line`, the issue (input -> sink), and a concrete fix. List dismissed scanner findings separately with a one-line reason each.

```text
Scope: git diff HEAD (5 files)
Scanners: govulncheck 1 affected | gosec 3 issues (2 dismissed) | go mod verify OK

[CRITICAL] internal/orders/repo.go:88  ORDER BY built from ?sort= query parameter via fmt.Sprintf
  Fix: map sort through an allowlist of column constants
[HIGH] go.mod  golang.org/x/net v0.20.0: GO-2024-XXXX reachable via http2 server
  Fix: go get golang.org/x/net@latest && go mod tidy
[MEDIUM] cmd/api/main.go:41  http.ListenAndServe without timeouts
  Fix: use http.Server with ReadHeaderTimeout, ReadTimeout, WriteTimeout, IdleTimeout

Dismissed: G401 internal/cache/key.go:12 (sha1 used for cache key, not security)

CRITICAL: 1 | HIGH: 1 | MEDIUM: 1 | LOW: 0
Verdict: Request changes
```

Verdict: **Request changes** for any CRITICAL or HIGH finding, **Approve with comments** for MEDIUM/LOW only, **Approve** when clean. If a secret was exposed, say explicitly that it must be rotated.

Do not edit code during the review; offer to apply the fixes afterwards.
