# How Flow Behaves

What to expect when you run Flow: how it responds when things go wrong, what it shares between instances, and how it handles bodies, connections, and long-lived streams. These are deliberate design choices, so knowing them helps you plan around them.

For what is available and what is planned, see the [Roadmap](roadmap.md). For running Flow (signals, upgrades, logging, sizing), see [Operations](operations.md).

## Upstream failures

If every backend in a route is unreachable or closes the connection, Flow returns **HTTP 502 Bad Gateway** to the client.

When a request arrives, Flow picks a backend (round-robin, random, or hashing), tries to connect and proxy the request, and returns 502 if the connection fails (refused, timed out, or closed).

**With active health checking** (`health-check "Active"`), Flow removes backends that fail probes before requests reach them. A backend is removed only after `unhealthy-threshold` consecutive probe failures, so a transient blip does not pull it out of rotation, and it is restored automatically when it recovers. See [Health checking](policies/health-checking.md) and [Configuration](configuration.md) for the syntax. Health checking catches backends that are completely down. For backends that are up but returning errors on real traffic, see circuit breaking in [Design Decisions](design-decisions.md#how-should-i-handle-backend-failures).

**Without health checking** (`health-check "None"`, the default), Flow has no view of backend health, and failures are discovered when a request fails.

**If every backend in a pool is marked unhealthy**, Flow routes requests anyway (fail-open). This keeps a misconfigured health probe from taking a service offline.

You will see 502s when all backends are down (maintenance, crash, a bad deployment), when a network partition isolates Flow from its backends, or when health checking is off and a backend fails silently. To handle them:

- Enable `health-check "Active"` so failed backends stop receiving traffic.
- Have clients retry on 502 with exponential backoff.
- Alert when Flow starts returning 502s.
- Before shutting a backend down for maintenance, remove it from DNS (for DNS-discovered connectors) or from Flow's configuration (for static addresses).

```
$ curl -v http://localhost:8080/api/
< HTTP/1.1 502 Bad Gateway
```

A backend configured `h2-only` that answers with HTTP/1.1 also produces a 502. See [Protocols](protocols.md#strict-http2-h2-only).

## Responses Flow generates

When Flow rejects a request itself, it answers with a short JSON body and `content-type: application/json`:

```json
{"error": "no_route_matched", "message": "No configured route matches this request"}
```

| Situation | Status | `error` | Also sent |
|---|---|---|---|
| No route matches the request | 502 | `no_route_matched` | |
| Method not allowed on the matched route | 405 | `method_not_allowed` | `Allow` header; an `allowed` field in the body |
| Non-standard HTTP method | 501 | `not_implemented` | |
| Route requires TLS and the request is plain | 426 | `upgrade_required` | `Upgrade: TLS/1.2, TLS/1.3` |
| Protocol not accepted on the route | 400 | `protocol_not_allowed` | |
| Rate limit exceeded | 429 | `rate_limit_exceeded` | `Retry-After` header |
| Missing or invalid token | 401 | `unauthorized` | |
| Rate-limit key missing from the request (`missing-key="reject"`) | the configured status, 400 by default | `bad_request` (`unauthorized` for 401, `forbidden` for 403) | |
| Authentication service unreachable | 503 | `service_unavailable` | |

Policies that return a configured status, such as [IP restriction](policies/ip-restriction.md), [Request termination](policies/request-termination.md) and [Request size limit](policies/request-size-limit.md), send the status and message you set. Responses that come from your backends pass through unchanged.

These are plain HTTP errors, with JSON bodies. On a route declared `grpc` or `grpcs`, and for a gRPC call that matches no route, Flow answers in gRPC instead, with a `grpc-status` the client can read. See [Errors Flow produces on a gRPC route](non-rest-protocols.md#errors-flow-produces-on-a-grpc-route).

## Rate limiting

### Single instance

Rate-limit buckets are in-memory counters held by each Flow instance. There is no shared state and no inter-process communication, which keeps the check fast.

### Multiple instances

Each instance enforces its limit independently, so the limit applies per instance and not across your fleet. With a limit of 100 requests per second on a route and two instances behind a load balancer, up to about 200 requests per second get through.

You will notice this if you expect strict fleet-wide limits while running several instances. To handle it:

- Run a single instance with multiple threads and cores, if throughput allows.
- Use an external rate limiter (for example Redis-based) for strict global limits.
- Accept per-instance limits and size them knowing the total multiplies by the number of instances.

> **Planned:** rate limiting shared across Flow instances. See [Roadmap](roadmap.md#traffic-control).

For the policy options, see [Rate limiting](policies/rate-limiting.md).

## Connection reuse

Flow keeps a connection pool to each upstream backend and reuses connections across requests.

- **HTTP/1.1:** connections use keep-alive. If the backend closes one, Flow opens a new connection on the next request.
- **HTTP/2:** many requests are multiplexed over one connection, which Flow keeps for later requests.

Connections close and reopen when the backend closes an idle connection, the network is interrupted, the backend runs out of file descriptors, or the backend process is killed.

You may observe an occasional 502 while a new connection opens, a latency bump on the first request after a connection cycle, and memory that grows with the number of concurrent connections. Flow sets no limit on concurrent connections per backend, so the file-descriptor limit of the host is the ceiling.

Routes that point to the same backend share its connections. With HTTP/2, several routes can multiplex over a single upstream connection, and each route keeps its own `read-timeout-secs` and `write-timeout-secs`: a timeout set on one route does not apply to the other routes on that connection, whichever route opened it. This lets a gRPC service give its unary methods a read timeout and leave its streaming methods without one, as in [gRPC streaming](non-rest-protocols.md#grpc-streaming).

The connector attributes `idle-timeout-secs`, `max-h2-streams`, and the read, write, and ping settings tune this. See [Configuration](configuration.md). The `flow_upstream_connect_duration_seconds` metric, labelled `reused`, shows how often connections are reused. See [Metrics](metrics.md).

## HTTP/2 stream concurrency

HTTP/2 multiplexes requests over one connection, and Flow caps concurrent streams per upstream connection at **100** by default. You do not need to set it. Requests beyond the cap wait on that connection until earlier ones finish. The limit is per connection, not global, and a new connection gets its own 100 streams, so this is rarely a constraint.

The effective limit is the smaller of Flow's setting and the backend's advertised `SETTINGS_MAX_CONCURRENT_STREAMS`. If request latency rises under high concurrency, raise `max-h2-streams` on the connector:

```kdl
"backend.example.com:443" max-h2-streams=200
```

## Header size

Flow accepts up to about 1 MB of headers per request, cookies included. Most APIs come nowhere near this. You could reach it with very large cookies or hundreds of custom headers, in which case Flow rejects the request and logs an error. The limit is not configurable; trim or compress header data before it reaches Flow.

## Request buffering

Flow streams request and response bodies and does not hold whole messages in memory. Large uploads and downloads do not use proportional memory, and streaming starts immediately.

- If a client sends slowly, Flow waits; it does not read ahead.
- If a backend closes the connection mid-stream, the client gets a partial response or a timeout.
- Because bodies are not buffered, no policy inspects them. Policies work on the request line, headers, and the identity of the caller.
- Flow places no size limit on response bodies, and none on request bodies unless you add [Request size limit](policies/request-size-limit.md) to a route.

> **Planned:** response size limits. See [Roadmap](roadmap.md#traffic-control).

## Request transformation

Flow proxies requests as they arrive, after policies have run. Policies rewrite **headers** (add, replace, remove), and routing uses paths, headers, and TLS SNI. Request and response bodies, query parameters, and URIs pass through unchanged. Responses can be cached with the [response cache](policies/cache-response.md), and bodies are not compressed or decompressed.

To change bodies or URIs, do it in the backend or in a service in front of Flow.

> **Planned:** URI and path rewriting, and request and response body transformation. See [Roadmap](roadmap.md#transformation-and-caching).

## WebSocket

Flow passes WebSocket upgrades through. The HTTP/1.1 upgrade handshake (`GET` with `Connection: Upgrade` and `Upgrade: websocket`) passes through, and after the `101 Switching Protocols` response the connection is a raw two-way byte stream between the client and the backend.

- Routes, policies (rate limiting, header rewriting, IP restriction, and so on), and authentication apply to the handshake request. They do not run on individual WebSocket frames.
- Flow does not negotiate subprotocols, inject ping/pong, or inspect frames.
- Set `downstream-read-timeout "None"` on a WebSocket route. Without it, an idle connection is closed after the default 60 seconds. See [Non-REST Protocols](non-rest-protocols.md#websocket) for a route example.
- With metrics enabled, WebSocket connections are recorded in the stream metrics (`flow_streams_total`, `flow_active_streams`, and the stream duration and byte metrics), not in the request duration histogram. See [Metrics](metrics.md#streams-sse-and-websocket).

> **Preview:** WebSocket passthrough. See [Roadmap](roadmap.md#protocols).

## License check

Flow verifies its license file once, at startup, before opening any listener. If the license is missing, invalid, or expired past its grace period, Flow refuses to start and says which check failed. It never starts in a degraded state.

A license that expires while Flow is running is not noticed until the next restart, and Flow keeps serving traffic normally in the meantime. See [Licensing](license.md) for where Flow looks for the file, the grace period, and common problems.

> **Planned:** a continuous license check while running. See [Roadmap](roadmap.md#operations).

## Configuration changes

Flow reads its configuration file once, at startup. Editing the file does not change a running instance, and nothing watches it. The listener's TLS certificate and key are read at startup too, so when a certificate is renewed Flow keeps serving the old one until the process is replaced. After the old certificate expires, clients see handshake failures.

To apply a change, validate it first, then replace the process:

```bash
flow --config-kdl /etc/flow/config.kdl --validate-configs
```

- **Restart.** The simplest option, and there is a gap while the new process starts.
- **Zero-downtime upgrade.** Start the new process with `--upgrade` and signal the old one, as described in [Operations](operations.md#zero-downtime-upgrades). The new process re-reads the configuration and the certificates. This works on Linux only.
- **In containers.** Roll the container instead: start a new one, move traffic to it, and retire the old one.

Backend addresses that are DNS names are the exception: Flow re-resolves them in the background and updates the backend pool without a restart. See [DNS discovery](dns.md).

> **Planned:** reloading configuration on a signal, and rotating TLS certificates, without replacing the process. See [Roadmap](roadmap.md#operations).
