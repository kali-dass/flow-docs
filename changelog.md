# Changelog

---

## v0.2.0-trial

### New features

#### Policies

- **[Response caching](policies/cache-response.md)** — in-memory HTTP response cache per route. Add a `cache-response { }` block to any route. Serves cache hits without contacting the backend; injects `X-Cache-Status: HIT/MISS/BYPASS` on every cached-route response. Configurable TTL, body size limit, total capacity, cacheable status codes and methods, extra key headers. Supports `Vary` header-based variant tracking (up to 64 variants per URL). Per-identity key isolation via `identity-from` — scope the cache namespace to a specific authenticated user (`"client-id"`), tenant (`"tenant-id"`), request header (`"header:<name>"`), or JWT claim (`"claim:<path>"`, including nested claims via JSON Pointer). Absent identity → `BYPASS` with a one-time warning rather than a silent data leak. RFC compliance: `Cache-Control: only-if-cached` returns `504` on a MISS without contacting the upstream (RFC7234 §5.4); upstream `Age` is forwarded correctly so hits reflect total origin age, not just Flow's receipt time (RFC7234 §5.1); `cache-content-types` allowlist restricts storage to specific content-type prefixes (default: allow all).

- **[Correlation ID](policies/correlation-id.md)** — inject or preserve a correlation ID on every request. Two generators: `uuid-counter` (unique UUID prefix per worker + monotonic counter suffix, default) and `uuid-v4` (fully random). `trust-incoming=true` (default) reuses an incoming ID, read from `x-request-id` by default, for end-to-end tracing; `trust-incoming=false` always generates a fresh ID. `echo-downstream=true` writes the ID back to the response. The ID is propagated to the upstream on every request regardless of these settings.

- **[Request termination](policies/request-termination.md)** — return a configured HTTP response immediately without calling the upstream. Supports status-only (bare 503), plain `message` (auto-wrapped as `{"message":"..."}`), raw `body` + `content-type` (HTML, XML, custom JSON), optional `trigger` header/query-param for conditional activation, and a `headers { }` block for extra response headers like `Retry-After`. Use cases: maintenance mode, deprecated endpoint sunset, emergency kill switch, static mock responses.

- **[IP restriction](policies/ip-restriction.md)** — allow or block requests by source IP address. Supports allow-only mode, deny-only mode, and combined mode (deny wins). Configurable rejection status code and message. Accepts IPv4 and IPv6 CIDR ranges.

- **[Rate limiting](policies/rate-limiting.md)** — unified `rate-limit` policy replacing three separate v0.1.0 policies. Configurable `key-from` dimension (`client-id`, `source-ip`, `uri`, `uri-group`, `header:<name>`, `composite`). Token-bucket algorithm: `max-tokens`, `refill-qty`, `refill-interval-ms`. Bucket capacity and worker threads are set with `max-buckets` and `threads`. Missing-key behavior is configurable per dimension. **JWT claim–based rate limiting** — `key-from="jwt-claim:<name>"` derives the bucket key from a named JWT claim (requires `token-auth` with `expose-claims=true` on the same route). **Subscription tiers** — a `tiers { }` block maps claim values to per-tier token-bucket parameters; `missing-tier "reject"` (default) or `"skip"` controls unknown tiers. **Per-client limit overrides** — `override-claims { claim … }` reads `max-tokens` and `refill-qty` from JWT claims, with operator-enforced ceilings; flat claim names and JSON Pointer nested paths (e.g. `/entitlements/rate/max_tokens`) are both supported.

- **[Token authentication](policies/token-auth.md)** — unified policy (`token-auth`) supporting
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
  See [Non-REST Protocols](non-rest-protocols.md).

#### Upstream connections

- **gRPC support: declared routes and gRPC-native errors.** gRPC is declared, not detected. On a connector, `proto="grpc"` (cleartext HTTP/2) and `proto="grpcs"` (HTTP/2 over TLS, with `tls-sni`) say which transport the backend uses; they are equivalent to the existing `proto="h2-only"` spellings, which remain valid. On a route, `allowed-protocols "grpc"` (cleartext clients), `"grpcs"` (TLS clients) or both declare a gRPC route; they imply `POST` and HTTP/2 (so `allowed-methods` can be omitted) and require every connector to be HTTP/2 only. Flow refuses a client on the wrong transport: `426` for a cleartext client on a `grpcs` route, `400` for TLS on a `grpc` route or for HTTP/1.1. When Flow itself answers a call on a gRPC route (no backend involved), it sends a gRPC response with a real `grpc-status` instead of a JSON HTTP error: `UNIMPLEMENTED` for no route, `UNAVAILABLE` for an unreachable backend, `RESOURCE_EXHAUSTED` for a rate limit, `UNAUTHENTICATED`, `PERMISSION_DENIED`, and `FAILED_PRECONDITION` for a client on the wrong transport. Backend responses are untouched and other routes keep their JSON errors. See [Non-REST Protocols](non-rest-protocols.md#grpc) and [errors Flow produces on a gRPC route](non-rest-protocols.md#errors-flow-produces-on-a-grpc-route).

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

- **`path-prefix "/"` now matches every path.** It previously matched only the exact path `/`, so a route written as a catch-all served nothing else. More specific routes still take precedence. If a config had a `"/"` route alongside other routes, requests that matched no route (and got a `502`) now go to the `"/"` route's backend. A `path-prefix` ending in `/` (such as `"/api/"`), which could never match, is now accepted with the trailing slash removed and a startup warning.

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

---

## v0.1.0-trial

### New features

#### Policies

- **[Token authentication](policies/token-auth.md)** (`jwt-validate`, now superseded by
  `token-auth` in v0.2.0): multi-issuer JWT verification against a live JWKS endpoint or a
  static pinned key. Algorithm-confusion defense, `exp`/`nbf`/`aud`/`iss`/required-claims
  enforcement, verification cache.

#### Routing and listeners

- **Downstream keepalive** — `keepalive-timeout-secs` bounds how long an idle client
  connection is held open.

#### Protocols

- **Downstream h2c support** — plain listeners can now serve cleartext HTTP/2 (`offer-h2`),
  enabling h2c clients (browsers, gRPC, HTTP/2 tooling) without TLS. One plain listener serves
  both h2c and HTTP/1.1 clients. Plain listeners in a service must all agree on `offer-h2`.

- **h2c upstream support** — connectors can send cleartext HTTP/2 to backends
  (`allow-h2c = true`) without upstream TLS, retaining HTTP/2 multiplexing on trusted internal
  networks. Requires `proto "h2-only"` and no `tls-sni`.

- **Strict `h2-only` enforcement** — `proto "h2-only"` connectors now return `502` if the
  upstream negotiates HTTP/1.1 instead of HTTP/2, rather than silently falling back. Use
  `proto "h2-or-h1"` when HTTP/1.1 is acceptable.

#### Upstream connections

- **Connector timeout controls** — `idle-timeout-secs`, `read-timeout-secs`,
  `write-timeout-secs`, `h2-ping-interval-secs` on upstream connectors.

#### Observability

- **Configurable application logging** — log level is now set in config
  (`system.logging.app-log.level`: `error`/`warn`/`info`/`debug`/`trace`, default `info`) or
  via `--log-level`. Logs can be written to a file (`output "file"` + `log-dir`) instead of
  stdout — useful when running as a background process. `RUST_LOG` is ignored; if set, a
  one-line warning is emitted.

- **[Timing headers](policies/timing-header.md)** (`timing-header`) — injects
  `X-Flow-Duration-Us` and `X-Upstream-Duration-Us` (microseconds) into responses for
  per-request latency visibility.

#### Operations

- **Startup license gate** — Flow refuses to start without a valid signed license file.
  Resolved via `--license-file`, `FLOW_LICENSE_FILE`, or `./license.json`. Verification is
  fully offline (no license server or network call at startup).

### Config changes / migration

#### Routing and listeners

- **A service combining an `offer-h2` plain listener with any TLS listener is rejected at
  config load.** Split into two services. See [Configuration Reference](configuration.md).

- **`route-cache-dynamic-capacity` moved** from `system` scope to per-service scope.

#### Protocols

- **`max-h2-streams` defaults to 100** on H2-capable connectors (overrides the previous
  default of 1, which silently disabled HTTP/2 multiplexing). Set explicitly to override.

#### Observability

- **`measure-timing` renamed to `timing-header`** — no alias; update config before upgrading.
  Headers changed from milliseconds to microseconds (`X-Flow-Duration-Us` /
  `X-Upstream-Duration-Us`).

#### Configuration

- **Duplicate config keys are now rejected** — a duplicate block, scalar, or attribute
  (previously silently resolved with inconsistent winner rules) now errors at config load,
  naming the second occurrence.

- **Unknown connector and listener attributes are now rejected** — a typo'd or misplaced
  attribute errors at load instead of being silently ignored.
