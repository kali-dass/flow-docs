# IP restriction

Rejects requests whose source IP falls inside any of the listed CIDR ranges.

**Phase:** `request-filters`

```kdl
path-control {
    request-filters {
        filter kind="ip-restriction" addrs="127.0.0.0/16, 10.0.0.0/8"
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
                filter kind="ip-restriction" addrs="192.168.0.0/16, 10.0.0.0/8"
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
            filter kind="ip-restriction" addrs="0.0.0.0/0"
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

This policy matches on the **source IP of the TCP connection** by default. If Flow sits behind
a proxy, CDN, or cloud load balancer, that source IP is the **upstream proxy's** address —
not the real client's. Blocking on it will not do what you expect.

To block on the real client IP in a proxied deployment, configure `trusted-proxies` and
`real-ip-source` at the system level:

```kdl
system {
    trusted-proxies {
        cidr "10.0.0.0/8"              // your LB / proxy CIDR range
    }
    real-ip-source "xff"               // or "header:CF-Connecting-IP" for Cloudflare
}
```

Flow then resolves the real client IP from the forwarded header before evaluating the block
list. `real-ip-source` requires `trusted-proxies` — Flow refuses to start without it. See
[Rate limiting — proxy deployments](rate-limiting.md#proxy-deployments-and-source-ip) for the
full CDN header reference table and security notes.

## Related

- [Rate limiting](rate-limiting.md) — to throttle rather than block outright
- [Policies index](../policies.md)
