# Comparisons

Flow measured against other gateways, on identical hardware with an identical workload.

**TL;DR**: Flow is ahead or roughly tied in every configuration — cleartext HTTP/1.1 (no
encryption cost, no HTTP/2 capability gap) is the closest matchup against both Kong and APISIX.
Everywhere TLS or HTTP/2 is involved, Flow's lead widens, up to 4.6× on TLS+HTTP/1.1 against
APISIX.

> **All of Flow's figures below are 5-minute sustained-confirmed.** Every page includes the exact commands to reproduce them. Kong's and KrakenD's are 60-second bursts (see individual pages for caveats); APISIX's are also sustained-confirmed.

These are engineering measurements of **proxy throughput and latency**, not feature comparisons
— each linked page states plainly what it does and does not measure, and includes the exact
commands to reproduce its numbers yourself. See the [glossary](glossary.md) if a term (p99,
sustained ceiling, h2c, ...) is unfamiliar.

| Comparison | Headline |
|---|---|
| [KrakenD CE](krakend.md) | Flow ~2.12× best-vs-best; 1.20–1.40× on like-for-like HTTP/1.1, 1.60× (TLS) / 2.10× (cleartext) on like-for-like client-H2/backend-H1 |
| [Kong Enterprise](kong.md) | Flow ~1.56× best-vs-best; **roughly tied on cleartext HTTP/1.1** (Kong's side provisional), ahead once TLS/HTTP/2 is introduced (~1.24× TLS, ~1.34× cleartext on the like-for-like HTTP/2 rows) |
| [Apache APISIX](apisix.md) | Flow ~1.47× best-vs-best; 1.12–4.6× depending on configuration, widest gap on TLS+HTTP/1.1 (4.6×), 1.60× (TLS) / 1.47× (cleartext) on like-for-like client-H2/backend-H1 |

## All six configurations, all three competitors

Sustained throughput (req/s), highest offered rate each gateway delivered ~100% of requests at.
Ratio in parentheses is Flow ÷ competitor; "Flow ahead" unless noted.

| Configuration | **Flow** | Kong | KrakenD CE | APISIX |
|---|---|---|---|---|
| TLS + HTTP/1.1 (both hops) | **14,300** | 11,500 (1.24×) | 11,900 (1.20×) | ~3,100 (4.6×) |
| HTTP/2 end-to-end (TLS, Flow only) | **20,000** | **Not supported** | **Not supported** | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), TLS (no capability caveat) | **17,000** | ~13,000 (1.31×, provisional) | 10,600 (1.60×) | 10,600 (1.60×) |
| Cleartext HTTP/1.1 (both hops) | **18,500** | ~18,000 (**~1.03×, roughly tied**) | 13,200 (1.40×) | ~16,500 (1.12×) |
| HTTP/2 end-to-end (cleartext, h2c, Flow only) | **28,000** | **Not supported** | **Not supported** | **Not supported** |
| HTTP/2 (client) + HTTP/1.1 (backend), cleartext (no capability caveat) | **22,700** | ~17,000 (~1.34×, provisional) | 10,800 (2.10×) | 19,000 (1.19×) |
| **Best configuration of each** | **28,000** | **~18,000 (~1.56×)** | **13,200 (2.12×)** | **19,000 (1.47×)** |

## Throughput by configuration

```text
TLS+HTTP/1.1        Flow    ████████████████████████            14,300
                    APISIX  ██                                   3,100
                    KrakenD ████████████████████                11,900
                    Kong    ███████████████████                 11,500

Cleartext H1/H1     Flow    ██████████████████████████          18,500
                    APISIX  █████████████████                   16,500
                    Kong    █████████████████████████           18,000
                    KrakenD ██████████████                      13,200

TLS+HTTP/2→H1       Flow    █████████████████████████           17,000
                    APISIX  ██████████████                      10,600
                    KrakenD ████████████████                    10,600
                    Kong    █████████████                       13,000

h2c-client→H1       Flow    ██████████████████████████████      22,700
                    APISIX  ████████████████████                19,000
                    Kong    ████████████████████                17,000
                    KrakenD ████████████                        10,800

TLS+HTTP/2 both     Flow    ██████████████████████████          20,000
(Flow only)

h2c both hops       Flow    ████████████████████████████████████████ 28,000
(Flow only)
```

The two end-to-end HTTP/2 rows are Flow-only — none of the three competitors run HTTP/2 to the
backend in these tests, a hard capability gap rather than a slower result; see each page's
HTTP/2 callout for detail. The two client-H2/backend-H1 rows are where all three competitors
already run that exact shape, so those are genuine like-for-like comparisons: Flow's 17,000 and
22,700 are both 5-minute sustained-confirmed, as are KrakenD's and APISIX's figures on both
rows; Kong's ~13,000 and ~17,000 are still 60-second-burst only. Flow's cleartext HTTP/1.1
(18,500) and TLS+HTTP/1.1 (14,300) figures also have 5-minute sustained-run confirmation; Kong's
~18,000 cleartext figure does not yet.

**Which row is relevant to you?** See "Which configuration is like mine?" on each competitor's
page — it maps these four lab configurations to common real-world deployment shapes (TLS
termination at the edge, internal plaintext backends, etc).

Each comparison names its own test setup, hardware, and reproduction commands — see the
individual pages.
