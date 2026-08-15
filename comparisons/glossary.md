# Glossary

Terms used across the Flow vs. gateway comparison pages.

| Term | Meaning |
|---|---|
| **Sustained ceiling** | The highest offered rate a gateway delivers ~100% of requests at, confirmed by a multi-minute run — not just a 60-second burst. Some gateways look fine on a short burst and degrade once held longer; see "Why sustained, not burst" in each page. |
| **Burst ceiling** | The highest rate that succeeds over a short (60-second) run. Can overstate real capacity if it doesn't match the sustained figure at the same rate. |
| **RPS / req/s** | Requests per second *delivered* by the gateway — not the same as the *offered* rate once a gateway starts dropping or queueing requests. |
| **Mean latency** | Average response time. Sensitive to outliers — a handful of slow requests can pull it up even if most requests are fast. |
| **p50 / median** | Half of requests were faster than this. Represents the "typical" request. |
| **p90 / p99** | 90%/99% of requests were faster than this. This is what users feel during traffic spikes — a good mean with a bad p99 means most requests are fine but a meaningful fraction are noticeably slow. |
| **Cliff (failure mode)** | Throughput or latency degrades sharply once the offered rate crosses a threshold, often within a 100 req/s window. |
| **Graceful degradation** | Latency rises smoothly as offered load increases past the ceiling, without a sudden collapse in delivered throughput. |
| **h2c** | HTTP/2 over cleartext — "prior knowledge" HTTP/2 without a TLS/ALPN negotiation. |
| **HTTP/2 multiplexing** | Multiple concurrent requests carried over one TCP connection, instead of one connection per request as in HTTP/1.1. Reduces connection-setup and TLS-handshake overhead under concurrency. |
| **TLS handshake cost** | CPU/time spent establishing an encrypted connection. Paid once per connection — HTTP/1.1 pays it per request; HTTP/2 amortizes it across every request multiplexed onto that connection. |
| **Accept queue** | A kernel-level queue of TCP connections waiting to be accepted by the application. Saturating it looks different from CPU saturation — the process can show low CPU while new connections back up.  |

For the load generator settings and reproduction steps behind these numbers, see each page's own "How to reproduce" section.
