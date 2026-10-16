# Correlation ID

Injects a correlation ID into every request so that client logs, gateway logs, and upstream logs can all be linked by the same identifier.
Phase: `request-filters` (reads incoming header, stores ID, writes to downstream request). Also runs in `upstream-request` (propagates to upstream) and optionally in `upstream-response` (echoes back to the caller).

## When to use it

- **Log correlation** — one header ties the client, gateway, and every downstream service together without a tracing agent
- **Debug / support** — include the request ID in error responses so callers can report it to your team
- **Anti-spoofing** — set `trust-incoming=false` when the upstream must never trust a client-supplied ID
- **Multi-service fan-out** — attach at service level so every route gets a correlation ID without per-route config

## Basic usage

Default: `uuid-counter` generator, `x-request-id` header, trusts incoming header if present.

```kdl
path-control {
    request-filters {
        filter kind="correlation-id"
    }
}
```

Flow reads `x-request-id` from the incoming request. If present and non-empty, it is reused as-is — the same ID travels from the client through Flow to every upstream. If absent, Flow generates a new ID. The ID is forwarded to the upstream on every request.

## Scenario: behind AWS ALB, Nginx, or Envoy

AWS ALB, Nginx (`$request_id`), and Envoy all generate and propagate `x-request-id` automatically. If Flow sits behind any of these, the upstream component has already stamped an ID on the request before Flow sees it.

With the default config (`trust-incoming=true`, `header="x-request-id"`), Flow reuses that ID — **no config needed**. The same ID flows end-to-end from the load balancer through Flow to every upstream without duplication or overwriting.

```kdl
// Zero config needed when behind AWS ALB, Nginx, or Envoy.
// Flow reuses the x-request-id they inject.
filter kind="correlation-id"
```

If you run Flow as the outermost entry point with no upstream load balancer, this same config generates the ID itself instead.

## Scenario: client supplies its own correlation ID

Some API clients (SDKs, service mesh sidecars, browser apps) generate their own correlation ID and send it on every request. The default `trust-incoming=true` preserves that ID so client logs, Flow logs, and backend logs all share the same value without any client-side changes.

```kdl
filter kind="correlation-id" echo-downstream=true
```

`echo-downstream=true` writes the ID back to the response so the client can confirm which ID Flow used — useful when the client generates the ID and needs it confirmed, or when error-handling middleware reads it from the response.

To prevent clients from injecting arbitrary IDs into your backend (anti-spoofing), set `trust-incoming=false` — Flow always generates a fresh ID regardless of what the client sent:

```kdl
filter kind="correlation-id" trust-incoming=false
```

## Scenario: OpenTelemetry / W3C traceparent passthrough

If your stack uses OpenTelemetry, the standard tracing header is `traceparent` (W3C Trace Context). You can use the correlation-id policy as a **header passthrough**: preserve an incoming `traceparent` from your OTel SDK or service mesh and forward it to all upstreams unchanged.

```kdl
filter kind="correlation-id" header="traceparent" trust-incoming=true
```

With this config, a request carrying `traceparent` passes through Flow to the upstream without modification — the existing trace context is preserved end-to-end.

**Limitation:** this is header passthrough only. Flow does not generate valid W3C trace context values, create spans, or record trace data. A request without `traceparent` gets a plain UUID injected under that header — not a valid trace context.

Full OTel integration (span creation, `traceparent` generation, `tracestate` propagation, trace export) is a separate planned policy. When that ships, the correlation-id passthrough approach here becomes unnecessary for OTel stacks — use the OTel policy instead.

## Scenario: Azure API Management or enterprise middleware

If your stack uses `x-correlation-id` (common in Azure APIM, IBM API Connect, and .NET/Java middleware), configure the header name explicitly:

```kdl
filter kind="correlation-id" header="x-correlation-id"
```

Everything else works identically — the policy name `correlation-id` describes the intent; the actual header is always configurable.

## Echo back to the caller

Set `echo-downstream=true` to include the ID in the response. Useful when the client needs to log the ID that was actually used (especially when `trust-incoming=false` forces a new ID) or when a mobile/browser client wants to report it in a bug report.

```kdl
filter kind="correlation-id" echo-downstream=true
```

## Generators

| Generator | Format | Use case |
|---|---|---|
| `uuid-counter` (default) | `<worker-uuid>#<n>` | Unique per request, monotonically numbered per worker — easy to correlate in logs |
| `uuid-v4` | UUID v4 string | Globally unique, no per-worker sequence — use when IDs must be unguessable across workers |

`uuid-counter` generates a UUID once at startup per worker process and appends a monotonically incrementing counter. IDs from different workers have different UUID prefixes; IDs within one worker are ordered by the suffix — which aids log sorting and correlation.

```kdl
filter kind="correlation-id" generator="uuid-v4"
```

## Options

| Option | Default | Meaning |
|---|---|---|
| `header` | `x-request-id` | Header name to read, write, and optionally echo. Set to `x-correlation-id` for Azure/enterprise stacks, `traceparent` for OTel passthrough, or any custom name. |
| `generator` | `uuid-counter` | ID generation strategy when no incoming header is reused. `uuid-v4` or `uuid-counter`. |
| `trust-incoming` | `true` | When `true`, a non-empty incoming header is reused as-is (for AWS/Nginx/client-supplied IDs). When `false`, always generate a fresh ID (anti-spoofing). |
| `echo-downstream` | `false` | When `true`, the correlation ID is written to the response header. |

## Where to attach it

**Service level** — every route in the service gets a correlation ID. Recommended default placement:

```kdl
services {
    MyGateway {
        path-control {
            request-filters {
                filter kind="correlation-id"
            }
        }
        routes { ... }
    }
}
```

**Route level** — different settings per route:

```kdl
route {
    match { path-prefix "/api/v1" }
    upstream "backend"
    path-control {
        request-filters {
            filter kind="correlation-id" echo-downstream=true trust-incoming=false
        }
    }
}
```
