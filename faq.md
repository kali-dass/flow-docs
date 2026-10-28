# FAQ

Short answers, with a link to the page that has the detail. For what is available and what is planned, see the [Roadmap](roadmap.md).

**Can Flow cache responses?** Yes. Add a `cache-response { }` block to a route. It is TTL-based, held in memory, `Cache-Control` aware, and supports Vary-based variants and per-identity isolation. See [Cache response](policies/cache-response.md).

**Can Flow proxy gRPC?** Yes. gRPC over cleartext HTTP/2 or TLS, on either hop between the client, Flow and the backend, including streaming calls, is proxied end to end. See [gRPC](non-rest-protocols.md#grpc).

**Can Flow check whether a backend is healthy?** Yes. Set `health-check "Active"` in the `load-balance` block and Flow probes each backend and removes unhealthy ones from the pool. See [Health checking](policies/health-checking.md).

**Do rate limits apply across several Flow instances?** Each instance keeps its own limits. See [How Flow Behaves](behavior.md#rate-limiting).

**What throughput can I expect?** It depends on CPU cores, protocol, and encryption: roughly 14,000 to 28,000 requests per second on 2 cores. See [Performance](performance.md).

**Can Flow do virtual hosting?** Yes, by TLS SNI or by header matching. See [Routing](routing.md).

**Can Flow run in Kubernetes?** Yes. Deploy it as a DaemonSet or a Deployment and expose it with a Service. Upstream connectors accept Kubernetes DNS names such as `orders-svc.default.svc.cluster.local:8080` and refresh the backend pool when the address changes. Discovery is DNS-based, not watch-based. See [DNS Resolution](dns.md).

**Does Flow need a license?** Yes. Every instance needs a valid license file, checked at startup before Flow opens a listener. See [Licensing](license.md) and [How Flow Behaves](behavior.md#license-check).

**How do I see what Flow is doing?** Flow exposes a Prometheus [metrics endpoint](metrics.md), can add [timing headers](policies/timing-header.md) to responses, and writes application logs. See [Operations](operations.md#logging).

**Is Flow a service mesh, a load balancer, or a WAF?** Flow sits alongside them; see [Where Flow Fits](scope.md).
