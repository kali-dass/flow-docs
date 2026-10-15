# Circuit breaker

Passively detects failing backends on real request outcomes and temporarily removes them from
the pool, giving them time to recover before traffic resumes.

**Configuration:** `circuit-breaker` is a block inside `load-balance` on a connector — not a
`path-control` filter, and not a route-level block.

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
        circuit-breaker {
            error-threshold-pct 50      // trip when ≥50% of requests fail
            window-secs 60              // over this rolling window
            min-requests 5              // require at least this many requests before tripping
            cooldown-secs 30            // stay Open for this long before sending a probe
            probe-requests 1            // probes to send in HalfOpen before closing
        }
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
}
```

All fields are optional. Omitting a field uses its default.

## Options

| Field | Default | Description |
|---|---|---|
| `error-threshold-pct` | `50` | Integer 1–100. Trip when this percentage of requests in the window return a 5xx or connection failure |
| `window-secs` | `60` | How far back (in seconds) to look when calculating the error rate |
| `min-requests` | `5` | Minimum requests in the window before the circuit can trip — prevents tripping on the first error at startup or after a long idle period |
| `cooldown-secs` | `30` | How long the circuit stays Open before allowing probe requests |
| `probe-requests` | `1` | Number of requests allowed through in HalfOpen state before closing or reopening |
| `window-strategy` | `"SlidingWindow"` | How outcomes are tracked — `"SlidingWindow"` (exact, default) or `"TimeBuckets"` (fixed-memory, approximate) |
| `bucket-count` | `10` | Number of time buckets. Only valid with `window-strategy "TimeBuckets"`. Must be ≥ 2 |

## How it works

Circuit breaking tracks real request outcomes per backend. When a backend's error rate exceeds
`error-threshold-pct` over the `window-secs` window (and at least `min-requests` have been
seen), its circuit **opens** and requests to that backend fail fast with 502.

After `cooldown-secs` the circuit moves to **half-open**: `probe-requests` requests are allowed
through. A successful probe closes the circuit and normal traffic resumes. Any failure
re-opens the circuit and resets the cooldown.

```
Closed ──[error rate ≥ threshold, min_requests met]──> Open
                                                         │
                                              [cooldown elapsed]
                                                         ▼
                                                      HalfOpen
                                                      │       │
                                            [probe ok] [probe fail]
                                                │             │
                                             Closed          Open
```

**What counts as a failure**: HTTP 5xx responses and connection-level failures (timeout, reset,
refused). 4xx responses are not counted — they represent client errors, not backend failures.

## Fail-open

If all backends on a connector are simultaneously circuit-open, Flow routes requests anyway
rather than returning 502 to every client. This prevents a misconfigured threshold from taking
a service offline entirely. Use `min-requests` to make accidental all-open scenarios less
likely at low traffic.

## Window strategy

Two strategies control how outcomes are tracked:

| Strategy | Memory | Accuracy | Best for |
|---|---|---|---|
| `"SlidingWindow"` (default) | Grows with traffic — ~1 MB per backend at 1,000 req/s with a 60 s window | Exact | Most routes — start here |
| `"TimeBuckets"` | Fixed — O(`bucket-count`) regardless of traffic, a few hundred bytes | ±one slot width | Very high-traffic routes (tens of thousands req/s per backend) where bounded memory matters |

With `"TimeBuckets"`, `bucket-count` sets how many slots divide the window. Example: 10 buckets
over 60 seconds gives 10 × 6-second slots. Finer bucket counts give better accuracy at slightly
higher memory cost.

```kdl
circuit-breaker {
    window-strategy "TimeBuckets"
    bucket-count 10
    window-secs 60
}
```

## Combining with active health checking

Circuit breaking and active health checking catch complementary failure modes:

- **Active health checking** (`health-check "Active"`) — catches backends that are completely
  down. Probes run independently of real traffic, so Flow knows before any client request hits
  a dead backend.
- **Circuit breaking** — catches backends that are alive and pass health probes but are
  returning errors on real traffic (overloaded, partially broken, returning 5xx for specific
  requests).

Using both gives defense in depth: health checking removes dead backends before circuit
breaking even sees them; circuit breaking handles the harder case of a backend that looks
healthy but is failing in production.

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

## Related

- [Health checking](health-checking.md) — active and passive health checking overview; active probing catches completely down backends and complements circuit breaking
- [Design Decisions → How should I handle backend failures?](../design-decisions.md#how-should-i-handle-backend-failures) — comparison of active vs passive failure detection
- [Policies](../policies.md)
