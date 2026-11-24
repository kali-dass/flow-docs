# FAQ

Short answers to frequently asked questions, with a link to the page that has the detail. For what is available and what is planned, see the [Roadmap](roadmap.md).

**Can Flow cache responses?** 
Yes. cache-response policy does that. It is TTL-based, held in memory, `Cache-Control` aware, and supports Vary-based variants and per-identity isolation. See [Cache response](policies/cache-response.md).

**Can Flow proxy gRPC?** 
Yes. gRPC over cleartext HTTP/2 or TLS, on either hop between the client, Flow and the backend, including streaming calls, is proxied end to end. See [gRPC](non-rest-protocols.md#grpc).

**Can Flow redirect HTTP to HTTPS, or move an API to a new address?** 
Yes. The `redirect` policy can do that. It answers with a `3xx` and a `Location` and never calls the backend. For http to https, run a small service on the plain port that redirects everything. See [Redirect](policies/redirect.md).

**Can Flow handle CORS for browser applications?** 
Yes. Add a `cors` policy to a service or route and Flow answers preflight requests itself and adds the `Access-Control-*` headers. See [CORS](policies/cors.md).

**Can Flow limit the size of uploads?** 
Yes. Add the `request-size-limit` policy to a route and requests with a larger body get `413`. See [Request size limit](policies/request-size-limit.md).

**Can Flow limit the size of responses?** 
Yes. Add the `response-size-limit` policy to the `upstream-response` block: a response whose declared length is over the limit gets a `502`, and one that grows past it while streaming is cut off. See [Response size limit](policies/response-size-limit.md).

**Can Flow check whether a backend is healthy?** 
Yes. Set `health-check "Active"` in the `load-balance` block and Flow probes each backend and removes unhealthy ones from the pool. See [Health checking](policies/health-checking.md).

**Do rate limits apply across several Flow instances?** 
No. Each instance keeps its own in-memory limits, so a client's effective limit is the configured limit multiplied by the number of instances behind your load balancer. See [How Flow Behaves](behavior.md#multiple-instances) for how to size for that, and the [Roadmap](roadmap.md#traffic-control) for status.

**What throughput can I expect?** 
It depends on CPU cores, protocol, encryption, payload size, number of policies and other factors based on usecase: roughly 14,000 to 28,000 requests per second on 2 cores is what has been benchmarked. For details See [Performance](performance.md).

**Can Flow do virtual hosting?** 
Yes, by TLS SNI or by header matching. See [Routing](routing.md).

**Can Flow run in Kubernetes?** 
Yes. Deploy it as a DaemonSet or a Deployment and expose it with a Service. Upstream connectors accept Kubernetes DNS names such as `orders-svc.default.svc.cluster.local:8080` and refresh the backend pool when the address changes. Discovery is DNS-based, not watch-based. See [DNS Resolution](dns.md).

**Does Flow need a license?** 
Yes. Every instance needs a valid license file, checked at startup. See [Licensing](license.md) and [How Flow Behaves](behavior.md#license-check).

**How do I see what Flow is doing?** 
Flow exposes a Prometheus [metrics endpoint](metrics.md), can add [timing headers](policies/timing-header.md) to responses, and writes application logs. See [Operations](operations.md#logging).

**Is Flow a service mesh, a load balancer, or a WAF?** 
Flow sits alongside them; see [Where Flow Fits](scope.md).
