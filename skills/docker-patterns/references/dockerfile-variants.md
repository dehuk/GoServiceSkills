# Dockerfile Variants for Go Services

Variations on the production Dockerfile in SKILL.md. Each section replaces or extends one part of it.

## scratch Instead of distroless

`scratch` is an empty filesystem. It saves about 2 MB over `distroless/static` but you must supply everything the Go runtime may read from disk: CA certificates (TLS), timezone data (`time.LoadLocation`), and a passwd entry if anything resolves the current user.

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25-alpine AS build
RUN apk add --no-cache ca-certificates tzdata
# Minimal passwd/group so the runtime user has a name; UID 65532 matches distroless.
RUN echo 'app:x:65532:65532::/nonexistent:/sbin/nologin' > /etc/passwd.app && \
    echo 'app:x:65532:' > /etc/group.app
# ... build steps as in SKILL.md, producing /out/server

FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /etc/passwd.app /etc/passwd
COPY --from=build /etc/group.app /etc/group
COPY --from=build /out/server /app/server
USER 65532:65532
ENTRYPOINT ["/app/server"]
```

Alternatively embed timezone data in the binary with `import _ "time/tzdata"` in `package main` (adds about 450 KB) and drop the zoneinfo copy. Prefer distroless unless image size is a hard constraint: it gets the same result with less to get wrong.

Use a numeric `USER` on scratch. Kubernetes `runAsNonRoot: true` can only verify a numeric UID; a name it cannot resolve makes the pod fail admission.

## Private Modules via Build Secrets

Private Go modules need credentials during `go mod download`. Pass them as a BuildKit secret: it is mounted only for that `RUN` and never written to a layer.

```dockerfile
ENV GOPRIVATE=github.com/acme/*
COPY go.mod go.sum ./
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    --mount=type=cache,target=/go/pkg/mod \
    go mod download
```

```bash
docker build --secret id=netrc,src=$HOME/.netrc -t orders-api .
```

For SSH-based access use `RUN --mount=type=ssh` with `git config --global url."git@github.com:".insteadOf "https://github.com/"` and `docker build --ssh default`. The alpine golang image needs `apk add --no-cache git openssh-client` for this.

Do not use `ARG GITHUB_TOKEN`: build args are visible in `docker history` and in the image config of every intermediate stage pushed to a cache registry.

## Several Binaries from One Dockerfile

A repository with `cmd/api`, `cmd/worker` and `cmd/migrator` can share one Dockerfile via the `SERVICE` build arg (as in SKILL.md). With `docker buildx bake` all images share the module and build caches:

```hcl
# docker-bake.hcl
group "default" { targets = ["api", "worker"] }

target "_common" {
  context   = "."
  platforms = ["linux/amd64", "linux/arm64"]
  args      = { VERSION = "dev" }
}

target "api" {
  inherits = ["_common"]
  args     = { SERVICE = "api" }
  tags     = ["registry.example.com/orders-api:dev"]
}

target "worker" {
  inherits = ["_common"]
  args     = { SERVICE = "worker" }
  tags     = ["registry.example.com/orders-worker:dev"]
}
```

`docker buildx bake` builds both; the `go build` step differs only in the package path, so the dependency layer is built once.

## Debug Image

When you need a shell next to the binary (investigating DNS, mounted files, env), add a target on the `:debug-nonroot` distroless tag, which includes busybox:

```dockerfile
FROM gcr.io/distroless/static-debian12:debug-nonroot AS debug
COPY --from=build /out/server /app/server
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]

FROM gcr.io/distroless/static-debian12:nonroot AS release
COPY --from=build /out/server /app/server
USER nonroot:nonroot
ENTRYPOINT ["/app/server"]
```

Build with `--target debug` only when investigating. Put the `release` stage last so a plain `docker build` produces the production image. For delve debugging, build without `-s -w` and with `-gcflags "all=-N -l"`; that belongs in the dev workflow, not in a production target.

## Reproducibility Checklist

- Go version pinned via `ARG GO_VERSION` and kept in sync with the `go`/`toolchain` line in `go.mod` (CI can compare them).
- Base images pinned by digest in CI.
- `-trimpath` and injected version, no timestamps in the binary (`-X main.buildTime` makes every build differ; use the commit time if you need a date).
- `go.sum` committed; `go mod download` runs with `GOFLAGS=-mod=readonly` (the default for builds) so the Dockerfile cannot modify dependencies.
