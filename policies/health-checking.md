# Health checking

Flow supports two complementary approaches to backend failure detection. Use one or both
depending on how your backends fail.

| | Active | Passive |
|---|---|---|
| **How it works** | Synthetic HTTP probes sent on a timer | Observes real request outcomes |
| **What it catches** | Completely down backends — unresponsive to TCP or HTTP | Backends that are up but failing real requests |
| **When it detects** | Before any client request hits the backend | After enough real requests fail |
| **Probe traffic** | Yes — background HTTP GETs per backend | None |
| **Config** | `health-check "Active"` in `load-balance` | `circuit-breaker { }` in `load-balance` |

**Best practice**: use both. Active health checking removes dead backends before real traffic
hits them; circuit breaking catches backends that pass probes but fail in production.

```kdl
load-balance {
    selection "RoundRobin"
    health-check "Active"
    health-check-path "/healthz"
    circuit-breaker {
        error-threshold-pct 50
        window-secs 60
        min-requests 5
        cooldown-secs 30
    }
}
```

---

## Active health checking

Periodically probes each backend with a synthetic HTTP GET and removes unresponsive backends
from the pool before any real client request hits them.

```kdl
load-balance {
    selection "RoundRobin"
    health-check "Active"
    health-check-path "/healthz"         // default
    health-check-interval-secs 10        // seconds between probes — default 10
    health-check-timeout-secs 2          // probe timeout — default 2
    unhealthy-threshold 3                // consecutive failures to mark down — default 3
    healthy-threshold 2                  // consecutive successes to restore — default 2
}
```

The only required field is `health-check "Active"`. All other fields are optional.

### Options

| Field | Default | Description |
|---|---|---|
| `health-check` | `"None"` | `"Active"` enables probing; `"None"` (default) disables it |
| `health-check-path` | `"/healthz"` | HTTP path Flow sends the GET probe to |
| `health-check-interval-secs` | `10` | Seconds between probes per backend. Lower values detect failures faster but add more background probe traffic |
| `health-check-timeout-secs` | `2` | How long Flow waits for a probe response before counting it as a failure. Should be less than `health-check-interval-secs` |
| `unhealthy-threshold` | `3` | Consecutive probe failures before a backend is removed from the pool |
| `healthy-threshold` | `2` | Consecutive probe successes before a removed backend is returned to the pool |

### How it works

Flow sends an HTTP GET to `health-check-path` on each backend independently on the
`health-check-interval-secs` schedule. A 2xx response is a success; anything else (non-2xx,
timeout, connection failure) is a failure.

After `unhealthy-threshold` consecutive failures the backend is removed from the pool. After
`healthy-threshold` consecutive successes it is restored.

**Fail-open**: if all backends are marked unhealthy simultaneously, Flow routes requests anyway
rather than returning errors to every client. This prevents a misconfigured probe path from
taking a service offline entirely.

### What active checking does not catch

Active probes go to `health-check-path` on a fixed schedule. A backend that returns 200 on
`/healthz` but is failing real API requests (overloaded, bad deployment, 5xx on specific
endpoints) will never be removed by active checking alone. Use circuit breaking for that.

### DNS-discovered connectors

Health checking works with both static IP and DNS-discovered connectors. The check runs
against each resolved IP independently. When DNS re-resolves to a new set of addresses,
probing starts fresh for any new address.

---

## Passive health checking

Passive health checking observes real request outcomes — no synthetic probes are sent. It is
implemented by circuit breaking. When a backend's error rate exceeds a configured threshold,
its circuit trips and it is removed from selection until it recovers.

```kdl
load-balance {
    selection "RoundRobin"
    circuit-breaker {
        error-threshold-pct 50   // trip at 50% errors
        window-secs 60           // over a 60-second window
        min-requests 5           // require at least 5 requests before tripping
        cooldown-secs 30         // stay tripped for 30s before sending a probe
        probe-requests 1         // probes to send before closing or reopening
    }
}
```

See [Circuit breaker](circuit-breaker.md) for the full reference including the state machine,
window strategy options, and operational guidance.

### Default behavior

By default, `health-check "None"` is in effect and no `circuit-breaker` is configured. Flow
discovers failures only when a real request fails and returns the error to the client
immediately — no automatic backend removal.

This is acceptable when:
- Your orchestrator (Kubernetes, an upstream load balancer) handles backend removal
- Retry/failover logic lives in the client or an upstream layer
- You want zero background overhead in Flow itself

For automatic failure detection inside Flow, add `circuit-breaker { }`, `health-check "Active"`,
or both.

---

## Related

- [Circuit breaker](circuit-breaker.md) — full circuit breaking reference (state machine, window strategies, operational guidance)
- [Design Decisions → How should I handle backend failures?](../design-decisions.md#how-should-i-handle-backend-failures)
- [Policies](../policies.md)
