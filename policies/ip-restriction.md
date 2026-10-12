# IP restriction

Accepts or rejects requests based on the source IP address. Supports allow lists, deny lists,
and combined allow+deny configurations. Phase: `request-filters`.

## Deny list (block specific ranges)

Reject requests from listed CIDRs; all other traffic passes.

```kdl
path-control {
    request-filters {
        filter kind="ip-restriction" deny="192.168.0.0/16, 10.0.0.0/8"
    }
}
```

## Allow list (permit only specific ranges)

Only listed CIDRs pass; everything else is rejected with 403.

```kdl
path-control {
    request-filters {
        filter kind="ip-restriction" allow="203.0.113.0/24, 198.51.100.5/32"
    }
}
```

Useful for internal APIs, partner-only endpoints, and admin routes.

## Combined allow + deny

Allow a broad range but carve out a specific IP. Deny takes precedence when both match.

```kdl
path-control {
    request-filters {
        filter kind="ip-restriction" allow="10.0.0.0/8" deny="10.0.1.5/32"
    }
}
```

## Options

| Option | Default | Meaning |
|---|---|---|
| `allow` | — | Comma-separated CIDRs to permit. All other IPs are rejected. |
| `deny` | — | Comma-separated CIDRs to reject. |
| `status` | `403` | HTTP status code returned on rejection (any 4xx). |
| `message` | — | Optional plain-text body included in the rejection response. |

At least one of `allow` or `deny` must be present.

Both IPv4 and IPv6 CIDRs are accepted in all fields.

### Custom rejection

```kdl
filter kind="ip-restriction" deny="192.168.0.0/16" status=403 message="Access denied by policy"
```

## Where to attach it

**Service level** — block traffic before it reaches any route:

```kdl
services {
    MyGateway {
        path-control {
            request-filters {
                filter kind="ip-restriction" deny="192.168.0.0/16, 10.0.0.0/8"
            }
        }
        routes { ... }
    }
}
```

**Route level** — protect a single path:

```kdl
route {
    match { path-prefix "/admin" }
    ...
    path-control {
        request-filters {
            filter kind="ip-restriction" allow="10.0.0.0/8"
        }
    }
}
```

## Source IP in proxied deployments

This policy matches the **resolved client IP**. By default that is the raw TCP peer address —
if Flow sits behind a proxy, CDN, or cloud load balancer, the peer is the proxy's address, not
the real client's.

To match on the real client IP, configure `trusted-proxies` and `real-ip-source` at the system
level:

```kdl
system {
    trusted-proxies {
        cidr "10.0.0.0/8"              // your LB / proxy CIDR range
    }
    real-ip-source "xff"               // or "header:CF-Connecting-IP" for Cloudflare
}
```

`real-ip-source` requires `trusted-proxies` — Flow refuses to start without it. See
[Configuration](../configuration.md#real-client-ip) for the full CDN header reference and
security notes.

## Related

- [Rate limiting](rate-limiting.md) — to throttle rather than block outright
- [Policies index](../policies.md)
