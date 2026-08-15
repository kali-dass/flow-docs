# Flow vs KrakenD CE

A throughput, latency, and memory comparison of Flow and KrakenD Community Edition, measured
on identical hardware with an identical workload.

**TL;DR**
- On like-for-like HTTP/1.1, Flow leads both configurations — 1.20× with TLS, 1.40× without.
  All Flow figures are 5-minute sustained-confirmed.
- On the two apples-to-apples HTTP/2-on-the-client-hop comparisons, Flow leads **1.60×** (TLS)
  and **1.98×** (cleartext) — because KrakenD CE cannot speak HTTP/2 to the backend, so it never
  gets the multiplexing benefit on that hop that Flow does.
- Flow's genuine end-to-end HTTP/2 configurations (20,000 TLS, 28,000 cleartext h2c) have **no
  KrakenD equivalent at all** — a hard capability gap, not a ratio. KrakenD CE cannot speak
  HTTP/2 to a backend under any configuration.
- Flow's memory footprint is roughly 3–4× smaller at every point measured.
- **KrakenD's ceilings are all from 60-second runs — see [Why sustained, not
  burst](#why-sustained-not-burst).** All six of Flow's own configurations here now have
  5-minute-or-longer sustained confirmation and matched their burst numbers closely; KrakenD's
  do not yet have the equivalent test.

| | |
|---|---|
| **Tested** | KrakenD CE v2.13 |
| **Hardware** | 2 vCPU gateway, private networking |
| **Workload** | one passthrough route to an HTTP echo backend returning a 14-byte body |
| **Load pattern** | fixed-rate, 60-second runs, swept upward until delivery fell short |

This is an engineering measurement of **proxy throughput and latency**, not a feature
comparison. See [What this measures](#what-this-measures--and-what-it-does-not) at the end —
and if a term below (p99, sustained ceiling, h2c, ...) is unfamiliar, see the
[glossary](glossary.md).

Every figure below comes with the exact commands to reproduce it yourself — see
[How to reproduce](#how-to-reproduce). Nothing here is inferred from a vendor's own benchmarks.

---

> ## ⚠️ What "HTTP/2" means for each product
>
> **KrakenD CE cannot speak HTTP/2 to the backend at all — this document no longer blends that
> into a comparison row.** HTTP/2 appears in two clearly separate rows below:
>
> - **HTTP/2 end-to-end (client and backend)** — a Flow-only configuration. KrakenD CE is marked
>   **Not supported**, not given a substitute number.
> - **HTTP/2 (client) + HTTP/1.1 (backend)**, explicitly labeled that way — the configuration
>   KrakenD CE actually runs when a client requests HTTP/2. Flow was tested in the identical
>   hop shape for both the TLS and cleartext variants, making both rows a genuine like-for-like
>   comparison — see the throughput table.

---

## The six configurations

TLS is present on **both** hops in the TLS rows, and absent from both hops in the cleartext
rows. The only other variable is HTTP version.

| Configuration | Flow | KrakenD CE |
|---|---|---|
| TLS + HTTP/1.1 | TLS-H1 → TLS-H1 | TLS-H1 → TLS-H1 |
| Cleartext HTTP/1.1 | H1 → H1 | H1 → H1 |
| HTTP/2 end-to-end (TLS) | TLS-H2 → TLS-H2 | **Not supported** |
| TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend) | TLS-H2 → TLS-H1 | TLS-H2 → TLS-H1 |
| HTTP/2 end-to-end (cleartext, h2c) | h2c → h2c | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext | h2c → H1 | h2c → H1 |

### Which configuration is like mine?

- **Terminate TLS at the gateway, plaintext to internal backends** → the **TLS+HTTP/1.1** row
  (if your clients are HTTP/1.1) or **TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend)** row (if
  they negotiate HTTP/2, e.g. modern browsers or gRPC-Web, but your backend is HTTP/1.1) is
  your best proxy.
- **No TLS anywhere — internal-only gateway, or TLS terminated upstream** → **cleartext
  HTTP/1.1**, the closest comparison in this document (no encryption cost, no multiplexing gap).
- **HTTP/2 end to end, cleartext internal network** → **HTTP/2 end-to-end (cleartext, h2c)**.
  KrakenD CE cannot serve this shape at all — it has no HTTP/2 backend option, full stop, not a
  slower version of it.

## Throughput

Sustainable rate — the highest offered rate at which the gateway delivered 100% of requests.

| Configuration | **Flow** | **KrakenD CE** | Ratio |
|---|---|---|---|
| TLS + HTTP/1.1 (both hops) | **14,300** | 11,900 | **1.20×** |
| Cleartext HTTP/1.1 (both hops) | **18,500** | 13,200 | **1.40×** |
| HTTP/2 end-to-end (TLS) | **20,000** | **Not supported** | — |
| TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend) | **17,000** | 10,600 | **1.60×** |
| HTTP/2 end-to-end (cleartext, h2c) | **28,000** | **Not supported** | — |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext | **22,700** | 10,800 | **2.10×** |
| **Best configuration of each** | **28,000** | **13,200** | **2.12×** |

All of Flow's figures are 5-minute sustained-confirmed — see [Flow's own sustained-run
sweep](../performance.md) for the underlying data. The TLS+HTTP/2 (client) + TLS+HTTP/1.1
(backend) row needs no capability caveat, since both products are terminating HTTP/2 at the
gateway and speaking HTTP/1.1 to the backend — a genuine apples-to-apples result. Its cleartext
counterpart (h2c client + HTTP/1.1 backend) is also apples-to-apples for the same reason. The two end-to-end HTTP/2 rows have no KrakenD number
because KrakenD CE cannot speak HTTP/2 to a backend under any configuration — that's a hard
capability gap, not a slower result.

```
Sustained req/s — longer bar is better

TLS+HTTP/1.1        Flow    ████████████████████████ 14,300
                    KrakenD ████████████████████     11,900

Cleartext H1/H1     Flow    ███████████████████████████████ 18,500
                    KrakenD ██████████████████████          13,200

HTTP/2 end-to-end   Flow    ██████████████████████████████████ 20,000
(TLS)               KrakenD Not supported

TLS+HTTP/2→H1.1     Flow    █████████████████████████████ 17,000
                    KrakenD ██████████████████             10,600

HTTP/2 end-to-end   Flow    ████████████████████████████████████████████████ 28,000
(cleartext, h2c)    KrakenD Not supported

h2c-client→H1.1     Flow    ███████████████████████████████████████ 22,700
                    KrakenD ██████████████████                    10,800
```

Three things to read from this table:

- **On like-for-like HTTP/1.1, Flow leads both configurations** — 1.20× with TLS, 1.40×
  without.
- **On the two apples-to-apples HTTP/2-on-the-client-hop comparisons** — Flow leads 1.60× with
  TLS, 1.98× cleartext. Both are genuine like-for-like: both products terminate HTTP/2 at the
  gateway and speak HTTP/1.1 to the backend.
- Flow's two end-to-end HTTP/2 rows (20,000 and 28,000) have no KrakenD equivalent at all:
  KrakenD CE cannot speak HTTP/2 to a backend, so those numbers describe what Flow alone is
  capable of, not a ratio.

Each product's **best** configuration is different: Flow's is cleartext HTTP/2 end-to-end
(28,000); KrakenD's is cleartext HTTP/1.1 (13,200), since HTTP/2 isn't available to KrakenD's
backend hop at all.

## Latency

### At each product's own ceiling — every comparable configuration

Each row uses both products' own ceiling for that configuration (see the throughput table
above), not a shared rate. **Flow's figures are 5-minute sustained; KrakenD's are 60-second
bursts only** — see [Why sustained, not burst](#why-sustained-not-burst) — so treat the KrakenD
side of every row as provisional. The row nearest each stated ceiling from KrakenD's own rate
sweep is used where the exact ceiling rate wasn't itself a tested point.

| Configuration | Product | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|
| TLS + HTTP/1.1 (Flow 14,300 / KrakenD 11,900) | **Flow** | **446 µs** | **5.90 ms** | **4.93 ms** | **10.94 ms** | **12.41 ms** | **15.48 ms** | **188.9 ms** |
| TLS + HTTP/1.1 (Flow 14,300 / KrakenD 11,900) | KrakenD | 448 µs | 12.53 ms | 10.95 ms | 21.44 ms | 24.60 ms | 31.86 ms | 432.2 ms |
| Cleartext H1/H1 (Flow 18,500 / KrakenD 13,200) | **Flow** | 433 µs | 6.91 ms | 6.83 ms | **10.10 ms** | **11.10 ms** | **13.41 ms** | **31.3 ms** |
| Cleartext H1/H1 (Flow 18,500 / KrakenD 13,200) | KrakenD | **283 µs** | **5.74 ms** | **4.40 ms** | 12.74 ms | 15.62 ms | 20.22 ms | 50.5 ms |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / KrakenD 10,600) | **Flow** | **620 µs** | **6.00 ms** | **5.50 ms** | **9.40 ms** | **10.86 ms** | **14.16 ms** | **92.4 ms** |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / KrakenD 10,600) | KrakenD | 864 µs | 13.76 ms | 13.11 ms | 22.26 ms | 25.29 ms | 29.79 ms | 345.4 ms |
| h2c (client) → H1 (backend) (Flow 22,700 / KrakenD 10,800) | **Flow** | **447 µs** | **7.16 ms** | **6.87 ms** | **11.34 ms** | **12.32 ms** | **14.05 ms** | **30.9 ms** |
| h2c (client) → H1 (backend) (Flow 22,700 / KrakenD 10,800) | KrakenD | 761 µs | 13.74 ms | 13.14 ms | 22.05 ms | 24.89 ms | 29.38 ms | 66.4 ms |

On three of the four rows, Flow's tail (p95/p99/max) stays well tighter than KrakenD's even
though Flow is absorbing a higher rate — clearest on the two HTTP/2 rows, where Flow's p99 is
under half of KrakenD's at 46–65% higher throughput. Cleartext HTTP/1.1 is the exception:
KrakenD actually runs faster at min/mean/p50 there (it's absorbing a lower rate at a lower
ceiling), but the gap flips at the tail — Flow's p90/p95/p99/max are all lower despite the
higher load.

```text
p99 latency at each product's own ceiling (ms)

TLS+HTTP/1.1    Flow    14,300 rps ███████████████████                      15.48 ms
                KrakenD 11,900 rps ████████████████████████████████████████ 31.86 ms

Cleartext H1/H1 Flow    18,500 rps █████████████████                        13.41 ms
                KrakenD 13,200 rps █████████████████████████                20.22 ms

TLS-H2 → H1     Flow    17,000 rps ██████████████████                       14.16 ms
                KrakenD 10,600 rps ███████████████████████████████████████  29.79 ms

h2c → H1        Flow    22,700 rps █████████████████                        14.05 ms
                KrakenD 10,800 rps ███████████████████████████████████████  29.38 ms
```

### Cleartext HTTP/1.1 sustained — the cleanest single comparison

Same protocol, no encryption on either side, both at their own ceiling — Flow 18,500 req/s
(5-minute sustained), KrakenD 13,200 req/s (60-second, provisional; see [Why sustained, not
burst](#why-sustained-not-burst)). This is the same row as in the table above, reproduced here
with its own writeup since it's the one configuration with no encryption cost and no HTTP/2
capability gap.

| | **Flow** | **KrakenD CE** |
|---|---|---|
| Min | 433 µs | **283 µs** |
| Mean | **6.91 ms** | 5.74 ms |
| p50 | **6.83 ms** | 4.40 ms |
| p90 | **10.10 ms** | 12.74 ms |
| p95 | **11.10 ms** | 15.62 ms |
| p99 | **13.41 ms** | 20.22 ms |
| Max | **31.3 ms** | 50.5 ms |

KrakenD is nominally faster at min/mean/p50 — it's absorbing roughly 5,300 fewer req/s at its
own provisional ceiling — but Flow leads from p90 onward despite the higher load. KrakenD's
figures here are also still a 60-second burst, unlike Flow's 5-minute sustained ones.

```text
p99 latency, cleartext HTTP/1.1 sustained (ms)

Flow    18,500 rps ██████████████████████████                13.41 ms
KrakenD 13,200 rps ████████████████████████████████████████  20.22 ms
```

### Latency distribution shape

At each product's own ceiling (see the table above), the *proportional* spread from p50 to p90
is similar for both products — roughly 2× on TLS + HTTP/1.1 (Flow 4.93 ms → 10.94 ms; KrakenD
10.95 ms → 21.44 ms). What differs is the floor: KrakenD's p50 already starts more than double
Flow's, and that gap persists rather than closing at the tail. The absolute difference at p99
and max is where the two products diverge most, and that difference is larger than the
throughput difference between them.

## Memory

Resident memory of the gateway process.

| | **Flow** | **KrakenD CE** |
|---|---|---|
| Idle | ~10 MB | ~73 MB |
| Under load (60 s runs) | 20–30 MB | 78–94 MB |
| Under sustained load (10–30 min) | 39–48 MB | — |

```text
Memory under load (MB)

Flow    ████████████                             25 MB
KrakenD ████████████████████████████████████████ 86 MB
```

Memory tracked open connection count in both products and plateaued; neither showed unbounded
growth over the runs measured. Flow's footprint was roughly 3–4× smaller at every point.

## Test setup

| Component | Detail |
|---|---|
| Gateway host | 2 vCPU, 4 GB, dedicated, Linux 7.0.0-1008-gcp |
| Backend | separate host, 4 vCPU, HTTP echo returning a 14-byte body, serving TLS and cleartext |
| Load generator | separate host, 6 vCPU |
| Network | same zone, private addressing, sub-millisecond RTT |
| Load tool | vegeta, fixed rate, 60 s per run |
| KrakenD | CE v2.13, default (JSON) encoding, access log disabled, log level WARNING |
| Flow | one route, one connector, no policies enabled |

All three hosts were otherwise idle. Gateway and backend were unchanged between the two
products — only the gateway process was swapped.

**KrakenD was run in its best-performing configuration.** Its alternative `no-op` passthrough
encoding was also tested and performed substantially worse, so the default JSON encoding is
used for every KrakenD figure in this document.

## Why sustained, not burst

**Every KrakenD ceiling in this document comes from a 60-second run — none has been confirmed
with a multi-minute sustained run.** This isn't a purely theoretical concern: this page's own
[APISIX comparison](apisix.md) found that a 60-second burst overstated one gateway's real
ceiling by more than 50% once GC and connection-pool pressure that only builds up over minutes
was accounted for. KrakenD is written in Go, not the Lua-on-nginx stack that hit that specific
issue, so the same mechanism may not apply — but a different mechanism isn't the same as a
confirmed-safe result.

All **six** of Flow's own configurations in this document (TLS-H1 at 14,300 req/s, TLS+HTTP/2
client + TLS+HTTP/1.1 backend at 17,000 req/s, TLS-H2 end-to-end at 20,000 req/s, cleartext
H1/H1 at 18,500 req/s, h2c-client + HTTP/1.1-backend at 22,700 req/s, and h2c end-to-end at
28,000 req/s) now have 5-minute-or-longer sustained confirmation, and matched their 60-second
numbers closely — which is reassuring for Flow's figures specifically, but doesn't say anything
about KrakenD's untested configurations. Treat every KrakenD ceiling and ratio on this page as
provisional pending a sustained-run pass.

## How to reproduce

### KrakenD configuration

TLS configuration (used for both TLS rows; the client's protocol selects H1 or H2):

```json
{
  "version": 3,
  "debug_endpoint": false,
  "echo_endpoint": false,
  "client_tls": { "allow_insecure_connections": true },
  "tls": { "keys": [{ "public_key": "/path/test.crt", "private_key": "/path/test.key" }] },
  "endpoints": [{
    "endpoint": "/echo", "method": "GET",
    "backend": [{ "url_pattern": "/echo", "method": "GET", "host": ["https://BACKEND:3000"] }]
  }],
  "extra_config": {
    "telemetry/logging": { "level": "WARNING", "stdout": false, "syslog": false },
    "router": { "disable_access_log": true }
  }
}
```

Cleartext configuration: remove the `tls` and `client_tls` blocks and change the backend host
to `http://BACKEND:3000`.

### Flow configuration

```kdl
services {
    Gateway {
        listeners {
            "0.0.0.0:8443" cert-path="/path/test.crt" key-path="/path/test.key" offer-h2=true
        }
        connectors {
            "BACKEND:3000" tls-sni="backend" proto="h2-only" verify-cert=false
        }
        routes {
            route {
                match { path-prefix "/echo" }
                allowed-methods "GET"
                allowed-protocols "https" "h2"
            }
        }
    }
}
```

For the other configurations: drop `offer-h2` for HTTP/1.1 from clients; use `proto="h1-only"`
for an HTTP/1.1 backend hop; remove `cert-path`/`key-path` and `tls-sni` (adding
`allow-h2c=true` for h2c) for the cleartext rows.

### Load commands

```bash
# TLS + HTTP/1.1
echo 'GET https://GATEWAY:PORT/echo' | vegeta attack -http2=false \
  -rate RATE -duration 60s -workers 100 -max-workers 300 -insecure

# TLS + HTTP/2
echo 'GET https://GATEWAY:PORT/echo' | vegeta attack -http2=true \
  -rate RATE -duration 60s -workers 100 -max-workers 300 -insecure

# Cleartext HTTP/1.1
echo 'GET http://GATEWAY:PORT/echo' | vegeta attack -http2=false \
  -rate RATE -duration 60s -workers 100 -max-workers 300

# Cleartext HTTP/2 (h2c)
echo 'GET http://GATEWAY:PORT/echo' | vegeta attack -h2c \
  -rate RATE -duration 60s -workers 100 -max-workers 300
```

Sweep `RATE` upward until the delivered rate falls below the offered rate. Confirm the protocol
actually in use by counting connections on each side — an HTTP/2 client hop shows single-digit
connections, HTTP/1.1 shows roughly one per load-generator worker:

```bash
ss -tn state established '( sport = :GATEWAY_PORT )' | wc -l   # client side
ss -tn state established '( dport = :3000 )' | wc -l           # backend side
```

### A note on load-generator concurrency

Use enough workers that the load generator is never the limit. Required concurrency is
`rate × latency` — at 13,000 req/s and 12 ms that is ~156 concurrent, so a 150-worker cap would
bind. Where a run appears to plateau, check this before recording it as a ceiling.

## What this measures — and what it does not

**It measures** the throughput, latency, and memory of each gateway proxying a request to a
backend, on one hardware class, with a trivial endpoint and a small response body.

**It does not measure:**

- **Feature parity.** KrakenD is a full API gateway with response aggregation, transformation,
  and a plugin ecosystem. None of that is exercised here — this is a passthrough route.
- **Behaviour under other payload sizes**, request mixes, or routing complexity. A 14-byte
  response maximizes the relative weight of per-request overhead; larger bodies would shift the
  balance toward I/O.
- **Performance on other hardware.** A 2 vCPU host is small; results may not scale linearly.
- **Anything about KrakenD Enterprise**, which was not tested.

Figures are specific to KrakenD CE v2.13 as of the test dates above. Both products change over
time; re-run the commands above to check current behaviour.

For Flow's own performance characteristics across configurations, independent of any
comparison, see [Performance](../performance.md). See also the [glossary](glossary.md) and
[comparisons overview](README.md).
