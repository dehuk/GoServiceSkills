# Go tooling

## Commands

```bash
# Build and run
go build ./...
go run ./cmd/myservice

# Format (golangci-lint fmt does the same using the formatters in the config)
gofmt -l .                 # list unformatted files; non-empty output should fail CI
goimports -w .

# Static analysis
go vet ./...
golangci-lint run ./...
golangci-lint run --fix ./...   # apply auto-fixes

# Tests (details in the golang-testing skill)
go test -race ./...

# Modules
go mod tidy                # run after adding/removing imports; CI should fail on a diff
go mod verify
go list -m -u all          # available dependency updates

# Tools pinned in go.mod (Go 1.24+), instead of tools.go
go get -tool github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest
go tool golangci-lint run ./...
```

Pinning tools with `go get -tool` records them in `go.mod`, so every developer and CI run uses the same version. Note: the golangci-lint project recommends its binary releases over `go install`/`go tool` builds; either works, but pin the version in both cases.

Security tooling (gosec, govulncheck) is covered by the **go-security** skill.

## golangci-lint v2 configuration

v1 configs do not load in v2. Key differences: the file starts with `version: "2"`; `linters-settings` moved to `linters.settings`; `gofmt`/`goimports` moved to a separate `formatters` section; `gosimple` and `stylecheck` were merged into `staticcheck`; `enable-all`/`disable-all` became `linters.default: all|none|standard`; `issues.exclude-*` moved to `linters.exclusions`. Migrate an old file with `golangci-lint migrate`.

Recommended starting point for a service:

```yaml
version: "2"

run:
  timeout: 5m

linters:
  default: standard        # errcheck, govet, ineffassign, staticcheck, unused
  enable:
    - bodyclose            # unclosed HTTP response bodies leak connections
    - errorlint            # == / type switches on wrapped errors
    - gocritic
    - misspell
    - nilerr               # returning nil after checking err != nil
    - noctx                # HTTP requests / DB calls without context
    - revive
    - unconvert
    - unparam
    - gosec                # configuration owned by the go-security skill
  settings:
    errcheck:
      check-type-assertions: true
    govet:
      enable:
        - shadow
  exclusions:
    generated: lax
    presets:
      - comments
      - std-error-handling
    rules:
      - path: _test\.go
        linters:
          - gosec
          - unparam

formatters:
  enable:
    - gofmt
    - goimports
  settings:
    goimports:
      local-prefixes:
        - github.com/your-org/your-module
```

Do not silence findings with blanket exclusions. Use a targeted `//nolint:lintername // reason` on the line, so the exception is reviewed together with the code.

## Edit loop for agents and editors

After each edit to `.go` files:

1. `goimports -w <files>` (or `gofmt -w`) so formatting never reaches review.
2. `go vet ./<changed packages>/...` for fast correctness checks.
3. `golangci-lint run ./<changed packages>/...` for the extended checks.
4. `go test -race ./<changed packages>/...`.

Before declaring the work done, run the same on `./...` and `go mod tidy`. If the project uses Claude Code hooks, steps 1-2 are good candidates for a PostToolUse hook on Edit/Write of `*.go` files.

## Makefile skeleton

```makefile
.PHONY: fmt lint test check

fmt:
	golangci-lint fmt

lint:
	golangci-lint run ./...

test:
	go test -race -shuffle=on ./...

check: lint test
	go mod tidy -diff
```

`go mod tidy -diff` (Go 1.23+) exits non-zero when `go.mod`/`go.sum` are not tidy, without modifying them.
