# Flow vs Kong Enterprise

A throughput, latency, and memory comparison of Flow and Kong Enterprise, measured on
identical hardware with an identical workload.

**TL;DR**
- On cleartext HTTP/1.1, the two products are roughly tied — the closest matchup in this
  document, not a clear win for either side.
- Flow's advantage appears once TLS or HTTP/2 is introduced (1.24×–1.56×), and its memory
  footprint is ~20–25× smaller than Kong's under load (20–30 MB vs. ~610–690 MB).
- Kong was tested unlicensed (free-tier feature set); a licensed deployment with plugins
  enabled was not measured.
- **Every Kong figure below is from a 60-second run, not a sustained one — see [Why sustained,
  not burst](#why-sustained-not-burst).** This page's [APISIX comparison](apisix.md) found that
  60-second figures overstated one nginx-based gateway's real ceiling by over 50%; Kong runs on
  the same nginx foundation and has not yet had a sustained-run pass. Flow's own cleartext
  HTTP/1.1 figure, by contrast, *has* been confirmed with a 5-minute sustained sweep (18,500
  req/s, up from an earlier 60-second-only figure of 17,000) — so this comparison is currently
  a validated Flow number against a provisional Kong one. Treat the cleartext-H1.1 ratio as
  provisional pending Kong's own re-test.

| | |
|---|---|
| **Tested** | Kong Enterprise 3.15.0.2, unlicensed (free-tier capability set) |
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
> **Kong cannot speak HTTP/2 to the backend at all — this document no longer blends that into a
> comparison row.** HTTP/2 appears in two clearly separate rows below:
>
> - **HTTP/2 end-to-end (client and backend)** — a Flow-only configuration. Kong is marked
>   **Not supported**, not given a substitute number.
> - **HTTP/2 (client) + HTTP/1.1 (backend)**, explicitly labeled that way — the configuration
>   Kong actually runs when a client requests HTTP/2. Flow was tested in the identical hop
>   shape for the TLS variant, making that row a genuine like-for-like comparison. The
>   cleartext variant of this row is now tested for both — see the throughput table.

---

## The six configurations

TLS is present on **both** hops in the TLS rows, and absent from both hops in the cleartext
rows. The only other variable is HTTP version.

| Configuration | Flow | Kong |
|---|---|---|
| TLS + HTTP/1.1 | TLS-H1 → TLS-H1 | TLS-H1 → TLS-H1 |
| Cleartext HTTP/1.1 | H1 → H1 | H1 → H1 |
| HTTP/2 end-to-end (TLS) | TLS-H2 → TLS-H2 | **Not supported** |
| TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend) | TLS-H2 → TLS-H1 | TLS-H2 → TLS-H1 |
| HTTP/2 end-to-end (cleartext, h2c) | h2c → h2c | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext | h2c → H1 | h2c → H1 |

### Which configuration is like mine?

- **No TLS anywhere — internal-only gateway, or TLS terminated upstream** → **cleartext
  HTTP/1.1**. This is where Kong is most competitive — its measured ceiling roughly matches
  Flow's (within ~3%, and still a provisional 60-second figure on Kong's side).
- **Terminate TLS at the gateway, plaintext to internal backends** → the **TLS+HTTP/1.1** row
  (HTTP/1.1 clients) or **TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend)** row (clients that
  negotiate HTTP/2, but your backend is HTTP/1.1). Flow's lead opens up here.
- **HTTP/2 end to end, cleartext internal network** → **HTTP/2 end-to-end (cleartext, h2c)**.
  Kong cannot serve this shape at all — its backend hop always falls back to HTTP/1.1, full
  stop, not a slower version of end-to-end HTTP/2.

## Throughput

Sustainable rate — the highest offered rate at which the gateway delivered essentially all
requests (≥99.9%).

| Configuration | **Flow** | **Kong** | Ratio |
|---|---|---|---|
| TLS + HTTP/1.1 (both hops) | **14,300** | 11,500 | **1.24×** |
| Cleartext HTTP/1.1 (both hops) | **18,500** (sustained-confirmed) | **~18,000** (provisional, 60s only) | **~1.03× — roughly tied** |
| HTTP/2 end-to-end (TLS) | **20,000** | **Not supported** | — |
| TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend) | **17,000** (sustained) | ~13,000 (provisional) | **~1.31×** |
| HTTP/2 end-to-end (cleartext, h2c) | **28,000** | **Not supported** | — |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext | **22,700** (sustained) | ~17,000 (provisional) | **~1.34×** |
| **Best configuration of each** | **28,000** | **~18,000** | **~1.56×** |

All of Flow's figures are 5-minute sustained-confirmed — see [Flow's own sustained-run sweep](../performance.md) for the underlying data. The TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend) row is the one *confirmed* HTTP/2 comparison here that needs no capability caveat, since both products terminate HTTP/2 at the gateway and speak HTTP/1.1 to the backend. Flow's side is a 5-minute sustained figure; Kong's ~13,000 is still 60-second-burst only, so read the 1.31× ratio as provisional pending Kong's own sustained re-run. The two end-to-end HTTP/2 rows have no Kong number because Kong cannot speak HTTP/2 to a backend under any configuration — a hard capability gap, not a slower result.

```
Sustained req/s — longer bar is better

TLS+HTTP/1.1        Flow ████████████████████            14,300
                    Kong ████████████████                11,500

Cleartext H1/H1     Flow ██████████████████████████       18,500
                    Kong █████████████████████████        ~18,000  (roughly tied)

HTTP/2 end-to-end   Flow ████████████████████████████     20,000
(TLS)               Kong Not supported

TLS+HTTP/2→H1.1     Flow ████████████████████████         17,000
                    Kong ██████████████████               ~13,000

HTTP/2 end-to-end   Flow ████████████████████████████████████████████ 28,000
(cleartext, h2c)    Kong Not supported

h2c-client→H1.1     Flow ███████████████████████████████████ 22,700
                    Kong ████████████████████████         ~17,000
```

Three things to read from this table:

- **On like-for-like cleartext HTTP/1.1, the two products are close enough that neither figure
  should be read as a confident win** — the one configuration with no encryption cost and no
  HTTP/2 capability gap. This is the closest, most direct comparison in this document. Flow's
  18,500 has 5-minute sustained-run confirmation; Kong's ~18,000 is still a 60-second-only
  figure — so today this reads as "roughly tied, pending Kong's own sustained-run pass," not a
  settled result in either direction.
- **On the two confirmed apples-to-apples HTTP/2 comparisons** — TLS+HTTP/2 (client) +
  TLS+HTTP/1.1 (backend), and the new cleartext h2c-client + HTTP/1.1-backend row — Flow leads
  ~1.31× and ~1.26× respectively (both provisional on Kong's side, pending Kong's own
  sustained re-run).
- **Flow's two end-to-end HTTP/2 rows (20,000 and 28,000) have no Kong equivalent at all**:
  Kong cannot speak HTTP/2 to a backend, so those numbers describe what Flow alone is capable
  of, not a ratio.

## Latency

### At each product's own ceiling — every comparable configuration

Each row uses both products' own ceiling for that configuration (see the throughput table
above), not a shared rate. **Flow's figures are 5-minute sustained; Kong's are 60-second
bursts only** — see [Why sustained, not burst](#why-sustained-not-burst) — so treat the Kong
side of every row as provisional. Kong's cleartext H1/H1 row spans two independently
converging test sessions; the range is shown where the two runs didn't land on the exact same
figure.

| Configuration | Product | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|
| TLS + HTTP/1.1 (Flow 14,300 / Kong 11,500) | **Flow** | 446 µs | **5.90 ms** | 4.93 ms | **10.94 ms** | **12.41 ms** | **15.48 ms** | **188.9 ms** |
| TLS + HTTP/1.1 (Flow 14,300 / Kong 11,500) | Kong | **374 µs** | 4.54 ms | **1.29 ms** | 11.71 ms | 13.16 ms | 28.12 ms | 256.2 ms |
| Cleartext H1/H1 (Flow 18,500 / Kong ~18,000) | **Flow** | 433 µs | **6.91 ms** | **6.83 ms** | **10.10 ms** | **11.10 ms** | **13.41 ms** | **31.3 ms** |
| Cleartext H1/H1 (Flow 18,500 / Kong ~18,000) | Kong | **320–358 µs** | 7.08–7.25 ms | 7.31–7.59 ms | 9.32–9.35 ms | 10.56–10.77 ms | 17.24–18.06 ms | 50.8–64.5 ms |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / Kong ~13,000) | **Flow** | 620 µs | **6.00 ms** | **5.50 ms** | **9.40 ms** | **10.86 ms** | **14.16 ms** | **92.4 ms** |
| TLS-H2 (client) → H1 (backend) (Flow 17,000 / Kong ~13,000) | Kong | **478 µs** | 10.95 ms | 11.64 ms | 18.96 ms | 21.60 ms | 44.54 ms | 233.1 ms |
| h2c (client) → H1 (backend) (Flow 22,700 / Kong ~17,000) | **Flow** | 447 µs | **7.16 ms** | **6.87 ms** | **11.34 ms** | **12.32 ms** | **14.05 ms** | **30.9 ms** |
| h2c (client) → H1 (backend) (Flow 22,700 / Kong ~17,000) | Kong | **436 µs** | 8.25 ms | 8.61 ms | 13.73 ms | 16.22 ms | 29.29 ms | 1.01 s (outlier) |

The pattern holds on every row: Kong's tail (p95/p99/max) runs well beyond Flow's even where
Kong's ceiling rate is lower. The h2c row's Kong max (1.01 s) is a documented single-run
outlier rather than a systemic tail problem — its own p99 was unremarkable at 29.29 ms.

```text
p99 latency at each product's own ceiling (ms)

TLS+HTTP/1.1    Flow 14,300 rps ██████████████                            15.48 ms
                Kong 11,500 rps █████████████████████████                 28.12 ms

Cleartext H1/H1 Flow 18,500 rps ████████████                              13.41 ms
                Kong ~18,000 rps ████████████████                         17.65 ms

TLS-H2 → H1     Flow 17,000 rps █████████████                             14.16 ms
                Kong ~13,000 rps ████████████████████████████████████████ 44.54 ms

h2c → H1        Flow 22,700 rps █████████████                             14.05 ms
                Kong ~17,000 rps ██████████████████████████               29.29 ms
```

### Cleartext HTTP/1.1 sustained — the cleanest single comparison

Same protocol, no encryption on either side, both at their own ceiling — Flow 18,500 req/s
(5-minute sustained), Kong ~18,000 req/s (60-second, provisional; see [Why sustained, not
burst](#why-sustained-not-burst)). This is the same row as in the table above, reproduced here
with its own writeup since it's the one configuration with no encryption cost and no HTTP/2
capability gap.

| | **Flow** | **Kong** |
|---|---|---|
| Min | 433 µs | **320–358 µs** |
| Mean | **6.91 ms** | 7.08–7.25 ms |
| p50 | **6.83 ms** | 7.31–7.59 ms |
| p90 | **10.10 ms** | 9.32–9.35 ms |
| p95 | **11.10 ms** | 10.56–10.77 ms |
| p99 | **13.41 ms** | 17.24–18.06 ms |
| Max | **31.3 ms** | 50.8–64.5 ms |

Kong is nominally faster at min/mean/p50 — it's absorbing roughly 500 fewer req/s at its own
provisional ceiling — but Flow leads from p90 onward. Kong's tail here is also its least
settled figure of the two: it's still a 60-second-burst result, unlike Flow's 5-minute
sustained one.

```text
p99 latency, cleartext HTTP/1.1 sustained (ms)

Flow 18,500 rps  █████████████████████████████             13.41 ms
Kong ~18,000 rps ████████████████████████████████████████  17.65 ms
```

### Latency behavior near the ceiling

Kong's four configurations degrade in visibly different ways as they approach their ceiling:

- **TLS-HTTP/1.1 and cleartext HTTP/1.1** show the median catching up to the tail as the CPU
  saturates — a classic capacity wall.
- **TLS-HTTP/2 (client hop)** shows tail latency (p99) plateauing early and staying flat across
  a wide range of rates, while the median climbs steadily toward it — consistent with a
  bounded-depth queue at the point where multiplexed client streams are translated to
  single-connection HTTP/1.1 requests to the backend, rather than a compute-cost effect.
- **Cleartext HTTP/2 (h2c, client hop)** collapses in delivered throughput well before CPU
  saturates — the bottleneck here is connection-pool related, not compute. See
  [What this does not measure](#what-this-measures--and-what-it-does-not).

## Memory

Resident memory of the gateway process.

| | **Flow** | **Kong** |
|---|---|---|
| Under load (60 s runs) | 20–30 MB | **~610–690 MB** |

```text
Memory under load (MB)

Flow ██                                       25 MB
Kong ████████████████████████████████████████ 650 MB
```

Kong's memory footprint was roughly 20–25× larger than Flow's at every point measured. No
unbounded growth was observed in any 60-second run — a repeated pair of runs on the same
already-warmed process showed growth essentially stop (well under 1%) after an initial
one-time increase on first start.

## A note on Kong's CPU-core balance

Kong's two nginx worker processes sometimes showed uneven CPU usage between cores at different
offered rates. Flow showed no such imbalance — its two cores remained balanced throughout the test.

## Test setup

| Component | Detail |
|---|---|
| Gateway host | 2 vCPU, 4 GB, `n2-custom-2-4096`, Intel Cascade Lake |
| Backend | separate host, HTTP echo returning a 14-byte body, serving TLS and cleartext |
| Load generator | separate host |
| Network | same zone, private addressing, sub-millisecond RTT |
| Load tool | vegeta, fixed rate, 60 s per run |
| Kong | Enterprise 3.15.0.2, **unlicensed** (free-tier feature set), default service/route config, no plugins |
| Flow | one route, one connector, no policies enabled |

All hosts were otherwise idle. Gateway and backend were unchanged between the two products —
only the gateway process was swapped.

## Why sustained, not burst

**Every Kong figure in this document is a 60-second run — none of Kong's numbers have been
confirmed with a multi-minute sustained run.** (Flow's own cleartext HTTP/1.1 figure, by
contrast, has been — see the TL;DR.) This matters concretely, not just in principle:
this page's own [APISIX comparison](apisix.md) found that a 60-second burst can overstate a
real ceiling by more than 50% for an nginx/OpenResty-based gateway — APISIX's cleartext
HTTP/1.1 figure moved from 10.7k (60-second) to 16.5k (5-minute sustained) once the correction
was made, because per-worker LuaJIT/GC pressure that a short run doesn't have time to build up
became visible under sustained load. Kong runs on the same nginx worker-process foundation and
has not yet had an equivalent sustained-run pass.

**Practical implication**: treat every Kong ceiling, ratio, and latency figure on this page as
provisional. The direction of any correction is unknown — it could move up, down, or stay flat
— but given the mechanism above, "unknown" is a materially different claim than "confirmed,"
and this page should not be read as a final word on Kong's sustained-load ceiling until that
testing is done.

## How to reproduce

### Kong configuration

A single service and route pointing at the echo backend, TLS enabled on the appropriate
listener for the TLS rows:

```yaml
services:
  - name: example-service
    url: http://BACKEND:3000/echo
routes:
  - name: example-route
    service: example-service
    paths:
      - /
```

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
echo 'GET https://GATEWAY:PORT/' | vegeta attack -http2=false \
  -rate RATE -duration 60s -workers 100 -max-workers 150 -insecure

# TLS + HTTP/2
echo 'GET https://GATEWAY:PORT/' | vegeta attack -http2=true \
  -rate RATE -duration 60s -workers 100 -max-workers 150 -insecure

# Cleartext HTTP/1.1
echo 'GET http://GATEWAY:PORT/' | vegeta attack -http2=false \
  -rate RATE -duration 60s -workers 100 -max-workers 150

# Cleartext HTTP/2 (h2c) — note the flag: -h2c, NOT -http2
echo 'GET http://GATEWAY:PORT/' | vegeta attack -h2c \
  -rate RATE -duration 60s -workers 100 -max-workers 150
```

`-http2` only takes effect over TLS (ALPN negotiation); it has no effect against a plain
`http://` target. Cleartext HTTP/2 requires the separate `-h2c` flag.

Sweep `RATE` upward until the delivered rate falls below the offered rate. Confirm the protocol
actually in use by counting connections on each side — an HTTP/2 client hop shows single-digit
connections, HTTP/1.1 shows roughly one per load-generator worker:

```bash
ss -tn state established '( sport = :GATEWAY_PORT )' | wc -l   # client side
ss -tn state established '( dport = :3000 )' | wc -l           # backend side
```

### A note on load-generator concurrency

Use enough workers that the load generator is never the limit. Required concurrency is
`rate × latency` — check this before recording a plateau as a ceiling.

## What this measures — and what it does not

**It measures** the throughput, latency, and memory of each gateway proxying a request to a
backend, on one hardware class, with a trivial endpoint and a small response body.

**It does not measure:**

- **Feature parity.** Kong is a full API gateway with a large plugin ecosystem, service mesh
  integration, and enterprise features. None of that is exercised here — this is a passthrough
  route, and Kong was tested without a license.
- **The exact mechanism behind Kong's cleartext-HTTP/2 throughput limit.** The measured ceiling
  for that configuration is lower than its CPU usage alone would predict, consistent with a
  backend connection-pool limit rather than a compute limit — but the specific Kong setting
  responsible was not identified in this round of testing.
- **Behaviour under other payload sizes**, request mixes, or routing complexity. A 14-byte
  response maximizes the relative weight of per-request overhead; larger bodies would shift the
  balance toward I/O.
- **Kong's sustained (multi-minute) behavior.** All of Kong's figures are 60-second runs; a
  longer run could reveal effects a short run misses — see [Why sustained, not
  burst](#why-sustained-not-burst). (Flow's own figures used in the like-for-like rows above
  are 5-minute sustained.)
- **Performance on other hardware.** A 2 vCPU host is small; results may not scale linearly.

Figures are specific to Kong Enterprise 3.15.0.2 (unlicensed) as of the test dates above. Both
products change over time; re-run the commands above to check current behaviour.

For Flow's own performance characteristics across configurations, independent of any
comparison, see [Performance](../performance.md). See also the [glossary](glossary.md) and
[comparisons overview](README.md). For a comparison against a second gateway, see
[Flow vs KrakenD CE](krakend.md).
