# Policies

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

Policies (called *filters* in the config) inspect or rewrite traffic as it passes through
Flow. They are declared in a `path-control` block.

## Available policies

| Policy | What it does |
|---|---|
| [Block CIDR range](policies/block-cidr-range.md) | Reject requests from given IP ranges |
| [Rate limiting](policies/rate-limiting.md) | Cap request rates — globally, per client IP, or per URI |
| [Upsert header](policies/upsert-header.md) | Set a header on the request or the response |
| [Remove header](policies/remove-header-key-regex.md) | Strip headers whose name matches a pattern |
| [Timing header](policies/timing-header.md) | Add latency timing headers to the response |

## Where policies attach

`path-control` can sit at the **service** level — applying to every request the service
handles — or on an individual **route**. Service-level filters run first.

```kdl
services {
    MyGateway {
        // Applies to every route in this service
        path-control {
            request-filters {
                filter kind="block-cidr-range" addrs="192.168.0.0/16"
            }
        }

        routes {
            route {
                match { path-prefix "/api" }
                ...
                // Applies only to this route
                path-control {
                    upstream-request {
                        filter kind="upsert-header" key="x-api-key" value="secret"
                    }
                }
            }
        }
    }
}
```

## The three phases

Every policy runs in one of three phases. The phase is chosen by the block you put the filter
in, not by the filter itself.

| Block | When it runs | What it can do |
|---|---|---|
| `request-filters` | On the incoming request, before it is routed to an upstream | **Reject** the request (block, rate limit) |
| `upstream-request` | Just before the request is sent to the backend | Rewrite **request** headers |
| `upstream-response` | When the response comes back from the backend | Rewrite **response** headers |

Note that some policies work in more than one phase. `upsert-header` and
`remove-header-key-regex` each work in both `upstream-request` and `upstream-response` — the
block you place them in decides whether they act on the request or the response.

## Filter syntax

Every filter is a `filter` node with a `kind` and its options as attributes:

```kdl
filter kind="<policy-name>" option="value" other-option=123
```

Filters run in the order they are written.
