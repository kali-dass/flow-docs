# Metrics

> **New here?** Start with [Getting Started](getting-started.md). The full `system` reference is in the [Configuration Reference](configuration.md#system-block).

Flow can expose Prometheus metrics on a separate endpoint: request counts, latency broken down into where the time went, in-flight requests, long-lived streams (SSE and WebSocket), and optional per-hostname DNS and per-route bandwidth series. This page covers how to turn it on, what is exposed, and how to keep the number of series bounded.

> **Preview.** The metric names and labels below are stable enough to build dashboards on, but the endpoint has no authentication or TLS yet. See [Securing the endpoint](#securing-the-endpoint).

---

## Turning it on

Metrics are **off** unless `system` has a `metrics` block with a `listen` address:

```kdl
system {
    metrics {
        listen "127.0.0.1:9090"
        // enabled true                 // default; false keeps the block but turns everything off
        // groups {                     // optional families, all off by default
        //     dns true
        //     bandwidth true
        // }
    }
}
```

| Setting | Default | Meaning |
|---|---|---|
| `listen` | *(required)* | `IP:port` the metrics endpoint binds to. No hostnames, port `0` is rejected. Must not be the same address as any service listener. |
| `enabled` | `true` | `false` keeps the block in the file (it is still validated) but Flow starts no metrics listener, registers no metrics, and records nothing. Useful for switching metrics off per environment without deleting the block. |
| `groups` | all off | Optional metric families. Each group is a boolean: `dns`, `bandwidth`. Unknown group names are rejected. See [Optional groups](#optional-groups). |

### Rules

- **File only.** There is no command-line flag or environment variable for metrics; the `metrics` block is the only way to configure it. It lives under `system`, appears once, and each key inside it appears once.
- **One endpoint for the whole process.** Metrics for every service and route are served from the one `listen` address and told apart by the `service` and `route` labels. When metrics are enabled, every route is recorded; there is no per-route or per-service switch (switches for individual policies are planned).
- **`listen` must be an IP address and port,** IPv4 or IPv6 (`"127.0.0.1:9090"`, `"[::1]:9090"`, `"0.0.0.0:9090"`). Hostnames, Unix sockets and port `0` are rejected, and so are TLS settings for now.
- **`listen` must not clash with any service listener.** The same rule as for service listeners applies: two TCP addresses clash when they have the same port and the same IP, or when either one is a wildcard (`0.0.0.0` or `::`) on that port. So `127.0.0.1:9090` and `10.0.0.1:9090` can coexist, but `0.0.0.0:9090` clashes with a service listening on `127.0.0.1:9090`. The check also applies when `enabled` is `false`.
- **`groups` takes only the names `dns` and `bandwidth`,** each set to `true` or `false`. An empty `groups { }` block is allowed and turns nothing on. An unknown group, a non-boolean value, or the same group or block twice is rejected.
- **Nothing is started or recorded when disabled.** With `enabled false` (or no block at all), Flow opens no extra listener and the request path records nothing.
- **The metrics listener uses its own thread,** separate from the `threads-per-service` workers.

Errors are reported when the config loads and name the offending line, for example:

```
metrics listen must be IP:port (for example "127.0.0.1:9090"), got 'localhost:9090'
metrics listen address 127.0.0.1:9090 conflicts with listener 127.0.0.1:9090 of service "api"; choose a different address or port
Unknown metrics group: 'cache' (supported: `dns`, `bandwidth`)
Unknown metrics setting: 'path' (supported: `listen`, `enabled`, `groups`)
```

Settings are read once at startup. Changing them requires a restart.

Flow checks at startup that the metrics address (and every service listener) can be bound, and refuses to start with one list of problems if any cannot. `--validate-configs` checks the shape of the `metrics` block without binding anything; add `--validate-configs-with-network` to also check that the addresses can be bound.

---

## Securing the endpoint

The metrics listener speaks **plain HTTP with no authentication**. Anyone who can reach the port can read route names, traffic volumes, error rates and backend hostnames.

- Bind it to loopback (`127.0.0.1`) or to an internal interface, and let your scraper reach it over a trusted network.
- If `listen` is not a loopback address, Flow logs a warning at startup.
- Use network controls (firewall, security group, Kubernetes `NetworkPolicy`) to restrict who can connect.

The listener serves `GET /metrics` only. Any other path returns `404` and any other method on `/metrics` returns `405` with `Allow: GET`. Responses are gzip-compressed when the scraper sends `Accept-Encoding: gzip`, which Prometheus does by default. The endpoint runs on its own listener and thread, so scrapes never compete with proxy traffic for worker threads.

Optional TLS and authentication for this listener are planned; see [Not yet available](#not-yet-available).

---

## During a zero-downtime upgrade

The metrics endpoint takes part in a [zero-downtime upgrade](operations.md#zero-downtime-upgrades) like any other listener; there is nothing to configure.

- **Same address:** if the new configuration keeps the same `listen` address, the new process takes over the metrics socket together with the proxy listeners, so the port keeps answering and there is no gap in your scrapes. The new process's startup bind check is skipped for the same reason.
- **Changed or removed:** if the new configuration changes `listen`, or removes the `metrics` block (or sets `enabled false`), the old socket is closed in the new process and the new address, if any, is bound normally.
- **Counters restart from zero.** The new process starts with fresh metrics. Prometheus treats this as a counter reset, so `rate()` and `increase()` handle it, but raw counter values drop.
- **Requests still draining in the old process are not scrapeable.** After the handover, scrapes reach the new process only, so requests that finish in the old process during its grace period are missing from the metrics. Gauges such as `flow_in_flight_requests` and `flow_active_streams` describe the new process only.

Zero-downtime upgrades are Linux only; see [Operations](operations.md#zero-downtime-upgrades).

---

## Scraping

```yaml
# prometheus.yml
scrape_configs:
  - job_name: flow
    static_configs:
      - targets: ["flow-host:9090"]
```

`/metrics` is Prometheus's default path, so no `metrics_path` is needed.

---

## What is exposed

All names start with `flow_`. Durations are in **seconds** (Flow measures in microseconds internally, so sub-millisecond values appear as small decimals such as `0.000180`), sizes are in bytes, and counters end in `_total`.

### Labels

| Label | Values |
|---|---|
| `service` | The service name from your config. |
| `route` | The route's `name`, or `unmatched` for requests that matched no route (see below). |
| `method` | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS`, `CONNECT`, `TRACE`, or `other`. |
| `status_class` | `1xx`, `2xx`, `3xx`, `4xx`, `5xx`, or `none` (no response was written, for example the client disconnected first). |
| `source` | Who produced the response the client received. `upstream`: a backend answered (any status, including a backend 5xx). `flow`: Flow answered without an upstream response: a policy rejection (rate limit, authentication, IP restriction), `405`/`501`, no matching route, [request termination](policies/request-termination.md), a cache hit, or a failed upstream connection. |

Every route has a required, unique-per-service [`name`](configuration.md#name-required), which is what the `route` label carries. Requests that match no route (and `405`/`501` responses, which are decided before a route is chosen) are counted under `route="unmatched"` for their service. `unmatched` is a reserved name and cannot be used for a route.

### Requests and latency (always on when metrics are enabled)

| Metric | Type | Labels | What it measures |
|---|---|---|---|
| `flow_requests_total` | counter | `service`, `route`, `status_class`, `method`, `source` | Finished requests. A series appears the first time that combination occurs. |
| `flow_in_flight_requests` | gauge | `service` | Requests being handled right now. Returns to 0 when idle, including after errors and client disconnects. |
| `flow_response_header_duration_seconds` | histogram | `service`, `route` | From request arrival until Flow had finished its response-side work and the response headers were ready to send. Upstream responses only. |
| `flow_request_duration_seconds` | histogram | `service`, `route` | The whole request: arrival until the client finished downloading the response. **Not recorded for [streams](#streams-sse-and-websocket).** |
| `flow_client_transfer_duration_seconds` | histogram | `service`, `route` | From headers ready until the body was sent to the client. Large for big downloads and slow clients. Upstream responses only. |
| `flow_upstream_duration_seconds` | histogram | `service`, `route` | The backend round trip as Flow sees it: from the start of the upstream-request phase (the `upstream-request` policies run first) until the response headers arrive. It includes sending the request, backend processing and the network, but not connection setup and not the response body. Connection setup is measured separately in `flow_upstream_connect_duration_seconds`, and it happens before this window opens. The response body is part of `flow_client_transfer_duration_seconds`: it is read from the backend and written to the client as one stream, and Flow does not time the two sides separately, so there is no backend-only body time. Recorded only when a backend answered with headers. |
| `flow_overhead_duration_seconds` | histogram | `service`, `route` | Whatever is left of the request after subtracting upstream, client transfer and upstream connection setup. For a normal proxied request that is Flow's own work, so it is the number to watch to see what Flow adds, but it is a remainder, not a direct measurement: see [Where the time goes](#where-the-time-goes-and-where-it-can-mislead). |
| `flow_upstream_connect_duration_seconds` | histogram | `service`, `route`, `reused` | Time to get a connection to the backend: a pool checkout when `reused="true"`, a TCP/TLS handshake when `reused="false"`. Connection setup is excluded from `flow_overhead_duration_seconds`, so a cold handshake never looks like Flow being slow. |

Duration histograms share these bucket upper bounds, in seconds: 0.00005, 0.0001, 0.00025, 0.0005, 0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10 (and `+Inf`). The low end is fine-grained because Flow's own overhead is usually well under a millisecond.

The overhead and connect numbers are derived from the same timestamps as the [`timing-header`](policies/timing-header.md) policy, so `X-Flow-Duration-Us` and these metrics are consistent.

### Streams (SSE and WebSocket)

Long-lived responses would make the request-duration histogram meaningless (a stream lasts minutes or hours), so Flow tracks them in their own family. A response is a stream when it is:

- `101 Switching Protocols` (`kind="websocket"`), or
- `Content-Type: text/event-stream` (`kind="sse"`), over HTTP/1.1 or HTTP/2. MCP over SSE falls in here.

Detection needs no configuration. gRPC is **not** treated as a stream, because most gRPC calls are short unary calls; gRPC traffic stays in the ordinary request metrics.

A stream still counts in `flow_requests_total` and the response-header, upstream, connect and overhead histograms; it is excluded only from `flow_request_duration_seconds`.

| Metric | Type | Labels | What it measures |
|---|---|---|---|
| `flow_active_streams` | gauge | `service`, `route`, `kind` | Streams open now. The main capacity-planning signal for streaming routes. |
| `flow_stream_duration_seconds` | histogram | `service`, `route`, `kind` | How long finished streams lasted. Buckets: 1 s, 5 s, 15 s, 30 s, 1 m, 2 m, 5 m, 10 m, 30 m, 1 h, 2 h, 4 h. |
| `flow_stream_bytes_total` | counter | `service`, `route`, `kind`, `direction` | Body bytes carried: `in` from the client, `out` to the client. |
| `flow_streams_total` | counter | `service`, `route`, `kind`, `end` | Finished streams by how they ended (below). |

`end` values: `normal` (clean finish), `client` (the client closed or failed), `upstream` (the backend closed or failed), `timeout` (a read or write timeout fired, such as `downstream-read-timeout`), `flow` (an internal error). A WebSocket the client closes cleanly is reported as `normal`; only abrupt closes and failures are attributed to `client` or `upstream`.

Stream series for a route appear when its first stream of that kind starts; routes that never stream export none.

### Optional groups

Turn each on under `metrics { groups { … } }`. Both are off by default and register nothing when off.

**`dns`**: per-hostname series for connectors that use [DNS discovery](dns.md). A connector with `discovery "Static"`, or one that is an IP literal, never refreshes and exports nothing.

| Metric | Type | Labels | What it measures |
|---|---|---|---|
| `flow_dns_resolved_addresses` | gauge | `hostname` | Addresses currently resolved for the name. `0` after the backend set was cleared (see [`empty-answer-threshold`](dns.md)). |
| `flow_dns_refresh_failures_total` | counter | `hostname`, `kind` | Refreshes that did not produce a fresh address set. `kind` is `empty_answer` (the answer held no addresses), `nxdomain` (NXDOMAIN or no records), or `transient` (timeout, SERVFAIL or I/O error; the last-known-good set is always kept). |
| `flow_dns_last_success_timestamp_seconds` | gauge | `hostname` | Unix time of the last successful refresh; `0` until the first success. `time() - flow_dns_last_success_timestamp_seconds` is how old the served address set is. |

A failed refresh leaves `flow_dns_resolved_addresses` unchanged, because Flow keeps serving the last-known-good addresses until the failure threshold is reached.

**`bandwidth`**:

| Metric | Type | Labels | What it measures |
|---|---|---|---|
| `flow_bandwidth_bytes_total` | counter | `service`, `route`, `direction` | Body bytes per route: `in` is the request body read from the client, `out` is the response body written to the client (after any compression). Counts every request, streams included. Headers are not counted. |

### Housekeeping

| Metric | Type | Labels | What it measures |
|---|---|---|---|
| `flow_build_info` | gauge | `version` | Always `1`; the label carries the Flow version. |
| `flow_metrics_errors_total` | counter | none | Requests whose metrics could not be attributed to a known route. Should stay `0`; anything else is a bug worth reporting. |

---

## Where the time goes, and where it can mislead

Every duration is measured from the moment Flow has received the request until it has finished handing the response to the network. A normal proxied request passes through these stages, and the metrics tile that span exactly:

```
 before Flow       |<-------------------------- Flow's window ---------------------------->|   after Flow
                   start                                                                  end
 client DNS, TCP,  | route, request   | connect to  | upstream-request    | response   | response body: backend    | tail
 TLS handshake,    | policies, pick   | the backend | policies, send,     | policies   | -> Flow -> client         | to the
 request headers   | a backend        |             | backend, headers    |            | (ends at Flow's last      | client
                   |                  |             | come back           |            |  write)                   |
 not measured      | overhead         | connect     | upstream            | overhead   | client_transfer           | not measured
```

`overhead + connect + upstream + client_transfer = flow_request_duration_seconds`. Overhead is the remainder, so anything that is not one of the other three lands in it.

### Reconciling with what a client sees

A client's total time is longer than `flow_request_duration_seconds`. The difference is mostly time Flow cannot see:

- **Before Flow:** the client's DNS lookup, TCP connection, TLS handshake, and the request headers crossing the network. A cold connection from a distant client can add 100 ms or more; a reused keep-alive connection adds very little.
- **After Flow:** Flow stops the clock when its last write is accepted by the operating system, not when the client has received the bytes. For a large download to a slow client the tail can be long.

You cannot tell from the metrics how the difference splits between the two directions, or how much of it is client-side delay. Use client-side measurement (a synthetic probe or browser timing) for the true end-to-end figure.

### Where a metric can point at the wrong place

Inside Flow's window nothing is unaccounted for, but some time is attributed to the metric next to where it really belongs:

| What is really happening | Where it shows up | How it can mislead |
|---|---|---|
| The backend streams its response body slowly | `client_transfer` | Looks like a slow client link. `client_transfer` covers the body from the backend through Flow to the client, and the two are not separated. |
| `upstream-request` policies run | `upstream` | Counted as backend time, because the upstream window opens before they run. |
| A client uploads a large request body slowly | `upstream` if the backend answers after reading the whole body (the usual case); `client_transfer` if the backend answers before the upload finishes | Looks like a slow backend, or a slow download. There is no separate upload metric. Never `overhead`. |
| A policy waits on an external service (for example token introspection or a JWKS fetch) | `overhead` | Looks like Flow being slow when Flow is waiting for a third party. |
| Routing, request-phase policies and backend selection | `overhead` | Reported as one lump; they are not separated, and neither are the response-phase policies. |

### Responses Flow produced itself

When Flow answers without a backend response (`source="flow"`), only some stages exist, and the metrics follow:

**A policy rejection, no matching route, `405`/`501`, request termination, or a cache hit.** The response is written in the request phase, so there is no connect, upstream or client-transfer stage:

```
 start | route, policies, rejection / cache hit / no route | write response to client | end
       |                            overhead = the whole window                       |
```

Only `flow_request_duration_seconds`, `flow_overhead_duration_seconds` and `flow_requests_total{source="flow"}` are recorded. Overhead equals the request duration. **Writing the response to the client counts as overhead here**, because there is no client-transfer stage, so a slow client reading a large cached body inflates overhead for cache hits. A cache hit's lookup time is in that window too.

**The backend connection failed** (typically a `502`):

```
 start | route, policies, pick | try to connect ... fails | write 502 | end
       |            overhead = the whole window, including the failed attempt     |
```

`flow_upstream_connect_duration_seconds` is not recorded (no connection was made), so the time spent trying to connect, including any connect timeout, appears in **overhead**.

**The backend accepted the request but did not answer** (a timeout or reset before any response headers):

```
 start | route.. pick | connect | upstream-request policies, send, wait ... timeout | write 502 | end
       |   overhead    | connect |                        overhead                    |
```

`flow_upstream_connect_duration_seconds` is recorded, but `flow_upstream_duration_seconds` is not, because it needs the response headers. The entire wait for the backend therefore appears in **overhead**, with `source="flow"` and a `5xx` status class. A backend that hangs and then times out looks like high overhead, not high upstream latency.

To catch these cases, alert on `5xx` with `source="flow"` rather than on overhead alone, and compare overhead with connect and upstream:

```promql
# 5xx that Flow produced (rejections, failed connections, backend that never answered)
sum by (service, route) (rate(flow_requests_total{status_class="5xx", source="flow"}[5m]))
```

---

## Cardinality

The number of series is bounded by your configuration, not by traffic:

- Labels use fixed sets (methods, status classes, sources, stream kinds) and your own service and route names. Nothing from the request (path, host, header values, client address) ever becomes a label.
- Each route exports about 140 histogram series (7 histograms, each with its buckets plus sum and count) from startup, plus one counter series per `(method, status_class, source)` combination that has actually occurred. Each service adds the same histogram set for its `unmatched` route.
- A route that serves streams adds about 25 series per stream kind it has seen.
- `dns` adds a handful of series per DNS hostname; `bandwidth` adds two per route.

As a rule of thumb, 100 routes cost roughly 14,000 histogram series. If that is too many for your Prometheus, reduce the number of routes that need separate names, or drop the histograms you do not use with `metric_relabel_configs`.

The overhead of recording is small: the request path performs only atomic operations and allocates nothing once a series exists.

---

## Useful queries

```promql
# Requests per second by route
sum by (service, route) (rate(flow_requests_total[5m]))

# Share of responses that are 5xx
sum(rate(flow_requests_total{status_class="5xx"}[5m]))
  / sum(rate(flow_requests_total[5m]))

# 5xx that the backend produced vs 5xx that Flow produced
sum by (source) (rate(flow_requests_total{status_class="5xx"}[5m]))

# p99 time Flow itself adds, per route
histogram_quantile(0.99,
  sum by (le, route) (rate(flow_overhead_duration_seconds_bucket[5m])))

# p99 backend latency, per route
histogram_quantile(0.99,
  sum by (le, route) (rate(flow_upstream_duration_seconds_bucket[5m])))

# Share of requests that had to open a new backend connection
sum(rate(flow_upstream_connect_duration_seconds_count{reused="false"}[5m]))
  / sum(rate(flow_upstream_connect_duration_seconds_count[5m]))

# Open streams per route and kind
sum by (route, kind) (flow_active_streams)

# Streams that ended because the backend failed
sum by (route) (rate(flow_streams_total{end="upstream"}[5m]))

# DNS name served from a stale answer for more than 5 minutes (needs the dns group)
time() - flow_dns_last_success_timestamp_seconds > 300

# DNS name with no addresses at all
flow_dns_resolved_addresses == 0
```

---

## Not yet available

These are planned and are **not** exposed today:

- Per-backend state (healthy, unhealthy, circuit open) and circuit-breaker state as metrics.
- Per-policy metrics (calls, outcomes and duration for individual policies, enabled per policy instance), and cache hit/miss counters.
- TLS and authentication on the metrics listener.
- A ready-made Grafana dashboard and alert rules.

See [What Flow Is](scope.md) for the roadmap.
