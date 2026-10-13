# Changelog

---

## v0.2.0-trial

### New features

- **[Circuit breaking](configuration.md#circuit-breaking)** — passive per-backend failure detection on real request outcomes. Add `circuit-breaker { … }` inside any `load-balance` block. Three-state machine (`Closed` → `Open` → `HalfOpen`) with configurable error rate threshold, time window, cooldown period, and probe count. Two window strategies: `"SlidingWindow"` (exact, default) and `"TimeBuckets"` (fixed-memory, for high-traffic routes). Fail-open when all backends are tripped. Compatible with `health-check "Active"`.

- **[Active upstream health checking](configuration.md#health-checking)** — periodic HTTP probes per backend. Enable with `health-check "Active"` in any `load-balance` block. Configurable probe path, interval, timeout, and thresholds for marking unhealthy and recovering. Fail-open: when all backends are unhealthy, requests route anyway rather than failing.

- **[IP restriction](policies/ip-restriction.md)** — allow or block requests by source IP address. Supports allow-only mode, deny-only mode, and combined mode (deny wins). Configurable rejection status code and message. Accepts IPv4 and IPv6 CIDR ranges.

- **Real client IP resolution** — `real-ip-source` and `trusted-proxies` fields under `system`. All IP-based policies now resolve the effective client IP from `X-Forwarded-For` (or a named header) when set, rather than always using the raw TCP peer address. A trusted-proxy list is required when using XFF — Flow refuses to start without one, preventing IP spoofing. See [Configuration Reference](configuration.md#real-ip-source).

- **`X-Forwarded-For` outbound** — Flow now writes `X-Forwarded-For`, `X-Forwarded-Proto`, and `X-Forwarded-Host` on every upstream request. Incoming headers are stripped before injection when the TCP peer is not in `trusted-proxies`, preventing header chain spoofing.

- **[DNS upstream discovery](configuration.md#dns-discovery)** — connector addresses can now be hostnames (`backend.internal:3000`) in addition to literal IP:port. Flow re-resolves them on a configurable interval (`dns-interval-secs`) so backends can be updated without a config change or restart. Requires a `system { dns-resolver { … } }` block or defaults to the system resolver.

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

### Fixes

- **Wrong-phase filter error now names the correct phase.** Placing a policy in a block it does not support (e.g. `token-auth` in `upstream-response`) now produces `"filter 'token-auth' is not valid in upstream-response — place it in request-filters"` instead of the misleading `"Unknown upstream response filter: 'token-auth'"`.

### Config changes / migration

- **`jwt-validate` is replaced by `token-auth`** — the `jwt-validate` policy kind is removed.
  Existing configs must change `filter kind="jwt-validate"` to `filter kind="token-auth"` and
  add `mode "jwt"`. See [Token Authentication](policies/token-auth.md) for the full reference
  and migration example.

- **`block-cidr-range` renamed to `ip-restriction`** — update `kind="block-cidr-range"` to `kind="ip-restriction"` in all configs. No behaviour change.

- **`rate-limit` replaces three separate policies** — `rate-limit-single-uri-group`, `rate-limit-multi-source-ip`, and `rate-limit-multi-uri` are removed. Use `kind="rate-limit"` with the appropriate `key-from` value (`uri-group`, `source-ip`, or `uri`). See [Rate limiting](policies/rate-limiting.md).

---

## v0.1.0-trial

### New features

- **[Token authentication](policies/token-auth.md)** (`jwt-validate`, now superseded by
  `token-auth` in v0.2.0): multi-issuer JWT verification against a live JWKS endpoint or a
  static pinned key. Algorithm-confusion defense, `exp`/`nbf`/`aud`/`iss`/required-claims
  enforcement, verification cache.

- **Configurable application logging** — log level is now set in config
  (`system.logging.app-log.level`: `error`/`warn`/`info`/`debug`/`trace`, default `info`) or
  via `--log-level`. Logs can be written to a file (`output "file"` + `log-dir`) instead of
  stdout — useful when running as a background process. `RUST_LOG` is ignored; if set, a
  one-line warning is emitted.

- **Startup license gate** — Flow refuses to start without a valid signed license file.
  Resolved via `--license-file`, `FLOW_LICENSE_FILE`, or `./license.json`. Verification is
  fully offline (no license server or network call at startup).

- **Downstream h2c support** — plain listeners can now serve cleartext HTTP/2 (`offer-h2`),
  enabling h2c clients (browsers, gRPC, HTTP/2 tooling) without TLS. One plain listener serves
  both h2c and HTTP/1.1 clients. Plain listeners in a service must all agree on `offer-h2`.

- **h2c upstream support** — connectors can send cleartext HTTP/2 to backends
  (`allow-h2c = true`) without upstream TLS, retaining HTTP/2 multiplexing on trusted internal
  networks. Requires `proto "h2-only"` and no `tls-sni`.

- **Strict `h2-only` enforcement** — `proto "h2-only"` connectors now return `502` if the
  upstream negotiates HTTP/1.1 instead of HTTP/2, rather than silently falling back. Use
  `proto "h2-or-h1"` when HTTP/1.1 is acceptable.

- **[Timing headers](policies/timing-header.md)** (`timing-header`) — injects
  `X-Flow-Duration-Us` and `X-Upstream-Duration-Us` (microseconds) into responses for
  per-request latency visibility.

- **Connector timeout controls** — `idle-timeout-secs`, `read-timeout-secs`,
  `write-timeout-secs`, `h2-ping-interval-secs` on upstream connectors.

- **Downstream keepalive** — `keepalive-timeout-secs` bounds how long an idle client
  connection is held open.

### Config changes / migration

- **A service combining an `offer-h2` plain listener with any TLS listener is rejected at
  config load.** Split into two services. See [Configuration Reference](configuration.md).

- **`max-h2-streams` defaults to 100** on H2-capable connectors (overrides the previous
  default of 1, which silently disabled HTTP/2 multiplexing). Set explicitly to override.

- **`route-cache-dynamic-capacity` moved** from `system` scope to per-service scope.

- **`measure-timing` renamed to `timing-header`** — no alias; update config before upgrading.
  Headers changed from milliseconds to microseconds (`X-Flow-Duration-Us` /
  `X-Upstream-Duration-Us`).

- **Duplicate config keys are now rejected** — a duplicate block, scalar, or attribute
  (previously silently resolved with inconsistent winner rules) now errors at config load,
  naming the second occurrence.

- **Unknown connector and listener attributes are now rejected** — a typo'd or misplaced
  attribute errors at load instead of being silently ignored.
