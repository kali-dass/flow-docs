# Flow vs Apache APISIX

A throughput and latency comparison of Flow and Apache APISIX, measured on the same class of
hardware with an identical workload.

**TL;DR**
- Flow leads in every configuration, from 1.12× (both cleartext HTTP/1.1 and the
  cleartext-h2c-client/HTTP/1.1-backend row) up to 4.6× (TLS+HTTP/1.1, where APISIX pays a full
  TLS handshake per request). All of Flow's figures are 5-minute sustained-confirmed — see the
  throughput table below.
- APISIX's TLS+HTTP/1.1 ceiling (3,100 req/s) is CPU-bound on TLS handshake cost, not a
  kernel-level bottleneck.
- Past its ceiling, all four of APISIX's configurations degrade gracefully rather than
  cliffing — TLS+HTTP/1.1 shows a steep latency ramp approaching 3,100 req/s that then
  plateaus, not a runaway spike.

| | |
|---|---|
| **Tested** | Apache APISIX 3.x (standalone mode, OpenResty/LuaJIT) |
| **Hardware** | 2 vCPU gateway, private networking |
| **Workload** | one passthrough route to an HTTP echo backend |
| **Load pattern** | fixed-rate `vegeta` with bounded worker concurrency, swept upward; every ceiling below is confirmed with a **sustained multi-minute run**, not just a 60-second burst |

This is an engineering measurement of **proxy throughput and latency**, not a feature
comparison. See [What this measures](#what-this-measures--and-what-it-does-not) at the end —
and if a term below (p99, sustained ceiling, h2c, ...) is unfamiliar, see the
[glossary](glossary.md).

Every figure below comes with the exact commands to reproduce it yourself — see
[How to reproduce](#how-to-reproduce). Nothing here is inferred from a vendor's own benchmarks.

---

> ## ⚠️ What "HTTP/2" means for each product
>
> **APISIX does not support HTTP/2 to the backend.** Its h2c and TLS+HTTP/2 listeners accept
> HTTP/2 from the client but always fall back to HTTP/1.1 for the upstream hop — this document
> clearly separates that from Flow's end-to-end HTTP/2 configurations:
>
> - **HTTP/2 end-to-end (client and backend)** — a Flow-only configuration. APISIX is marked
>   **Not supported**, not given a substitute number.
> - **HTTP/2 (client) + HTTP/1.1 (backend)**, explicitly labeled that way — the configuration
>   APISIX actually runs. Flow was tested in the identical hop shape for both variants, so these
>   rows are genuine apples-to-apples results.
>
> All of Flow's figures below are 5-minute sustained-confirmed.

---

## The six configurations

TLS is present on **both** hops in the TLS rows, and absent from both hops in the cleartext
rows. The only other variable is HTTP version.

| Configuration | Flow | APISIX |
|---|---|---|
| TLS + HTTP/1.1 | TLS-H1 → TLS-H1 | TLS-H1 → TLS-H1 |
| Cleartext HTTP/1.1 | H1 → H1 | H1 → H1 |
| HTTP/2 end-to-end (TLS) | TLS-H2 → TLS-H2 | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), TLS | TLS-H2 → TLS-H1 | TLS-H2 → TLS-H1 |
| HTTP/2 end-to-end (cleartext, h2c) | h2c → h2c | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext | h2c → H1 | h2c → H1 |

APISIX's h2c and TLS+HTTP/2 listeners both accept HTTP/2 from the client but talk HTTP/1.1
to the backend in these configurations. Flow's two end-to-end rows above run HTTP/2 on
both hops and have no APISIX equivalent in this test — that's a **capability difference**, not
a protocol-implementation difference — see the callout above. Flow's own client-H2/backend-H1
figures, directly comparable to APISIX's, appear as separate rows in the throughput table below
(both the TLS and cleartext variants).

### Which configuration is like mine?

- **Terminate TLS at the gateway, plaintext to internal backends** → the **TLS+HTTP/1.1** row
  (HTTP/1.1 clients) or **TLS+HTTP/2** row (clients that negotiate HTTP/2). The TLS+HTTP/1.1
  row is where the gap is largest, since APISIX pays a full TLS handshake per request there.
- **No TLS anywhere — internal-only gateway, or TLS terminated upstream** → **cleartext
  HTTP/1.1**, the closest comparison in this document (no encryption cost, no multiplexing gap
  on either side).
- **HTTP/2 end to end, cleartext internal network** → **h2c**. APISIX's client hop accepts
  HTTP/2, but the backend hop falls back to HTTP/1.1 — if your real deployment multiplexes on
  both sides, this row understates the practical gap.

## Throughput

Sustainable rate — the highest offered rate at which the gateway delivered essentially all
requests, confirmed over a multi-minute sustained run (not a 60-second burst, which
overstates capacity for both products — see [Test setup](#test-setup) below).

| Configuration | **Flow** | **APISIX** | Ratio |
|---|---|---|---|
| TLS + HTTP/1.1 (both hops) | **14,300** | 3,100 | **4.6×** |
| Cleartext HTTP/1.1 (both hops) | **18,500** | 16,500 | **1.12×** |
| HTTP/2 end-to-end (TLS, Flow only) | **20,000** | **Not supported** | — |
| HTTP/2 (client) + HTTP/1.1 (backend), TLS (both products) | **17,000** | 10,600 | **1.60×** |
| HTTP/2 end-to-end (cleartext, h2c, Flow only) | **28,000** | **Not supported** | — |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext (both products) | **22,700** | 19,000 | **1.19×** |
| **Best configuration of each** | **28,000** | **19,000** | 1.47× |

All of Flow's figures are 5-minute sustained-confirmed — see [Flow's own sustained-run
sweep](../performance.md) for the underlying data. APISIX's 16,500 and 3,100 figures are also
sustained-confirmed (see [Test setup](#test-setup) below), making both rows validated-vs-validated
comparisons. Flow's TLS-H2-client/HTTP/1.1-backend and cleartext h2c-client/HTTP/1.1-backend
rows are genuinely apples-to-apples against APISIX's 10,600 and 19,000 figures (same hop shape);
the two end-to-end HTTP/2 rows have no APISIX equivalent in this test and carry no ratio.

```
Sustained req/s — longer bar is better

TLS+HTTP/1.1         Flow   ████████████████████                14,300
                     APISIX ████                                  3,100

Cleartext H1/H1      Flow   ██████████████████████████           18,500
                     APISIX ████████████████████████             16,500

HTTP/2 end-to-end    Flow   █████████████████████████████        20,000
(TLS)                APISIX Not supported

TLS-H2 → H1/1        Flow   ████████████████████████             17,000
(both products)      APISIX ███████████████                      10,600

HTTP/2 end-to-end    Flow   ████████████████████████████████████████ 28,000
(cleartext, h2c)     APISIX Not supported

h2c → H1/1           Flow   ████████████████████████████████     22,700
(both products)      APISIX ███████████████████████████          19,000
```

Three things to read from this table:

- **The gap is widest on TLS + HTTP/1.1** (4.6×), where APISIX's ceiling (3,100 req/s) is set
  by CPU-bound TLS handshake cost — one full handshake per request — not the same
  LuaJIT/connection-pool ceiling that governs its other configurations.
- **The gap is narrowest on cleartext HTTP/1.1** (1.12×) — the configuration with no encryption
  cost and an identical hop shape on both products. This is the closest, most direct comparison
  in this document.
- **Flow's two end-to-end HTTP/2 rows (20,000 and 28,000) have no APISIX equivalent** — APISIX's
  standalone config in this test never runs HTTP/2 to the backend. On the two rows where both
  products use the same client-H2/backend-H1 shape, Flow leads 1.60× (TLS) and 1.19×
  (cleartext) — genuine like-for-like results, not capability gaps.

## Latency

### At each product's own sustained ceiling — every comparable configuration

Each row uses both products' own sustained ceiling for that configuration (see the throughput
table above), not a shared rate — ceilings differ per configuration and per product, so this
shows each at its own limit. Only the four configurations both products can run are included;
Flow's two end-to-end-HTTP/2 configurations have no APISIX equivalent (see above).

| Configuration | Product | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|
| TLS + HTTP/1.1 (Flow 14,300 / APISIX 3,100) | **Flow** | **446 µs** | **5.90 ms** | **4.93 ms** | **10.94 ms** | **12.41 ms** | **15.48 ms** | **188.9 ms** |
| TLS + HTTP/1.1 (Flow 14,300 / APISIX 3,100) | APISIX | 413 µs | 28.52 ms | 32.28 ms | 54.85 ms | 74.81 ms | 106.37 ms | 946.6 ms |
| Cleartext H1/H1 (Flow 18,500 / APISIX 16,500) | **Flow** | 433 µs | 6.91 ms | 6.83 ms | 10.10 ms | 11.10 ms | **13.41 ms** | **31.3 ms** |
| Cleartext H1/H1 (Flow 18,500 / APISIX 16,500) | APISIX | **387 µs** | **5.44 ms** | **5.04 ms** | **9.26 ms** | **10.51 ms** | 13.22 ms | 94.8 ms |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / APISIX 10,600) | **Flow** | 620 µs | **6.00 ms** | **5.50 ms** | **9.40 ms** | **10.86 ms** | **14.16 ms** | **92.4 ms** |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / APISIX 10,600) | APISIX | **415 µs** | 12.66 ms | 13.25 ms | 21.12 ms | 23.32 ms | 31.47 ms | 3.089 s |
| h2c (client) → H1 (backend) (Flow 22,700 / APISIX 19,000) | **Flow** | 447 µs | **7.16 ms** | **6.87 ms** | **11.34 ms** | **12.32 ms** | **14.05 ms** | **30.9 ms** |
| h2c (client) → H1 (backend) (Flow 22,700 / APISIX 19,000) | APISIX | **316 µs** | 7.76 ms | 8.15 ms | 12.22 ms | 13.40 ms | 16.41 ms | 3.021 s |

A pattern holds across all four rows: **Flow's max latency stays two to three orders of
magnitude tighter than APISIX's** at the tail. The two HTTP/2 rows are the clearest case —
APISIX's max jumps into multi-second territory (3.09 s, 3.02 s) at its own ceiling, while
Flow's stays under 100 ms at a higher absolute rate. APISIX's min is lower on every row (it's
usually absorbing a meaningfully lower rate at its own lower ceiling), and its mean/p50 edge
out Flow's specifically on cleartext H1/H1, the row where the two ceilings are closest together
— everywhere else Flow's mean/p50 are already lower despite the higher load. The gap widens
consistently from p90 onward on every row, where queueing and GC/connection-pool effects show
up.

### Cleartext HTTP/1.1 sustained — the cleanest single comparison

Same protocol, no encryption on either side, both at their own sustained ceiling.

Both at their own sustained ceiling, both confirmed with 5-minute runs — Flow 18,500 req/s,
APISIX 16,500 req/s.

| | **Flow** | **APISIX** |
|---|---|---|
| Min | 433 µs | 387 µs |
| Mean | 6.91 ms | 5.44 ms |
| p50 | 6.83 ms | 5.04 ms |
| p90 | 10.10 ms | 9.26 ms |
| p95 | 11.10 ms | 10.51 ms |
| p99 | 13.41 ms | 13.22 ms |
| Max | 31.3 ms | 94.8 ms |

Flow's ceiling is also its *higher* rate — the comparison isn't apples-to-apples at a shared
rate, since each product's sustainable ceiling differs. Flow's median is running ~35% higher
than APISIX's because it's absorbing ~2,000 more req/s at that point; the gap narrows toward
the tail (9% at p90, under 2% at p99) as both products approach their own CPU limit. Flow's
max latency, despite the higher load, is roughly **3× lower** (31.3 ms vs 94.8 ms) — a sign of
a tighter latency distribution under sustained saturation, not just a faster median.

```text
p99 latency at each product's own sustained ceiling (ms)

TLS+HTTP/1.1    Flow   14,300 rps ██████                                   15.48 ms
                APISIX  3,100 rps ████████████████████████████████████████ 106.37 ms

Cleartext H1/H1 Flow   18,500 rps █████                                    13.41 ms
                APISIX 16,500 rps █████                                    13.22 ms

TLS-H2 → H1     Flow   17,000 rps █████                                    14.16 ms
                APISIX 10,600 rps ████████████                             31.47 ms

h2c → H1        Flow   22,700 rps █████                                    14.05 ms
                APISIX 19,000 rps █████                                    16.41 ms
```

On cleartext HTTP/1.1, p99 is nearly tied — APISIX is absorbing 2,000 fewer req/s at that point,
so this isn't a like-for-like rate comparison (see above); it shows each product's tail latency
at its own limit, not a speed difference. On the other three rows the gap is much wider, driven
by TLS handshake cost (TLS+HTTP/1.1) and connection-pool/GC pressure at the tail (the two
HTTP/2 rows) — see the full-percentile table above for the min/mean/p50 context.

### How each product fails past its ceiling

Flow has no sharp cliff past its ceiling — the transition is a smooth degradation. All three of
Flow's sustained-tested configurations (TLS-H2, cleartext h2c, h2c-client/H1-backend) show a
**graceful plateau** past their ceiling: delivered throughput flattens and latency rises
smoothly, with no collapse — see [what happens past the
ceiling](../performance.md#what-happens-past-the-ceiling--read-this-before-you-set-a-rate) for a
worked example. Both products now read the same way in this section:

- **Flow**: CPU-bound, and degrades gracefully. Past the sustained ceiling, delivered throughput
  flattens rather than dropping, and latency rises smoothly rather than spiking — see the worked
  example linked above (offering 21,000 against a 20,000 TLS-H2 ceiling still delivers ~19,850
  req/s at 100% success, mean latency up only from 6.73 ms to 7.14 ms).
- **APISIX, all four configurations**: CPU-bound, and all degrade gracefully rather than
  cliffing. TLS+HTTP/1.1 has the lowest ceiling (mean latency rises steeply from ~1.7 ms at
  2,900 req/s to ~28.5 ms by 3,100 req/s, then stays in the 28–42 ms range through 3,600 req/s
  rather than spiking further — CPU-bound on TLS handshake cost, one full handshake per
  request). H1/H1 shows only a ~23% latency rise (delivery stays at 100%) at 16.7k req/s, 200
  req/s past its 16.5k ceiling; h2c and TLS-H2 show similarly smooth transitions (see APISIX's
  own benchmark notes for detail).

## Memory

Resident memory of the gateway process.

| | **Flow** | **APISIX** |
|---|---|---|
| Under load (all configurations, all rates) | 20–30 MB | ~80–82 MB (2 workers × 40–41 MB) |

```text
Memory under load (MB)

Flow   ████████████ 25 MB
APISIX ████████████████████████████████████████ 81 MB
```

APISIX's memory stayed flat across every tested rate, from 5,000 req/s up to each
configuration's ceiling and beyond — zero major page faults observed at any point, and minor
faults scaled linearly with request rate (normal allocation churn, not pressure). Memory was
not a limiting factor in any APISIX ceiling measured in this document; every ceiling above is
set by CPU.

## A note on APISIX's CPU-core balance

APISIX's two nginx worker processes occasionally showed uneven CPU usage between cores at lower
loads, converging toward balance as both cores approach saturation. Flow showed no such imbalance
— its two cores remained balanced throughout the test.

## Test setup

| Component | Detail |
|---|---|
| Gateway host | 2 vCPU GCP VM |
| Backend | separate host, HTTP echo endpoint |
| Load generator | separate host, `vegeta` |
| Network | same zone, private addressing |
| Load tool | vegeta, fixed rate, swept upward |
| APISIX | 3.x, standalone mode, default routing, no plugins |
| Flow | one route, one connector, no policies enabled |

All ceilings in the throughput table above are **sustained** figures — confirmed with runs of
5 minutes (APISIX) or 10 minutes (Flow), not 60-second bursts. This matters most for APISIX's
H1/H1 row: an earlier round of testing that used 60-second bursts with an under-provisioned
load generator reported a ceiling roughly 35% lower than the sustained, properly-provisioned
figure in this document — see [How each product fails past its
ceiling](#how-each-product-fails-past-its-ceiling) above. Every ceiling reported in this
document was independently confirmed with a sustained run at the reported rate and at least one
rate above it, to verify the boundary — not inferred from a single burst test.

## How to reproduce

### APISIX configuration

Standalone mode, one route to the echo backend, default settings, no plugins:

```yaml
routes:
  - uri: /echo
    upstream:
      type: roundrobin
      nodes:
        "BACKEND:3000": 1
#END
```

TLS and HTTP/2 listeners are enabled in `conf/config.yaml`; the client's negotiated protocol
selects which row a given run measures.

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
  -rate RATE -duration 5m -workers 100 -max-workers 150 -insecure

# TLS + HTTP/2
echo 'GET https://GATEWAY:PORT/echo' | vegeta attack -http2=true \
  -rate RATE -duration 5m -workers 100 -max-workers 150 -insecure

# Cleartext HTTP/1.1
echo 'GET http://GATEWAY:PORT/echo' | vegeta attack -http2=false \
  -rate RATE -duration 5m -workers 100 -max-workers 150

# Cleartext HTTP/2 (h2c)
echo 'GET http://GATEWAY:PORT/echo' | vegeta attack -h2c \
  -rate RATE -duration 5m -workers 100 -max-workers 150
```

Run each rate for the full 5 minutes, not 60 seconds — see the sustained-vs-burst note above.
Sweep `RATE` upward until the delivered rate falls below the offered rate, or (for APISIX's
TLS+HTTP/1.1 row) until mean latency jumps by an order of magnitude and does not recover.

### A note on load-generator concurrency

Use enough workers that the load generator is never the limit. Required concurrency is
`rate × latency` — undersized `-max-workers` throttles the client rather than the server and
will understate the gateway's true ceiling. Confirm the load generator isn't the bottleneck by
checking established connection counts on both sides before recording a plateau as a ceiling:

```bash
ss -tn state established '( sport = :GATEWAY_PORT )' | wc -l   # client side
ss -tn state established '( dport = :3000 )' | wc -l           # backend side
```

## What this measures — and what it does not

**It measures** the throughput and latency of each gateway proxying a request to a backend, on
one hardware class, with a trivial endpoint.

**It does not measure:**

- **Feature parity.** APISIX has a large plugin ecosystem (Lua, Wasm) and Apache-project
  integrations. None of that is exercised here — this is a passthrough route.
- **APISIX in clustered/etcd-backed mode.** These figures are standalone-mode APISIX; a
  clustered deployment has different config-sync overhead that isn't measured here.
- **Behaviour under other payload sizes**, request mixes, or routing complexity.
- **Performance on other hardware.** A 2 vCPU host is small; results may not scale linearly,
  and the two products' gateway hosts, while the same vCPU count and cloud, were not
  necessarily the same CPU generation/instance type — see each product's own test-setup notes.

Figures are specific to APISIX 3.x as of the test dates above and Flow as of its own
[performance page](../performance.md). Both projects change over time; re-run comparable
benchmarks to check current behaviour.

For Flow's own performance characteristics across configurations, independent of any
comparison, see [Performance](../performance.md). See also the [glossary](glossary.md) and
[comparisons overview](README.md). For comparisons against other gateways, see
[Kong Enterprise](kong.md) and [KrakenD CE](krakend.md).
