# Request termination

Returns a configured HTTP response immediately without forwarding the request to any upstream.
Phase: `request-filters`.

## When to use it

- **Maintenance mode** — return 503 for all traffic during a planned outage
- **Deprecated endpoint sunset** — return 410 Gone with a migration message
- **Emergency kill switch** — block a specific route or service instantly without removing it from config
- **Static mock** — return a fixed response on a route during integration testing

## Basic usage

Status code only — no body:

```kdl
path-control {
    request-filters {
        filter kind="request-termination" status=503
    }
}
```

## With a message

`message` is wrapped in a JSON envelope: `{"message":"..."}` with `Content-Type: application/json`.

```kdl
filter kind="request-termination" status=503 message="Down for maintenance"
```

## With a raw body

`body` is returned verbatim. `content-type` is required when `body` is set.

```kdl
filter kind="request-termination" status=503 \
    body="<html><h1>Maintenance</h1></html>" \
    content-type="text/html"
```

```kdl
filter kind="request-termination" status=200 \
    body="{\"ok\":true}" \
    content-type="application/json"
```

`message` and `body` are mutually exclusive — use one or the other.

## Conditional termination (`trigger`)

When `trigger` is set, the filter only terminates the request if that name appears as a
**request header** or **query parameter**. Requests without the trigger pass through normally.

```kdl
// Only terminate requests that include the header (or ?x-maintenance=...) —
// all other traffic reaches the upstream as normal.
path-control {
    request-filters {
        filter kind="request-termination" status=503 message="Maintenance window" \
            trigger="x-maintenance"
    }
}
```

Activate with:
```bash
curl -H "x-maintenance: 1" https://api.example.com/endpoint
```

Or via query parameter:
```bash
curl "https://api.example.com/endpoint?x-maintenance=1"
```

## Extra response headers

Use the `headers` block to add headers to the termination response:

```kdl
filter kind="request-termination" status=503 message="Unavailable" {
    headers {
        header key="Retry-After" value="3600"
        header key="X-Sunset-Date" value="2026-12-01"
    }
}
```

## Emergency kill switch

Block a specific route instantly — no upstream is called, config change takes effect on next
reload without removing or restructuring the route:

```kdl
route {
    name "api-payments"
    match {
        path-prefix "/api/payments"
    }
    upstream "payments-backend"
    path-control {
        request-filters {
            filter kind="request-termination" status=503 message="Payments temporarily disabled"
        }
    }
}
```

Remove the filter line and reload to re-enable — the route and upstream are untouched.

## Static mock response

Return a fixed response for testing or stubbing an upstream that isn't ready:

```kdl
route {
    name "api-v2-orders"
    match {
        path-prefix "/api/v2/orders"
    }
    upstream "orders-backend"
    path-control {
        request-filters {
            filter kind="request-termination" status=200 \
                body="{\"orders\":[]}" \
                content-type="application/json"
        }
    }
}
```

## Options

| Option | Default | Meaning |
|---|---|---|
| `status` | — | **Required.** HTTP status code (100–599). |
| `message` | — | Plain string body; wrapped as `{"message":"..."}` with `Content-Type: application/json`. Mutually exclusive with `body`. |
| `body` | — | Raw verbatim response body. Requires `content-type`. Mutually exclusive with `message`. |
| `content-type` | — | `Content-Type` header for `body`. Required when `body` is set; not valid without it. |
| `trigger` | — | Header or query parameter name. When set, only terminate if that name is present on the request; otherwise pass through. |
| `headers { }` | — | Optional block of extra response headers. Each entry: `header key="Name" value="val"`. |

## Distinction from `redirect`

A redirect returns a `3xx` with a `Location` header telling the client to go elsewhere.
`request-termination` ends the request here with any status and body — the client is not sent anywhere.

> **Planned:** a `redirect` policy. See [Roadmap](../roadmap.md#traffic-control).

## Where to attach it

**Service level** — terminate all traffic entering a service:

```kdl
services {
    MyGateway {
        path-control {
            request-filters {
                filter kind="request-termination" status=503 message="Down for maintenance"
            }
        }
        routes {
            ...
        }
    }
}
```

**Route level** — sunset a specific path:

```kdl
route {
    name "api-v1-legacy"
    match {
        path-prefix "/api/v1/legacy"
    }
    upstream "backend"
    path-control {
        request-filters {
            filter kind="request-termination" status=410 \
                message="This endpoint is retired — use /api/v2/"
        }
    }
}
```
