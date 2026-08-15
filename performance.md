# Performance

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Protocols](protocols.md) • [Configuration Reference](configuration.md) • [Routing](routing.md) • [Operations](operations.md)

Flow's throughput is **CPU-bound and depends heavily on your configuration** .
- TLS vs. cleartext
- HTTP/1.1 vs. HTTP/2 
- for HTTP/2 whether both hops multiplex or only the client hop does 

ranging from **~14,300 to ~28,000 requests/second on 2 CPU cores** in testing. 
This page explains what was measured, what changes the number, and how to size your own deployment.

## Headline numbers

All rows below come from **the same test methodology, the same hardware, and the same
backend** — the only variable changed between rows is protocol/encryption, so these numbers are
directly comparable to each other.

| Configuration | Sustained | Wall | Mean latency @ knee |
|---|---|---|---|
| TLS + HTTP/1.1 (both hops, encrypted, no multiplexing) | ~14,300 req/s | ~14,400 | 5.90 ms |
| TLS + HTTP/2 (client) + TLS + HTTP/1.1 (backend) | ~17,000 req/s | ~17,200 | 6.00 ms |
| TLS + HTTP/2 (both hops, encrypted, multiplexed) | 20,000 req/s | ~20,000 | 6.73 ms |
| Cleartext HTTP/1.1 (both hops, no encryption, no multiplexing) | ~18,500 req/s | ~18,600 | 6.91 ms |
| Cleartext HTTP/2 (client) + HTTP/1.1 (backend) | 22,700 req/s | ~23,000 | 7.16 ms |
| **Cleartext HTTP/2 — h2c** (both hops, no encryption, multiplexed) | **28,000 req/s** | ~29,000 | 6.66 ms |

```text
Sustained throughput (req/s, 2 vCPU)

TLS+H1 (both hops)                ████████████████████ 14,300
TLS+H2/H1 (client H2, backend H1) ████████████████████████ 17,000
TLS+H2 (both hops)                █████████████████████████████ 20,000
Cleartext H1 (both hops)          ██████████████████████████ 18,500
h2c/H1 (client h2c, backend H1)   ████████████████████████████████ 22,700
h2c (both hops)                   ████████████████████████████████████████ 28,000
```

Two configuration remains untested as of this writing and is intentionally omitted. 

Any config where only the *backend* hop uses HTTP/2 while the client hop is HTTP/1.1
or TLS-HTTP/1.1.
- Client - TLS HTTP/1.1 - flow - TLS HTTP/2 - backend
- Client - HTTP/1.1 - flow - HTTP/2 - backend

This page doesn't claim numbers Flow hasn't actually measured

The pattern: **encryption costs roughly the same whichever protocol you use, and multiplexing —
even on just the client hop — recovers a meaningful share of that cost**. Full end-to-end
multiplexing recovers the most. See [the two levers](#the-two-levers-measured-together--http2-vs-http11-encrypted-vs-not)
for the full breakdown.

## Detailed metrics at the sustained ceiling

The headline table above gives one throughput number and one latency number per configuration.
If you're sizing a deployment, you need more than that: the full latency distribution and how
much CPU headroom you have at that rate. All figures below are from the **same sustained run**
that defines each configuration's ceiling in the table above — not a best-case 60-second burst.

> **On memory:** the RSS figures below were captured during multi-rate test sequences — each
> configuration was swept through several rates in the same long-running process before the
> sustained run, rather than measured immediately after a fresh restart. Treat them as
> representative of steady-state memory use under realistic, sustained traffic rather than a
> bare-minimum figure.

All runs delivered 100% of achieved requests successfully (the two H2 configs marked below
self-throttle a hair under nominal at their ceiling — noted inline).

| Configuration | Sustained | Min | Mean | p50 | p90 | p95 | p99 | Max | CPU busy | Memory (RSS) |
|---|---|---|---|---|---|---|---|---|---|---|
| TLS + HTTP/1.1 (both hops) | 14,300 req/s | 446 µs | 5.90 ms | 4.93 ms | 10.94 ms | 12.41 ms | 15.48 ms | 188.9 ms | 96.5% | ~35.2 MB |
| TLS + HTTP/2 (client) + TLS + HTTP/1.1 (backend) | 17,000 req/s | 620 µs | 6.00 ms | 5.50 ms | 9.40 ms | 10.86 ms | 14.16 ms | 92.4 ms | 90.1% | ~25–26 MB |
| TLS + HTTP/2 (both hops) | 20,000 req/s* | 617 µs | 6.73 ms | 6.49 ms | 9.60 ms | 10.88 ms | 13.31 ms | 115.8 ms | 91.5% | ~43–53 MB |
| Cleartext HTTP/2 (client) + HTTP/1.1 (backend) | 22,700 req/s* | 447 µs | 7.16 ms | 6.87 ms | 11.34 ms | 12.32 ms | 14.05 ms | 30.9 ms | 94.7% | ~16.3–16.7 MB |
| Cleartext HTTP/1.1 (both hops) | 18,500 req/s | 433 µs | 6.91 ms | 6.83 ms | 10.10 ms | 11.10 ms | 13.41 ms | 31.3 ms | 98.2% | ~35.2 MB |
| Cleartext HTTP/2 — h2c (both hops) | 28,000 req/s* | 550 µs | 6.66 ms | 6.52 ms | 9.28 ms | 9.98 ms | 11.25 ms | 44.9 ms | 94.6% | ~32–43 MB |

\* Achieved 99.95%, 99.97%, and 99.81% of nominal respectively (h2c-client/H1-backend,
h2c-both-hops, and TLS+H2 in that order) — 100% of the *achieved* rate delivered successfully.
See [the two levers](#the-two-levers-measured-together--http2-vs-http11-encrypted-vs-not) for
why the H2 configs self-throttle a hair under nominal at their ceiling.

```text
p99 latency at the sustained ceiling (ms, 2 vCPU)

TLS+H1 (both hops)                ████████████████████████████████████████ 15.48 ms
TLS+H2/H1 (client H2, backend H1) █████████████████████████████████████ 14.16 ms
h2c/H1 (client h2c, backend H1)   ████████████████████████████████████ 14.05 ms
Cleartext H1 (both hops)          ███████████████████████████████████ 13.41 ms
TLS+H2 (both hops)                ██████████████████████████████████ 13.31 ms
h2c (both hops)                   █████████████████████████████ 11.25 ms
```

The general pattern still holds — h2c (both hops), the highest-throughput configuration, also
has the *lowest* p99 — but it's no longer a strict inverse relationship across every row:
h2c-client/H1-backend now sustains the second-highest throughput (22,700) while carrying a
higher p99 (14.05 ms) than lower-throughput configs like cleartext H1 and TLS+H2. That's the
backend-H1 hop's per-connection churn showing up at the tail even though the client hop is
multiplexed — full end-to-end multiplexing (h2c) is what actually delivers both the throughput
and the latency win together; multiplexing only the client hop delivers the throughput without
the same tail-latency benefit.

**Reading these together:** mean and p50 track throughput closely — the busier configurations
(higher rps) also show higher latency at every percentile, since they're running closer to CPU
saturation. p99/mean ratios stay in a tight 1.7×–2.6× band across all six configs, so no single
configuration has a disproportionately worse tail — the whole distribution shifts together with
CPU busy. Max latency in particular is a single outlier per run, not representative; use it to
sanity-check for hiccups, not as an SLA number.

## Test setup

All figures come from Google Cloud, with **the load generator, Flow, and the backend all in
the same zone and VPC**. They talk over private addresses, so network latency is
sub-millisecond and never a factor in these numbers — what you see is Flow's own cost.

| Role | Machine | CPU | Memory |
|---|---|---|---|
| **Flow** | GCP N2, 2 vCPU | Intel Xeon (Cascade Lake) @ 2.80 GHz | 4 GB |
| **Backend** | separate GCP VM, 4 vCPU | RPS-tuned, dual-protocol echo server | 4 GB |
| **Load generator** | separate GCP VM, 6 vCPU | RPS-tuned | — |

### Backend
The backend is a small HTTP echo service returning a short JSON payload. It was RPS-tuned and
verified to handle far more than Flow's ceiling on its own, so **the backend is never the
bottleneck** in any of these figures.

### Load Generator
Load was generated with `vegeta` at a fixed rate. Rates in the table above are the confirmed
**sustained** figure — TLS+HTTP/1.1, TLS+HTTP/2 (client) + TLS+HTTP/1.1 (backend), Cleartext
HTTP/2 (client) + HTTP/1.1 (backend), Cleartext HTTP/1.1, TLS+HTTP/2 (both hops), and Cleartext
HTTP/2 (h2c, both hops) were each verified with a 5-minute (or longer) run. See
[Measuring performance in your environment](#measuring-performance-in-your-environment) for why
that distinction matters before you trust a number from a short run.

An older, differently-configured measurement appears later in this page (the
[hardware notes](#hardware-notes) section) — it used a different backend, a different load
tool (`wrk2`), and a **plain HTTP/1.1 client hop** with only the backend hop set to h2c, a
configuration not present in the table above. It's labeled where it applies; don't average
its numbers with the headline table's.

## What limits Flow

Flow is **CPU-bound**, not I/O-bound or memory-bound. Memory never mattered; 4 GB is far more
than Flow uses.

The CPU goes mostly to **encryption**. A proxy that terminates TLS from the client and
originates TLS to the backend pays that cost **twice per request** — once to decrypt what the
client sent, once to re-encrypt it for the backend. That double cost, not Flow's routing or
filtering, is what sets the ceiling. Flow's own logic takes only tens of microseconds per
request.

That gives you two levers: **encrypt less**, or **add cores**.

### Lever 1 — remove encryption from the backend hop

If Flow and your backend sit inside a trusted network, the backend hop may not need TLS.
Removing it saves a full round of AES-GCM — the [headline numbers](#headline-numbers) above
show encryption costs ~26–27% throughput on both protocols, so dropping it on the backend hop
alone (while keeping the client hop as-is) is expected to recover a meaningful fraction of
that, though the exact combined number for "TLS client, cleartext backend" specifically hasn't
been re-measured under the current methodology — treat it as directionally correct, not a
precise figure. For the fully-measured "cleartext on both hops" case with hard numbers, see
[Lever 3](#lever-3--offer-h2c-to-clients-too-when-theyre-also-inside-a-trusted-boundary) below.

Use **h2c** (cleartext HTTP/2):

```kdl
"10.0.0.1:3000" proto="h2-only" allow-h2c=true
```

> ⚠️ **Do not simply delete `tls-sni`.** HTTP/2 to the backend is negotiated as part of the
> TLS handshake, so removing TLS without `allow-h2c` silently drops you to HTTP/1.1 — which
> means one request per connection. The extra connection churn cancels out the saving and
> makes your tail latency about **3× worse**. You end up slower than where you started.
>
> `allow-h2c=true` is what keeps HTTP/2 while dropping TLS. Always use it together with
> `proto="h2-only"`.

**This is a security trade-off, not just a performance one — read
[Choosing an upstream configuration](#choosing-an-upstream-configuration) before adopting
it.**

### Lever 2 — add cores, then add instances

Flow scales **close to linearly with CPU cores**. Both cores saturated evenly in testing, so
doubling from 2 to 4 vCPU should roughly double throughput. Set `threads-per-service` to your
core count:

```kdl
system {
    threads-per-service 4    // on a 4-core machine
}
```

Hyperthreading does not help CPU-bound proxy work — count physical cores, not hyperthreads.

Beyond a single machine, **run several Flow instances behind a load balancer**. Flow keeps no
shared state between requests, so instances scale out horizontally: two instances behind an
L4 load balancer give you roughly twice the capacity of one. This is also how you get
redundancy — a single Flow is a single point of failure.

For most deployments, **scaling out is the right answer** once you approach the ceiling of one
box. It gives you headroom and availability at the same time.

### Lever 3 — offer h2c to clients too, when they're also inside a trusted boundary

The numbers above assume an HTTP/1.1 client. If your clients are services you control — not a
browser — and the connection to Flow is also inside a trusted network, offering cleartext
HTTP/2 to clients as well removes encryption from *both* hops and lets HTTP/2 multiplex the
client-facing connection too, not just the backend one:

```kdl
listeners {
    "0.0.0.0:8000" offer-h2=true    // clients speaking h2c connect here; H1 clients still work
}
connectors {
    "10.0.0.1:3000" proto="h2-only" allow-h2c=true
}
```

Flow runs HTTP/2 on both hops either way — encrypted or not, genuinely multiplexed client-to-Flow
and Flow-to-backend, not just on the client-facing side. On the same 2 vCPU class of machine,
sustained over 5-minute runs:

| Both hops | Sustained | Per minute | Mean latency |
|---|---|---|---|
| **TLS + HTTP/2** | 20,000 req/s | 1.2 million | ~6.7 ms |
| **Cleartext HTTP/2 (h2c)** | 28,000 req/s | 1.68 million | ~6.7 ms |

Dropping encryption from both hops is worth roughly **40% more throughput**. A 30-minute
endurance run on the cleartext path also confirmed memory stays flat, well under 50 MB, for as
long as the load continues rather than only at the start.

⚠️ **Same security trade-off as the backend hop** — see
[Choosing an upstream configuration](#choosing-an-upstream-configuration) — plus one more
constraint specific to the client side: h2c requires the client to speak HTTP/2 by **prior
knowledge** (no ALPN negotiation, no TLS handshake to negotiate it during). Ordinary web
browsers cannot do this. This lever is only usable when your clients are services you control
and that support prior-knowledge HTTP/2 directly — internal APIs, service-mesh traffic, gRPC
clients — never for public or browser-facing endpoints.

---

## The two levers, measured together — HTTP/2 vs HTTP/1.1, encrypted vs not

The two levers above — **connection reuse** (HTTP/2's multiplexing) and **encryption** — are
independent. A separate, more fully instrumented benchmark measured all four combinations
end-to-end on the same 2-core class of machine, so you can see exactly what each one is worth:

|  | **HTTP/1.1** (one request per connection) | **HTTP/2** (multiplexed) |
|---|---|---|
| **Cleartext** (no TLS) | ~18,500 req/s · 1.1M/min | **28,000 req/s · 1.68M/min** |
| **TLS** (both hops) | ~14,300 req/s · 0.86M/min | 20,000 req/s · 1.2M/min |

Reading it two ways:

- **Multiplexing is the bigger lever.** Moving from HTTP/1.1 to HTTP/2 buys **+40–51%** whether
  or not you encrypt (14,300 → 20,000 with TLS, +40%; 18,500 → 28,000 cleartext, +51%). One
  HTTP/2 connection carrying many concurrent requests avoids the per-request connection setup
  that HTTP/1.1 repeats every time. Multiplexing only the client hop (cleartext HTTP/2 client,
  HTTP/1.1 backend — see [headline numbers](#headline-numbers)) still recovers a meaningful
  share of that gain on its own, landing at 22,700 — HTTP/2 helps even when only one side of the
  connection uses it, though clearly less than multiplexing both hops.
- **Encryption is the smaller, steadier cost on HTTP/1.1, but grows on HTTP/2.** Turning on TLS
  costs about **−22.7%** on HTTP/1.1 (18,500 → 14,300) and now **−28.6%** on HTTP/2
  (28,000 → 20,000) — the wider HTTP/2 gap reflects the corrected, much higher cleartext-h2c
  ceiling rather than a change in TLS's own per-request cost.

The two stack: the fastest cell (cleartext HTTP/2) is about **1.96× the slowest** (TLS + HTTP/1.1),
which is roughly the two levers multiplied together.

**A note on latency:** the HTTP/1.1 rows have *lower* average latency at light load — with one
request per connection, nothing queues behind anything else — but they hit their ceiling much
sooner and their tail latency degrades faster near it. Low latency at low load is not the same as
high capacity. Size for capacity, then check your tail percentiles.

This table and the [headline numbers](#headline-numbers) above are the same dataset — this
section just breaks out the two levers (multiplexing, encryption) individually so you can see
what each is worth on its own.

---

## What happens past the ceiling — read this before you set a rate

Every configuration in the [headline numbers](#headline-numbers) above was pushed past its
sustained ceiling as part of finding it, and all of them degrade the same way: **gracefully**.
Push the offered rate past the ceiling and the *achieved* rate flattens into a plateau instead
of continuing to climb — the extra offered load doesn't get through, but what does get through
keeps succeeding, and latency rises smoothly rather than jumping by orders of magnitude.

For example, on TLS+HTTP/2 (ceiling 20,000 req/s): offering 21,000 delivered only ~19,850 req/s
(self-throttled), at 100% success and a mean latency of 7.14 ms — barely above the 6.73 ms at
the 20,000 ceiling itself. The same pattern held on cleartext h2c (ceiling 28,000) and on
cleartext-h2c-client/HTTP-1.1-backend (ceiling 22,700): each configuration's achieved rate
plateaus at its ceiling when offered more, rather than collapsing.

**Still watch headroom, not just success rate.** The plateau means you won't see failed
requests or timeouts as an early warning — you'll see rising latency and a widening gap between
offered and achieved rate. Both are real signals:

- **Check the delivered rate, not just the offered rate.** If your load generator reports fewer
  requests/second than you asked for, you're past the wall.
- **Watch CPU headroom** (`mpstat -P ALL 1`), not average CPU. Idle percentage predicts how much
  further you can push before latency starts climbing.
- **Run with real headroom regardless of configuration.** Target meaningfully below your own
  measured ceiling and scale out before you get close to it — see
  [Measuring performance in your environment](#measuring-performance-in-your-environment).

---

## Measuring performance in your environment

The numbers above are a starting point, not a promise. Your payload sizes, routing rules,
policies, TLS settings, and backend behaviour will all move them. **Measure your own
workload.**

Two recommendations:

**Run each test for at least 10–15 minutes.** Short runs overstate capacity — badly. A rate
that looks perfectly stable for 60 seconds can collapse when held for 10 minutes, because a
small capacity deficit takes minutes to build into a visible queue. A 60-second result is a
*burst* figure, not a sustained one. Anything under 10 minutes will mislead you.

**Then adjust for your own use case.** Test with your real payload sizes, your real routing
and policy configuration, and traffic that looks like your traffic. A benchmark against a
trivial echo endpoint tells you Flow's ceiling, not yours.

When you measure:

- Use a load generator that holds a **fixed request rate** and reports latency percentiles
  honestly under load (`wrk2` does; plain `wrk` does not).
- **Check the delivered rate, not just the offered rate.** If the tool reports fewer
  requests/second than you asked for, you are past the wall and the latency figures are
  queueing artifacts.
- **Watch CPU headroom** (`mpstat -P ALL 1`), not average CPU. Idle percentage is what
  predicts the cliff.
- **Turn the `timing-header` filter off** if enabled — it adds two response headers per request; not a big cost, but keep the benchmark clean.
- **Repeat runs near the limit.** Close to the cliff, whether a run collapses is partly luck.
  A single run near the limit tells you very little.

---

## Choosing an upstream configuration

> **Decide this on security grounds first, then optimize within that constraint.**
> The table below ranks configurations by throughput, but throughput is the *last* thing you
> should choose on. An unencrypted backend hop is a security decision — it belongs to your
> threat model, your compliance obligations (PCI-DSS, HIPAA, GDPR and similar regimes may
> require encryption in transit), and your network architecture. **Never adopt h2c for
> performance reasons alone.**

Start by answering: **can traffic on the Flow→backend hop be read by anyone who shouldn't
see it?** That means anyone who could observe the network path — other tenants, other
workloads in the same VPC, an operator with packet capture, or anything traversing a network
boundary you don't control.

- If **yes**, or you're unsure, or you're bound by a compliance regime — **use TLS**. Accept
  the ~14,300–20,000 req/s ceiling (H1 or H2 on the backend hop, see
  [headline numbers](#headline-numbers)) and scale out with more instances if you need more.
- If **no** — the hop is genuinely inside a trusted boundary you control — then h2c is
  available, and it's a meaningful win.

| Your backend hop | Configuration | Sustained | Encrypted? |
|---|---|---|---|
| Crosses any untrusted network | `tls-sni="host" proto="h2-or-h1"` | 20,000 req/s (H2 both hops) / ~17,000 req/s (H2 client only, H1 backend) / ~14,300 req/s (H1 both hops) | ✅ Yes |
| Inside a trusted network you control | `proto="h2-only" allow-h2c=true` | meaningfully higher — see [Lever 1](#lever-1--remove-encryption-from-the-backend-hop) (client stays TLS) or 28,000 req/s if the client hop is also trusted and cleartext (see [Lever 3](#lever-3--offer-h2c-to-clients-too-when-theyre-also-inside-a-trusted-boundary)) | ❌ **No** |
| Backend only speaks HTTP/1.1 | `proto="h1-only"` | ~14,300–18,500 req/s depending on client-hop encryption; ~17,000 req/s if the client hop is TLS+HTTP/2; **22,700 req/s if the client hop is cleartext HTTP/2** | Depends on `tls-sni` |

Note that "inside a VPC" is not automatically "trusted" — a shared VPC, a multi-tenant
cluster, or a network you don't fully control may still warrant encryption. Encryption in
transit is also increasingly expected as a default (zero-trust networking), regardless of
where the traffic flows.

**If in doubt, use TLS and add a core.** Scaling out is cheap; a data breach is not.

---

## Hardware notes

> Measured on an older configuration (plain HTTP/1.1 client, h2c backend, `wrk2` load tool) —
> not one of the [headline configurations](#headline-numbers) above. The comparison itself
> (relative CPU cost between vendors) is expected to generalize; the absolute rate (19,000) is
> specific to that config and hasn't been re-verified under the current methodology.

On identical 2 vCPU machines running the same workload, **AMD (GCP N2D, EPYC Milan) needed
about 5 percentage points less CPU than Intel (GCP N2, Xeon Cascade Lake)** — 86% vs 91% busy
at 19,000 req/s (this configuration's own ceiling).

This is a difference in **raw per-core processing speed** — how much work each core gets done
per clock cycle. It is not specific to any one part of Flow's work; the AMD cores are simply
faster at this workload. (The comparison was run on the cleartext configuration, so encryption
is not what separates them.)

The practical consequence: at the same request rate, the AMD machine has more headroom before
the cliff. **If you're choosing an instance type for Flow, N2D is the better value.**

Memory is a non-issue at any of these rates.
