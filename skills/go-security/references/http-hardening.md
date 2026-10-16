# HTTP Layer Hardening

Complete, copyable building blocks for the HTTP side of a Go service. Standard library unless noted.

## Contents

- Server and listeners
- Decoding request bodies safely
- Security headers
- CORS
- CSRF
- Client IP behind proxies
- Rate limiting
- TLS and mTLS
- SSRF-safe outbound client

## Server and Listeners

Public traffic and operational endpoints (pprof, metrics, health) go on separate listeners, so debug handlers are never reachable from outside even if a route is misconfigured.

```go
func newPublicServer(addr string, h http.Handler) *http.Server {
	return &http.Server{
		Addr:              addr,
		Handler:           h,
		ReadHeaderTimeout: 5 * time.Second,  // Slowloris: limits time to send headers
		ReadTimeout:       15 * time.Second, // whole request including body
		WriteTimeout:      30 * time.Second, // raise per route with http.ResponseController for streaming
		IdleTimeout:       120 * time.Second,
		MaxHeaderBytes:    1 << 20,
		ErrorLog:          slog.NewLogLogger(slog.Default().Handler(), slog.LevelWarn),
	}
}

func newAdminMux() *http.ServeMux {
	mux := http.NewServeMux()
	// Register pprof explicitly on a private mux instead of relying on the
	// side-effect registration on http.DefaultServeMux.
	mux.HandleFunc("GET /debug/pprof/", pprof.Index)
	mux.HandleFunc("GET /debug/pprof/profile", pprof.Profile)
	mux.HandleFunc("GET /debug/pprof/trace", pprof.Trace)
	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusNoContent)
	})
	return mux
}
```

Bind the admin server to an internal interface or port that is not published (`127.0.0.1:6060`, or a port not exposed by the container/ingress).

For long streaming responses, extend the deadline per request instead of disabling `WriteTimeout` globally:

```go
rc := http.NewResponseController(w)
if err := rc.SetWriteDeadline(time.Now().Add(10 * time.Minute)); err != nil {
	// Underlying writer does not support deadlines; keep the server default.
	slog.WarnContext(r.Context(), "cannot extend write deadline", "err", err)
}
```

## Decoding Request Bodies Safely

```go
const maxBodyBytes = 1 << 20 // 1 MiB; set per route for uploads

var errBodyTooLarge = errors.New("request body too large")

func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) error {
	r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // surfaces client bugs and mass-assignment attempts early

	if err := dec.Decode(dst); err != nil {
		var tooLarge *http.MaxBytesError
		if errors.As(err, &tooLarge) {
			return errBodyTooLarge // map to 413
		}
		return fmt.Errorf("decode body: %w", err) // map to 400 with a generic message
	}
	// Reject trailing data such as `{"a":1}{"b":2}`.
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return errors.New("body must contain a single JSON object")
	}
	return nil
}
```

For multipart uploads, call `r.ParseMultipartForm(maxMemory)` after `MaxBytesReader`, check the declared and sniffed content type (`http.DetectContentType` on the first 512 bytes), and store under a generated name.

## Security Headers

For a JSON API most browser headers are cheap insurance; for anything that serves HTML they are essential.

```go
func securityHeaders(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := w.Header()
		h.Set("X-Content-Type-Options", "nosniff")
		h.Set("X-Frame-Options", "DENY")
		h.Set("Referrer-Policy", "no-referrer")
		h.Set("Content-Security-Policy", "default-src 'none'; frame-ancestors 'none'")
		h.Set("Cross-Origin-Resource-Policy", "same-origin")
		// Only when every client reaches the service over HTTPS (usually set at the edge).
		h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
		h.Set("Cache-Control", "no-store") // API responses often carry personal data
		next.ServeHTTP(w, r)
	})
}
```

Relax `Content-Security-Policy` per route when the service actually serves HTML, scripts, or images.

## CORS

CORS is a browser relaxation of the same-origin policy, not a server-side access control. Misconfiguring it lets any website make authenticated calls on a user's behalf.

```go
func cors(allowed []string) func(http.Handler) http.Handler {
	allow := make(map[string]bool, len(allowed))
	for _, o := range allowed {
		allow[o] = true
	}
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Header().Add("Vary", "Origin") // caches must not reuse one origin's answer for another
			origin := r.Header.Get("Origin")
			if origin != "" && allow[origin] {
				w.Header().Set("Access-Control-Allow-Origin", origin)
				w.Header().Set("Access-Control-Allow-Credentials", "true")
				if r.Method == http.MethodOptions && r.Header.Get("Access-Control-Request-Method") != "" {
					w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE")
					w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type")
					w.Header().Set("Access-Control-Max-Age", "600")
					w.WriteHeader(http.StatusNoContent)
					return
				}
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

Pitfalls: echoing any `Origin` back, matching with `strings.HasSuffix(origin, "example.com")` (matches `evilexample.com`), allowing `null`, and `*` together with credentials.

## CSRF

Only relevant when the browser sends credentials automatically (cookies). Go 1.25+ ships a defense based on `Sec-Fetch-Site`/`Origin`:

```go
csrf := http.NewCrossOriginProtection()
if err := csrf.AddTrustedOrigin("https://app.example.com"); err != nil {
	return fmt.Errorf("csrf trusted origin: %w", err)
}
handler := csrf.Handler(mux) // rejects cross-origin non-safe methods (POST, PUT, DELETE, ...)
```

Keep GET/HEAD side-effect free, since they are always allowed. Session cookies: `HttpOnly: true, Secure: true, SameSite: http.SameSiteLaxMode` (or `Strict`), a narrow `Path`, and no `Domain` unless subdomains truly need it.

## Client IP Behind Proxies

`X-Forwarded-For` is client-controlled: the leftmost value is whatever the caller sent. Using it for rate limiting or audit logs lets attackers spoof any IP.

```go
// clientIP trusts X-Forwarded-For only when the direct peer is a known proxy, and then takes
// the rightmost address that is not itself a trusted proxy.
func clientIP(r *http.Request, trusted []netip.Prefix) netip.Addr {
	peer, err := netip.ParseAddrPort(r.RemoteAddr)
	if err != nil {
		return netip.Addr{}
	}
	ip := peer.Addr().Unmap()
	if !inPrefixes(ip, trusted) {
		return ip
	}
	hops := strings.Split(r.Header.Get("X-Forwarded-For"), ",")
	for i := len(hops) - 1; i >= 0; i-- {
		a, err := netip.ParseAddr(strings.TrimSpace(hops[i]))
		if err != nil {
			return ip // malformed chain: fall back to the proxy address
		}
		a = a.Unmap()
		if !inPrefixes(a, trusted) {
			return a
		}
	}
	return ip
}

func inPrefixes(a netip.Addr, ps []netip.Prefix) bool {
	for _, p := range ps {
		if p.Contains(a) {
			return true
		}
	}
	return false
}
```

## Rate Limiting

In-process limiting with `golang.org/x/time/rate` works for a single instance or as a first line of defense. Bound the map and evict idle entries, or the limiter itself becomes a memory-exhaustion vector.

```go
type keyedLimiter struct {
	mu      sync.Mutex
	entries map[string]*limiterEntry
	limit   rate.Limit
	burst   int
}

type limiterEntry struct {
	lim      *rate.Limiter
	lastSeen time.Time
}

func newKeyedLimiter(limit rate.Limit, burst int) *keyedLimiter {
	return &keyedLimiter{entries: make(map[string]*limiterEntry), limit: limit, burst: burst}
}

func (k *keyedLimiter) Allow(key string) bool {
	k.mu.Lock()
	defer k.mu.Unlock()
	e, ok := k.entries[key]
	if !ok {
		e = &limiterEntry{lim: rate.NewLimiter(k.limit, k.burst)}
		k.entries[key] = e
	}
	e.lastSeen = time.Now()
	return e.lim.Allow()
}

// Sweep removes idle keys; run it from a ticker goroutine tied to the server's context.
func (k *keyedLimiter) Sweep(idle time.Duration) {
	cutoff := time.Now().Add(-idle)
	k.mu.Lock()
	defer k.mu.Unlock()
	maps.DeleteFunc(k.entries, func(_ string, e *limiterEntry) bool { return e.lastSeen.Before(cutoff) })
}

func (k *keyedLimiter) Middleware(keyFn func(*http.Request) string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if !k.Allow(keyFn(r)) {
				w.Header().Set("Retry-After", "1")
				http.Error(w, http.StatusText(http.StatusTooManyRequests), http.StatusTooManyRequests)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```

- Key by authenticated principal where possible, by `clientIP` otherwise.
- Use stricter limits on login, password reset, OTP verification, and expensive search/export endpoints. Also limit per account (not only per IP) on login, to stop distributed credential stuffing.
- With multiple replicas, enforce shared limits in Redis (atomic `INCR` + `EXPIRE` in a Lua script or a GCRA library); see the redis-patterns skill.

## TLS and mTLS

Usually TLS terminates at an ingress or load balancer. When the service terminates TLS itself, or calls other services over TLS:

```go
serverTLS := &tls.Config{
	MinVersion: tls.VersionTLS12, // Go's default cipher suites for 1.2/1.3 are already safe
}

// Service-to-service mTLS: only clients with a cert signed by the internal CA get in.
mtls := &tls.Config{
	MinVersion: tls.VersionTLS13,
	ClientAuth: tls.RequireAndVerifyClientCert,
	ClientCAs:  internalCAPool,
}

// Outbound client trusting a private CA - never InsecureSkipVerify.
clientTLS := &tls.Config{
	MinVersion: tls.VersionTLS12,
	RootCAs:    internalCAPool,
}
```

`InsecureSkipVerify: true` disables all server authentication; anyone on the path can impersonate the server. If a test needs it, keep it in test code, not behind a config flag that can reach production. Do not hand-pick `CipherSuites` unless a compliance rule demands it - the defaults are maintained by the Go team.

## SSRF-Safe Outbound Client

For any request whose destination is influenced by input. The `Control` hook sees the resolved IP for every connection attempt, including after redirects and DNS re-resolution.

```go
var blockedPrefixes = []netip.Prefix{
	netip.MustParsePrefix("0.0.0.0/8"),
	netip.MustParsePrefix("100.64.0.0/10"), // carrier-grade NAT, often internal
	netip.MustParsePrefix("192.0.0.0/24"),
	netip.MustParsePrefix("198.18.0.0/15"),
	netip.MustParsePrefix("64:ff9b::/96"), // NAT64 can map to internal IPv4
}

func isPublicIP(ip netip.Addr) bool {
	ip = ip.Unmap() // ::ffff:127.0.0.1 must be treated as 127.0.0.1
	if !ip.IsGlobalUnicast() || ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() {
		return false
	}
	for _, p := range blockedPrefixes {
		if p.Contains(ip) {
			return false
		}
	}
	return true
}

func ssrfControl(network, address string, _ syscall.RawConn) error {
	if network != "tcp4" && network != "tcp6" {
		return fmt.Errorf("ssrf: network %q not allowed", network)
	}
	ap, err := netip.ParseAddrPort(address)
	if err != nil {
		return fmt.Errorf("ssrf: %w", err)
	}
	if !isPublicIP(ap.Addr()) {
		return fmt.Errorf("ssrf: destination %s not allowed", ap.Addr())
	}
	if p := ap.Port(); p != 80 && p != 443 {
		return fmt.Errorf("ssrf: port %d not allowed", p)
	}
	return nil
}

func newExternalClient() *http.Client {
	dialer := &net.Dialer{Timeout: 5 * time.Second, Control: ssrfControl}
	transport := &http.Transport{
		Proxy:                 nil, // a proxy would hide the real destination from Control
		DialContext:           dialer.DialContext,
		TLSHandshakeTimeout:   5 * time.Second,
		ResponseHeaderTimeout: 10 * time.Second,
		MaxIdleConns:          50,
		IdleConnTimeout:       60 * time.Second,
	}
	return &http.Client{
		Transport: transport,
		Timeout:   30 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= 3 {
				return errors.New("too many redirects")
			}
			if req.URL.Scheme != "https" && req.URL.Scheme != "http" {
				return fmt.Errorf("redirect to scheme %q not allowed", req.URL.Scheme)
			}
			return nil
		},
	}
}

func fetchExternal(ctx context.Context, c *http.Client, raw string) ([]byte, error) {
	u, err := url.Parse(raw)
	if err != nil || (u.Scheme != "https" && u.Scheme != "http") || u.Host == "" {
		return nil, errors.New("invalid url")
	}
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, u.String(), nil)
	if err != nil {
		return nil, fmt.Errorf("build request: %w", err)
	}
	resp, err := c.Do(req)
	if err != nil {
		return nil, fmt.Errorf("fetch: %w", err)
	}
	defer resp.Body.Close()
	const maxResp = 5 << 20
	body, err := io.ReadAll(io.LimitReader(resp.Body, maxResp+1))
	if err != nil {
		return nil, fmt.Errorf("read body: %w", err)
	}
	if len(body) > maxResp {
		return nil, errors.New("response too large")
	}
	return body, nil
}
```

Use this client only for untrusted destinations; internal service-to-service clients need the opposite (private IPs allowed) and should be separate instances so the two policies never mix. Do not forward the user's credentials or internal auth headers to user-supplied URLs.
