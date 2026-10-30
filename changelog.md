# Changelog

---

## v0.2.0-trial

### New features

#### Policies

- **[Response caching](policies/cache-response.md)** — in-memory HTTP response cache per route. Add a `cache-response { }` block to any route. Serves cache hits without contacting the backend; injects `X-Cache-Status: HIT/MISS/BYPASS` on every cached-route response. Configurable TTL, body size limit, total capacity, cacheable status codes and methods, extra key headers. Supports `Vary` header-based variant tracking (up to 64 variants per URL). Per-identity key isolation via `identity-from` — scope the cache namespace to a specific authenticated user (`"client-id"`), tenant (`"tenant-id"`), request header (`"header:<name>"`), or JWT claim (`"claim:<path>"`, including nested claims via JSON Pointer). Absent identity → `BYPASS` with a one-time warning rather than a silent data leak. RFC compliance: `Cache-Control: only-if-cached` returns `504` on a MISS without contacting the upstream (RFC7234 §5.4); upstream `Age` is forwarded correctly so hits reflect total origin age, not just Flow's receipt time (RFC7234 §5.1); `cache-content-types` allowlist restricts storage to specific content-type prefixes (default: allow all).

- **[Request size limit](policies/request-size-limit.md)** — reject requests whose body is over `max-body-size` (a number of bytes, or `KB`/`MB`/`GB`) with `413`, before the backend is contacted when the request declares its length, and as the body arrives for chunked and HTTP/2 uploads. Not accepted on gRPC routes, and not applied to an upgraded WebSocket connection.

- **[Correlation ID](policies/correlation-id.md)** — inject or preserve a correlation ID on every request. Two generators: `uuid-counter` (unique UUID prefix per worker + monotonic counter suffix, default) and `uuid-v4` (fully random). `trust-incoming=true` (default) reuses an incoming ID, read from `x-request-id` by default, for end-to-end tracing; `trust-incoming=false` always generates a fresh ID. `echo-downstream=true` writes the ID back to the response. The ID is propagated to the upstream on every request regardless of these settings.

- **[Request termination](policies/request-termination.md)** — return a configured HTTP response immediately without calling the upstream. Supports status-only (bare 503), plain `message` (auto-wrapped as `{"message":"..."}`), raw `body` + `content-type` (HTML, XML, custom JSON), optional `trigger` header/query-param for conditional activation, and a `headers { }` block for extra response headers like `Retry-After`. Use cases: maintenance mode, deprecated endpoint sunset, emergency kill switch, static mock responses.

- **[IP restriction](policies/ip-restriction.md)** (replaces `block-cidr-range`) — allow or block requests by source IP address. Supports allow-only mode, deny-only mode, and combined mode (deny wins). Configurable rejection status code and message. Accepts IPv4 and IPv6 CIDR ranges.

- **[Rate limiting](policies/rate-limiting.md)** — unified `rate-limit` policy replacing three separate v0.1.0 policies. Configurable `key-from` dimension (`client-id`, `source-ip`, `uri`, `uri-group`, `header:<name>`, `composite`). Token-bucket algorithm: `max-tokens`, `refill-qty`, `refill-interval-ms`. Bucket capacity and worker threads are set with `max-buckets` and `threads`. Missing-key behavior is configurable per dimension. **JWT claim–based rate limiting** — `key-from="jwt-claim:<name>"` derives the bucket key from a named JWT claim (requires `token-auth` with `expose-claims=true` on the same route). **Subscription tiers** — a `tiers { }` block maps claim values to per-tier token-bucket parameters; `missing-tier "reject"` (default) or `"skip"` controls unknown tiers. **Per-client limit overrides** — `override-claims { claim … }` reads `max-tokens` and `refill-qty` from JWT claims, with operator-enforced ceilings; flat claim names and JSON Pointer nested paths (e.g. `/entitlements/rate/max_tokens`) are both supported.

- **[Token authentication](policies/token-auth.md)** (replaces `jwt-validate`) — unified policy (`token-auth`) supporting
  two modes on the same route:
  - **JWT mode** (`mode "jwt"`): verifies signed tokens against a JWKS endpoint or a
    locally-pinned static key. Multi-issuer, algorithm-pinned per issuer (never read from the
    token), audience validation, required-claims enforcement. Verification results are cached;
    `jwks-required-at-startup` (default `true`) refuses to start if the initial key fetch fails.
  - **Introspection mode** (`mode "introspection"`): validates opaque tokens via an RFC 7662
    introspection endpoint using HTTP Basic authentication. Results are cached by configurable
    TTL.
  - Both modes support identity normalization: `client-id-claim`, `tenant-id-claim`,
    `roles-claim`, `scopes-claim`.
  - `strip-token` (default `true`) removes the token header before forwarding to the upstream,
    so credentials are never sent to backends.
  - `realm` controls the `WWW-Authenticate` challenge returned on `401`.

#### Routing and listeners

- **Per-route downstream timeouts** — two new per-route fields control downstream H1 connection behavior (both no-ops on HTTP/2):
  - `downstream-read-timeout` — overrides the 60-second default downstream HTTP/1.1 read timeout. `"None"` disables it entirely (needed for WebSocket routes); a positive integer sets an explicit cap.
  - `downstream-write-timeout` — caps how long Flow waits for a slow HTTP/1.1 client to accept a write. Flow applies no write timeout by default; set a positive integer to reclaim connections from stalled clients.
  See [Timeouts at a glance](configuration.md#timeouts-at-a-glance) for how these relate to the upstream timeouts, and [Non-REST Protocols](non-rest-protocols.md).

#### Protocols

- **gRPC support: declared routes and gRPC-native errors.** gRPC is declared, not detected, on both sides of Flow. See [Non-REST Protocols](non-rest-protocols.md#grpc).
  - **Connector:** `proto="grpc"` (cleartext HTTP/2) and `proto="grpcs"` (HTTP/2 over TLS, with `tls-sni`) say which transport the backend uses. They are equivalent to the existing `proto="h2-only"` spellings, which remain valid.
  - **Route:** `allowed-protocols "grpc"` (cleartext clients), `"grpcs"` (TLS clients) or both declare a gRPC route. They imply `POST` and HTTP/2, so `allowed-methods` can be omitted, and they require every connector to be HTTP/2 only.
  - **Wrong transport:** Flow refuses a client on the wrong transport: `426` for a cleartext client on a `grpcs` route, and `400` for TLS on a `grpc` route or for HTTP/1.1.
  - **Errors:** when Flow itself answers a call on a gRPC route (no backend involved), it sends a gRPC response with a real `grpc-status` instead of a JSON HTTP error: `UNIMPLEMENTED` for no route, `UNAVAILABLE` for an unreachable backend, `RESOURCE_EXHAUSTED` for a rate limit, `UNAUTHENTICATED`, `PERMISSION_DENIED`, and `FAILED_PRECONDITION` for a client on the wrong transport. Backend responses are untouched and other routes keep their JSON errors. See [errors Flow produces on a gRPC route](non-rest-protocols.md#errors-flow-produces-on-a-grpc-route).

#### Upstream connections

- **[Circuit breaking](configuration.md#circuit-breaking)** — passive per-backend failure detection on real request outcomes. Add `circuit-breaker { … }` inside any `load-balance` block. Three-state machine (`Closed` → `Open` → `HalfOpen`) with configurable error rate threshold, time window, cooldown period, and probe count. Two window strategies: `"SlidingWindow"` (exact, default) and `"TimeBuckets"` (fixed-memory, for high-traffic routes). Fail-open when all backends are tripped. Compatible with `health-check "Active"`.

- **[Active upstream health checking](configuration.md#health-checking)** — periodic HTTP probes per backend. Enable with `health-check "Active"` in any `load-balance` block. Configurable probe path, interval, timeout, and thresholds for marking unhealthy and recovering. Fail-open: when all backends are unhealthy, requests route anyway rather than failing.

- **[DNS upstream discovery](dns.md)** — connector addresses can now be hostnames (`backend.internal:3000`) in addition to literal IP:port. Flow re-resolves them on a configurable interval (`dns-interval-secs`) so backends can be updated without a config change or restart. The `system { dns-resolver { … } }` block is optional; without it Flow uses the system resolver. `system.dns-resolver` also accepts `ip-version` (`"ipv4"`/`"ipv6"`/`"dual"`) and `empty-answer-threshold`, and `search-domain`/`ndots`/`timeout-secs`/`attempts` apply with or without custom nameservers. Changes to a hostname's address set are logged. See [DNS Resolution](dns.md).

#### Client IP and forwarded headers

- **Real client IP resolution** — `real-ip-source` and `trusted-proxies` fields under `system`. All IP-based policies now resolve the effective client IP from `X-Forwarded-For` (or a named header) when set, rather than always using the raw TCP peer address. A trusted-proxy list is required when using XFF — Flow refuses to start without one, preventing IP spoofing. See [Configuration Reference](configuration.md#system-block).

- **`X-Forwarded-For` outbound** — Flow now writes `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Host` on every upstream request. Incoming headers are stripped before injection when the TCP peer is not in `trusted-proxies`, preventing header chain spoofing.

#### Observability

- **[Prometheus metrics](metrics.md) (preview)** — add `system { metrics { listen "127.0.0.1:9090" } }` to expose `GET /metrics` on its own listener. Per-route request counts (`flow_requests_total`, with a `source` label that separates backend responses from responses Flow produced itself), latency histograms for the whole request, response headers, client transfer, backend round trip, backend connection setup, and the remaining overhead (Flow's own work for a normal request), an in-flight gauge, and a separate family for SSE and WebSocket streams (active streams, lifetime, bytes, how each ended). Optional `groups { dns true }` (per-hostname DNS resolution state) and `groups { bandwidth true }` (bytes in and out per route); `enabled false` turns the endpoint off while keeping the block. The endpoint has no authentication or TLS yet. See [Metrics](metrics.md).

### Fixes

#### Policies

- **Wrong-phase filter error now names the correct phase.** Placing a policy in a block it does not support (e.g. `token-auth` in `upstream-response`) now produces `"filter 'token-auth' is not valid in upstream-response — place it in request-filters"` instead of the misleading `"Unknown upstream response filter: 'token-auth'"`.

#### Routing and listeners

- **Error responses Flow generates carry `Server: Flow`.** Upstream-connect failures, upstream timeouts and other errors produced by Flow itself previously reported a different server name in the `Server` header. Update any check that matches on the old value.

- **`path-prefix "/"` now matches every path.** It previously matched only the exact path `/`, so a route written as a catch-all served nothing else. More specific routes still take precedence. If a config had a `"/"` route alongside other routes, requests that matched no route (and got a `502`) now go to the `"/"` route's backend. A `path-prefix` ending in `/` (such as `"/api/"`), which could never match, is now accepted with the trailing slash removed and a startup warning.

#### Client IP and forwarded headers

- **`X-Forwarded-Proto` is now correct for cleartext HTTP/2.** It was derived from the HTTP version, so clients using cleartext HTTP/2 (h2c) were reported as `https`. It now follows the client's transport: `http` for a cleartext connection, `https` for TLS.

#### Observability

- **`X-Flow-Duration-Us` no longer includes the time to connect to the backend.** Opening a new TCP/TLS connection to the backend used to be counted as Flow overhead, so the first request on a new connection could show a Flow time hundreds of times too high. The header no longer counts connection setup (it is reported separately as `flow_upstream_connect_duration_seconds`); values on requests that open a new backend connection drop, and requests on reused connections are unchanged. See [Timing headers](policies/timing-header.md).

### Config changes / migration

#### Policies

- **`jwt-validate` is replaced by `token-auth`** — the `jwt-validate` policy kind is removed.
  Existing configs must change `filter kind="jwt-validate"` to `filter kind="token-auth"` and
  add `mode "jwt"`. See [Token Authentication](policies/token-auth.md) for the full reference
  and migration example.

- **`block-cidr-range` renamed to `ip-restriction`** — update `kind="block-cidr-range"` to `kind="ip-restriction"` in all configs. No behavior change.

- **`rate-limit` replaces three separate policies** — `rate-limit-single-uri-group`, `rate-limit-multi-source-ip`, and `rate-limit-multi-uri` are removed. Use `kind="rate-limit"` with the appropriate `key-from` value (`uri-group`, `source-ip`, or `uri`). See [Rate limiting](policies/rate-limiting.md).

#### Routing and listeners

- **Listener addresses must now be unique, and Flow refuses to start if one cannot be bound.** Two listeners that share an address (same port with the same IP, or either one a wildcard) are rejected when the config loads, naming both services. Previously both loaded, and the second service silently failed to start after about 30 seconds while Flow kept running. At startup Flow now also tries to bind every listener and exits with a list of failures (port in use, address not present on the host, missing privileges); this check is skipped for `--validate-configs` and `--upgrade`. See [Configuration Reference](configuration.md#listener-addresses-must-be-unique-and-bindable).

- **Every route now requires a `name`, and service names must be unique.** Add `name "..."` as the first line inside each `route { }`. A route name must be unique within its service (the same name may appear in different services), use lowercase letters, digits, `_`, `.` and `-`, start with a letter or digit, and be at most 63 characters; `unmatched` is reserved. Two services with the same name are now rejected as well (previously both were loaded under the same name). Flow refuses to start on a missing, invalid or duplicate name and reports the line. See [Configuration Reference](configuration.md#name-required).

#### Client IP and forwarded headers

- **`X-Forwarded-Proto` changes for cleartext HTTP/2 clients.** Backends now see `http` for h2c clients, where they previously saw `https`. If a backend redirects or builds URLs from this header, check it. See [Fixes](#fixes).

---

## v0.1.0-trial

### New features

#### Routing and listeners

- **Route matching** on path prefix, path regex, header, SNI and SNI suffix, plus `all` (AND) and `any` (OR) combinators. The best match wins.
- **Per-route limits** on allowed methods and allowed protocols.
- **Plain and TLS listeners**, with `offer-h2` to offer HTTP/2 to clients.
- **Downstream keepalive** — `keepalive-timeout-secs` bounds how long an idle client connection is held open.
- **Route cache sizing** — `route-cache-dynamic-capacity`, set per service.

#### Load balancing and upstreams

- **Selection** — `RoundRobin`, `Random`, `FNV` hash and `Ketama` consistent hashing.
- **Static upstream addresses.**
- **Connector options** — TLS (`tls-sni`, `verify-cert`, `ca-cert-path`), `idle-timeout-secs`, `read-timeout-secs`, `write-timeout-secs`, `h2-ping-interval-secs` and `max-h2-streams`.

#### Protocols

- **Supported protocols.** HTTP/1.1, HTTP/2 over TLS (negotiated with ALPN) and cleartext HTTP/2 (h2c), each on both the client-facing and upstream side, with TLS optional on either hop. See [Protocols](protocols.md).
- **Cleartext HTTP/2** from clients (`offer-h2` on a plain listener, which also serves HTTP/1.1 clients) and to backends (`allow-h2c=true`, which requires `proto="h2-only"` and no `tls-sni`).
- **Strict `h2-only`.** A connector set to `proto="h2-only"` returns `502` if the backend negotiates HTTP/1.1. Use `proto="h2-or-h1"` when HTTP/1.1 is acceptable.
- **HTTP/2 multiplexing by default.** Connectors that speak HTTP/2 allow up to 100 concurrent streams per upstream connection (`max-h2-streams`), so several requests share one connection without any setting. Set it explicitly to override.

#### Policies

- **[Token authentication](policies/token-auth.md)** (`jwt-validate`) — multi-issuer JWT verification against a live JWKS endpoint or a static pinned key, with `exp`, `nbf`, `aud`, `iss` and required-claims checks and a verification cache.
- **IP restriction** (`block-cidr-range`) — allow or deny clients by CIDR range.
- **Rate limiting** — `rate-limit-single-uri-group`, `rate-limit-multi-source-ip` and `rate-limit-multi-uri`.
- **Header policies** — `upsert-header` and `remove-header-key-regex`, on requests and responses.
- **[Timing headers](policies/timing-header.md)** (`timing-header`) — adds latency headers to each response so you can see how long Flow took and how long the backend took for the call: `X-Flow-Duration-Us` and `X-Upstream-Duration-Us`, in microseconds.

#### Observability

- **Configurable application log** — set the level (`system.logging.app-log.level`: `error`, `warn`, `info`, `debug` or `trace`, default `info`, or `--log-level`) and write to stdout or to a file (`output "file"` with `log-dir`).

#### Operations

- **Startup license gate** — Flow refuses to start without a valid signed license file, resolved from `--license-file`, `FLOW_LICENSE_FILE` or `./license.json`. Verification is fully offline.
- **Zero-downtime upgrade and daemon mode** (Linux) — `--upgrade` hands over to a new binary without dropping connections, and `daemonize` runs Flow in the background, with `pid-file` and `upgrade-socket`.

#### Configuration

- **Strict validation.** A duplicate block, scalar or attribute is rejected at config load, naming the second occurrence, and an unknown or misplaced connector or listener attribute is an error instead of being silently ignored.
