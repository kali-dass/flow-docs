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
| [JWT validation](policies/jwt-validate.md) | Verify a bearer/cookie/query JWT against JWKS or a static key before routing |
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
                filter kind="block-cidr-range" addrs="203.0.113.0/24"
                filter kind="rate-limit-multi-source-ip" max-tokens=100 refill-qty=10 refill-interval-ms=100
            }
        }

        routes {
            route {
                match { path-prefix "/api" }
                path-control {
                    request-filters {
                        filter kind="jwt-validate" {
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
2. **`block-cidr-range`** (service level, written first) — a client IP in `203.0.113.0/24` is
   rejected here. `rate-limit-multi-source-ip` and `jwt-validate` never run for that request —
   an attacker in the blocked range never even gets to try presenting a token.
3. **`rate-limit-multi-source-ip`** (service level, written second) — only reached if the IP
   wasn't blocked. A client over its quota is rejected here; `jwt-validate` still never runs —
   Flow never bothers verifying a token for a request it's about to rate-limit anyway.
4. **`jwt-validate`** (route level) — only reached once both service-level filters passed.
   Missing/invalid token → `401`/`503` here, and the request never reaches your backend.

Only a request that clears all three — not CIDR-blocked, not rate-limited, and carrying a valid
token — reaches `upstream-request`/`upstream-response` phase filters and then your backend.

**Ordering has real cost implications.** Putting cheap, high-rejection-rate checks first (an IP
block list, a rate limiter) means expensive checks (JWT signature verification, especially
RSA/ECDSA) never run for traffic that was going to be rejected anyway. Putting `jwt-validate`
first instead would mean verifying a signature for every request, even ones from an IP you were
always going to block — usually wasted work. There's no automatic reordering by cost; this is a
config choice you make by write order.
