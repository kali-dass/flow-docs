# DNS Resolution

> **New here?** Start with [Getting Started](getting-started.md). The full connector and `system` reference is in the [Configuration Reference](configuration.md).

Flow resolves upstream hostnames itself, keeps the resolved set of IP addresses current in the background, and balances requests across every address a name returns. This page explains how that works, what Flow does when DNS misbehaves, and how to configure it.

---

## What gets resolved

A connector address is either an IP literal or a hostname:

```kdl
connectors {
    "10.0.0.1:3000"                      // IP literal: used as-is, never resolved
    "orders-svc.default.svc.cluster.local:8080"   // hostname: resolved and refreshed
}
```

| Connector address | Discovery mode | Behavior |
|---|---|---|
| IP literal | `Static` | Fixed for the life of the process. No DNS. |
| Hostname | `Dns` | Resolved at startup, then re-resolved in the background. |

You can force `discovery "Static"` on a hostname to resolve it once at startup and keep only the first address. Use it only for stable single-address targets. A static lookup uses the operating system resolver, so the `dns-resolver` settings below do not apply to it, and it is never retried: if it fails at startup the connector stays unreachable until Flow restarts. See [Discovery](configuration.md#discovery).

Always write connector addresses as `host:port`. Flow does not infer a default port.

---

## Where answers come from

When Flow looks up a name, it consults these sources:

1. **`/etc/hosts`**: static `IP hostname` entries. Checked first, no network involved.
2. **DNS servers**: queried over the network. By default these come from `/etc/resolv.conf` (correct for Kubernetes, Docker, and bare-metal). If you configure `system { dns-resolver { nameserver … } }`, Flow uses exactly those servers and ignores `resolv.conf`.

`/etc/resolv.conf` also supplies the `search` domains, `ndots`, timeout, and attempts unless you override them in a `dns-resolver` block (see [Configuring the resolver](#configuring-the-resolver)).

### Which record types are used

By default Flow asks for both **A** (IPv4) and **AAAA** (IPv6) records and uses every address returned, so a dual-stack name contributes both IPv4 and IPv6 backends. If your backends publish AAAA records that Flow's network cannot reach, restrict the lookup with `ip-version`:

```kdl
system {
    dns-resolver {
        ip-version "ipv4"    // "ipv4" | "ipv6" | "dual" (default: dual)
    }
}
```

| `ip-version` | Records queried | Addresses used |
|---|---|---|
| `"dual"` (default) | A and AAAA | Every IPv4 and IPv6 address the name returns |
| `"ipv4"` | A only | IPv4 addresses only |
| `"ipv6"` | AAAA only | IPv6 addresses only |

Things to know:

- **Each address is its own backend with equal weight.** Flow does not try one family first and fall back to the other, so with `"dual"` an unreachable family still receives its share of requests until health checking or the circuit breaker removes it.
- **The setting is global.** It applies to every hostname connector; there is no per-connector override.
- **A mismatch empties the pool.** With `"ipv4"`, a name that has only AAAA records returns no addresses. That counts toward `empty-answer-threshold`, and after that many consecutive empty answers the pool is cleared and the route returns `502` (and the reverse for `"ipv6"`).
- **It does not apply to IP-literal connectors** (never resolved) **or to `discovery "Static"` hostnames** (resolved by the operating system). It also has no effect on which address families clients use to reach Flow's listeners.

SRV records are not supported yet.

---

## Multiple addresses: DNS as a load balancer

If a hostname returns several addresses, Flow keeps **all** of them as backends and balances requests across them with the route's `selection` algorithm. This works well for services that publish one A record per instance (Kubernetes headless services, internal service names, multi-IP vendor endpoints):

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
    }
    "api.vendor.internal:443" tls-sni="api.vendor.internal" proto="h2-or-h1"
}
```

Compared with leaving the balancing to clients (plain DNS round-robin):

- **The choice is made per request, inside Flow.** With client-side DNS round-robin, each client picks an address from a cached answer and then keeps its connection for a long time, so load is only as even as resolver caching and keep-alive allow.
- **Failed backends stop receiving traffic quickly** when combined with [active health checking](configuration.md#health-checking) and [circuit breaking](configuration.md#circuit-breaking), rather than waiting for DNS TTLs to expire.
- **Address changes are picked up without a restart** (see below).

Things to keep in mind:

- Every returned address gets equal weight. DNS-level weighting tricks do not carry over.
- Balancing is by request count, not by load. Flow has no least-connections or latency-aware selection yet.
- A change in the DNS answer is visible to Flow only at the next re-resolution (default 30 seconds).
- Hash-based selection (`"FNV"`, `"Ketama"`) trades even spread for affinity, and the mapping shifts when the address set changes.

---

## Refreshing: how changes are picked up

Flow re-resolves each hostname on a fixed interval:

```kdl
"catalog.internal:443" proto="h2-or-h1" dns-interval-secs=30
```

| Attribute | Default | Meaning |
|---|---|---|
| `dns-interval-secs` | `30` | Seconds between re-resolutions of this hostname. Lower values react to DNS changes faster at the cost of more queries. Ignored on IP-literal connectors. |

A successful re-resolution replaces the backend set atomically. The interval is fixed and does not follow the record TTL.

> **Planned:** a refresh interval that follows the record TTL. See [Roadmap](roadmap.md#routing-and-load-balancing).

---

## Watching address changes

Whenever the set of addresses behind a hostname changes (including the first resolution and a pool clear), Flow logs one `info` line naming the hostname and the `added` and `removed` addresses:

```
INFO DNS address set changed hostname=orders-svc.default.svc.cluster.local added=[10.0.1.7] removed=[10.0.1.4]
```

(Timestamp and module prefix omitted.)

Nothing is logged when a refresh returns the same set.

---

## When DNS fails

| Situation | What Flow does |
|---|---|
| Hostname fails to resolve at startup | Logs a warning and starts with an **empty pool** for that connector. Requests to its routes return `502` until a re-resolution succeeds. Flow does not refuse to start. |
| Transient error on re-resolution (timeout, SERVFAIL, I/O error) | Keeps the **last known-good** addresses and logs a warning on each failure. |
| Name no longer exists, or returns no addresses (NXDOMAIN / empty answer) | Keeps the last known-good addresses until `empty-answer-threshold` (default **3**) consecutive empty results have occurred, then clears the pool, and routes return `502`. |
| A later lookup succeeds | Resets the failure count and replaces the pool. |

Two consequences worth knowing:

- A short resolver outage does not take traffic down, because the old addresses keep serving.
- A long resolver outage is not bounded yet: Flow keeps serving the last known-good addresses for as long as errors are transient. Pair DNS discovery with health checking so a dead address is removed from rotation even when DNS cannot tell you.

---

## Configuring the resolver

The default behavior needs no configuration. To point Flow at specific DNS servers, add a `dns-resolver` block under `system`:

```kdl
system {
    dns-resolver {
        nameserver "10.96.0.10:53"        // Kubernetes CoreDNS
        nameserver "8.8.8.8:53"           // fallback
        search-domain "svc.cluster.local"
        search-domain "cluster.local"
        ndots 5
        timeout-secs 5
        attempts 2
        ip-version "dual"
        empty-answer-threshold 3
    }
}
```

| Setting | Repeats | Default | Meaning |
|---|---|---|---|
| `nameserver` | yes | *(from `resolv.conf`)* | DNS server as `IP:port`. When set, replaces the system servers entirely. Provide at least two for redundancy. |
| `search-domain` | yes | *(from `resolv.conf`)* | Domain appended when a bare name does not resolve. Same as `search` in `resolv.conf`. |
| `ndots` | no | *(`resolv.conf`, else `1`)* | A name with fewer dots than this is tried with the search domains first. Kubernetes clusters conventionally use `5`. |
| `timeout-secs` | no | *(`resolv.conf`, else `5`)* | Per-query timeout before retrying or failing. |
| `attempts` | no | *(`resolv.conf`, else `2`)* | Attempts per query before returning a failure. |
| `ip-version` | no | `"dual"` | Record families to resolve: `"ipv4"` (A only), `"ipv6"` (AAAA only), or `"dual"` (both). Global to all hostname connectors. See [Which record types are used](#which-record-types-are-used). |
| `empty-answer-threshold` | no | `3` | Consecutive NXDOMAIN / empty answers before the pool is cleared. Integer ≥ 1. Raise it to ride out longer resolver glitches; lower it to fail fast. |

The `search-domain`, `ndots`, `timeout-secs`, and `attempts` settings apply whether or not you set `nameserver`. Without `nameserver`, Flow starts from `/etc/resolv.conf` and overrides only the settings you specify; with `nameserver`, it starts from built-in defaults. A `search-domain` list replaces the `search` list from `resolv.conf`.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `502` on a route right after start | The hostname failed to resolve at startup and the pool is empty. Check the startup log for `initial DNS discovery failed`; the pool fills at the next successful re-resolution. |
| Traffic still going to an old address | The change has not reached Flow yet (up to `dns-interval-secs`), or the answer is cached by an upstream resolver for its TTL. |
| Requests failing against unreachable IPv6 addresses | The name publishes AAAA records Flow cannot reach. Set `ip-version "ipv4"`. |
| Short names do not resolve | Search domains or `ndots` are not what you expect. Use the fully qualified name (`name.namespace.svc.cluster.local`) or set `search-domain` together with `nameserver`. |
| Pool cleared and `502`s after an outage | `empty-answer-threshold` (default 3) consecutive empty or NXDOMAIN answers clear the pool. Fix the name or the resolver; the next successful lookup restores it. |

---

## See also

- [Configuration Reference: Discovery](configuration.md#discovery): `discovery` modes and connector attributes
- [Health checking](configuration.md#health-checking) and [Circuit breaking](configuration.md#circuit-breaking): removing failed backends between DNS refreshes
- [Zero-downtime upgrades](operations.md#zero-downtime-upgrades): graceful shutdown and DNS-based drains
