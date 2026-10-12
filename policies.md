# Policies

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Performance](performance.md)

Policies (called *filters* in the config) inspect or rewrite traffic as it passes through
Flow. They are declared in a `path-control` block.

## Available policies

| Category | Policy | What it does |
|---|---|---|
| Authentication | [Token authentication](policies/token-auth.md) | Verify a bearer/cookie/query token — JWT (JWKS or static key) or opaque token via RFC 7662 introspection |
| Security | [IP restriction](policies/ip-restriction.md) | Allow or block requests by source IP (CIDR ranges) |
| Traffic Control | [Rate limiting](policies/rate-limiting.md) | Cap request rates — per IP, per client, per URI, or per custom dimension |
| Transformation | [Upsert header](policies/upsert-header.md) | Set a header on the request or the response |
| Transformation | [Remove header](policies/remove-header-key-regex.md) | Strip headers whose name matches a pattern |
| Observability | [Timing header](policies/timing-header.md) | Add latency timing headers to the response |

## Where policies attach

`path-control` can sit at the **service** level — applying to every request the service
handles — or on an individual **route**. Service-level filters run first.

```kdl
services {
    MyGateway {
        // Applies to every route in this service
        path-control {
            request-filters {
                filter kind="ip-restriction" deny="192.168.0.0/16"
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

## Execution order

For the `request-filters` phase, the full sequence for one incoming request is:

1. **Route matching** — Flow picks the best-matching route. No match → `502` immediately, no
   filter runs at all.
2. **Protocol enforcement** — the matched route's `allowed-protocols` is checked. A mismatch
   returns `426`/`400`, still before any filter runs.
3. **Service-level `request-filters`**, in the order they're written.
4. **Route-level `request-filters`** (for the matched route only), in the order they're
   written — only reached if every service-level filter passed.

**This is a first-reject-wins chain.** The moment one filter rejects the request (or errors),
every filter after it — in that phase, at that scope — is skipped entirely. It doesn't matter
whether the later filter would also have rejected the request or not; it never runs, so it
never gets the chance to log, count, or otherwise act on that request.

**Service level always runs before route level.** If the same kind of policy is attached both
on the service and on a specific route, the service-level instance evaluates first.

### Worked example

```kdl
services {
    MyGateway {
        path-control {
            request-filters {
                filter kind="ip-restriction" allow="203.0.113.0/24"
                filter kind="rate-limit" key-from="source-ip" max-tokens=100 refill-qty=10 refill-interval-ms=100 max-buckets=50000 threads=8
            }
        }

        routes {
            route {
                match { path-prefix "/api" }
                path-control {
                    request-filters {
                        filter kind="token-auth" {
                            issuers {
                                issuer name="https://idp.example.com" \
                                    jwks-uri="https://idp.example.com/.well-known/jwks.json" \
                                    algorithm="RS256"
                            }
                        }
                    }
                }
                ...
            }
        }
    }
}
```

A request to `/api` is checked in exactly this order:

1. Routed to this `route` (path-prefix `/api` matches) and its protocol checked.
2. **`ip-restriction`** (service level, written first) — a client IP in `203.0.113.0/24` is
   rejected here. The `rate-limit` and `token-auth` filters never run for that request —
   an attacker in the blocked range never even gets to try presenting a token.
3. **`rate-limit`** (service level, written second) — only reached if the IP wasn't blocked.
   A client over its quota is rejected here; `token-auth` still never runs —
   Flow never bothers verifying a token for a request it's about to rate-limit anyway.
4. **`token-auth`** (route level) — only reached once both service-level filters passed.
   Missing/invalid token → `401`/`503` here, and the request never reaches your backend.

Only a request that clears all three — not CIDR-blocked, not rate-limited, and carrying a valid
token — reaches `upstream-request`/`upstream-response` phase filters and then your backend.

**Ordering has real cost implications.** Putting cheap, high-rejection-rate checks first (an IP
block list, a rate limiter) means expensive checks (JWT signature verification, especially
RSA/ECDSA) never run for traffic that was going to be rejected anyway. Putting `token-auth`
first instead would mean verifying a signature for every request, even ones from an IP you were
always going to block — usually wasted work. There's no automatic reordering by cost; this is a
config choice you make by write order.

## Stacking the same policy multiple times

Each filter declaration is an independent instance. You can list the same `kind` more than once
— at the same scope, across scopes, or both — and each instance runs as a separate check with
its own state. There is no cross-instance aggregation.

**Multiple instances at the same scope** are all evaluated in order. A request must pass every
one of them. This is the correct way to enforce several independent dimensions simultaneously:

```kdl
request-filters {
    // Per-IP guard — one noisy address cannot starve others
    filter kind="rate-limit" key-from="source-ip" \
        max-tokens=100 refill-qty=10 refill-interval-ms=100 \
        max-buckets=50000 threads=8
    // Per-client quota — each authenticated tenant gets its own budget
    filter kind="rate-limit" key-from="client-id" \
        max-tokens=500 refill-qty=50 refill-interval-ms=1000 \
        max-buckets=100000 threads=8
}
```

Both limits apply independently. A client that exceeds its per-IP allowance is rejected by the
first filter and the second never runs. A client within its IP allowance but over its client
quota is passed by the first and rejected by the second.

**Instances at different scopes** (service + route) also both apply, subject to the
first-reject-wins rule. The service-level instance runs first and acts as an outer cap; the
route-level instance runs second and acts as a tighter inner cap for that route:

```kdl
service {
    // Outer cap — all routes, all clients
    path-control {
        request-filters {
            filter kind="rate-limit" key-from="source-ip" \
                max-tokens=200 refill-qty=20 refill-interval-ms=1000 \
                max-buckets=50000 threads=8
        }
    }
    routes {
        route {
            match { path-prefix "/api/export" }
            // Inner cap — tighter limit on this expensive route only
            path-control {
                request-filters {
                    filter kind="rate-limit" key-from="client-id" \
                        max-tokens=5 refill-qty=1 refill-interval-ms=60000 \
                        max-buckets=10000 threads=8
                }
            }
            // ...
        }
    }
}
```

A request to `/api/export` must clear both: the service-level IP cap, then the route-level
client cap. A request to any other route only sees the service-level cap.

**Buckets are never shared.** Two `rate-limit` declarations with the same `key-from` maintain
completely separate bucket stores — traffic consumed against one has no effect on the other.
