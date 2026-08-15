# Block CIDR range

Rejects requests whose source IP falls inside any of the listed CIDR ranges.

**Phase:** `request-filters`

```kdl
path-control {
    request-filters {
        filter kind="block-cidr-range" addrs="127.0.0.0/16, 10.0.0.0/8"
    }
}
```

## Options

| Option | Meaning |
|---|---|
| `addrs` | Comma-separated list of CIDR ranges to block |

Both IPv4 and IPv6 ranges are accepted.

## Where to attach it

Put it at the **service** level to block traffic from reaching any route:

```kdl
services {
    MyGateway {
        path-control {
            request-filters {
                filter kind="block-cidr-range" addrs="192.168.0.0/16, 10.0.0.0/8"
            }
        }
        routes { ... }
    }
}
```

Or on a single **route**, to protect just that path:

```kdl
route {
    match { path-prefix "/admin" }
    ...
    path-control {
        request-filters {
            filter kind="block-cidr-range" addrs="0.0.0.0/0"
        }
    }
}
```

## Trying it out

```bash
# Blocked — request is rejected
curl -v http://localhost:8080/admin

# From an allowed range — passes through
curl -v http://localhost:8080/api
```

## A caution about source IPs

This policy matches on the **source IP of the TCP connection**. If Flow sits behind another
proxy, a CDN, or a cloud load balancer, that source IP is the **upstream proxy's** address —
not the real client's. Blocking on it will not do what you expect.

Flow does not currently derive the client IP from `X-Forwarded-For`. If something else
terminates connections in front of Flow, apply IP blocking there instead.

## Related

- [Rate limiting](rate-limiting.md) — to throttle rather than block outright
- [Policies index](../policies.md)
