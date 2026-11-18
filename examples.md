# Examples

Complete, working KDL configs — each a real file you can download and run, not a fragment.
Every file in [`examples/`](examples/) passes `--validate-configs` as shown.

## Try one

1. Get a license file. See [Licensing](license.md).
2. Pick an example below and download it, or clone the file from `examples/`.
3. If it needs TLS, generate a certificate and key for trying it out (never use this for
   production — it's a throwaway, self-signed pair):

   ```bash
   openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
     -keyout key.pem -out cert.pem -subj "/CN=localhost"
   ```

   Run this in the same directory as the example file — the examples reference `./cert.pem`
   and `./key.pem`.
4. Point Flow at your license file (`export FLOW_LICENSE_FILE=/path/to/license.json` is the
   simplest way; see [Licensing](license.md) for the full precedence) and run:

   ```bash
   flow --config-kdl <example-file>.kdl
   ```

## The examples

| File | What it shows | Needs |
|---|---|---|
| [`quickstart.kdl`](examples/quickstart.kdl) | The simplest possible gateway — one route, plain HTTP, to a public backend ([httpbin.org](https://httpbin.org)). The first thing to run. | Nothing extra |
| [`tls-termination.kdl`](examples/tls-termination.kdl) | TLS in, plain HTTP to the backend. | A cert/key pair |
| [`rest-api-gateway.kdl`](examples/rest-api-gateway.kdl) | A fuller REST gateway: TLS, [token authentication](policies/token-auth.md), [rate limiting](policies/rate-limiting.md), and [CORS](policies/cors.md) together on one route. | A cert/key pair, and your own identity provider to see a request succeed (every request gets `401` without one — that's the route working as designed) |
| [`dns-discovered-upstream.kdl`](examples/dns-discovered-upstream.kdl) | A bare-hostname connector ([DNS-discovered](dns.md)) combined with [active health checking](policies/health-checking.md) and a [circuit breaker](policies/circuit-breaker.md). | Nothing extra |
| [`grpc-gateway.kdl`](examples/grpc-gateway.kdl) | A cleartext gRPC (h2c) gateway. See [Non-REST protocols](non-rest-protocols.md). | Your own gRPC backend |
| [`grpcs-gateway.kdl`](examples/grpcs-gateway.kdl) | gRPC over TLS, both hops. | A cert/key pair, and your own gRPC backend |
| [`metrics-enabled.kdl`](examples/metrics-enabled.kdl) | A working gateway with the [Prometheus metrics endpoint](metrics.md) secured with TLS, bearer-token auth, and a source CIDR allowlist. | A cert/key pair, and `FLOW_METRICS_TOKEN` set |

Most examples route to [httpbin.org](https://httpbin.org), a public echo service — enough to see
real traffic flow without standing up a backend first. Swap `connectors` for your own service
whenever you're ready. The gRPC examples are the exception: httpbin doesn't speak gRPC, so
those use a placeholder backend address instead.

## Related

- [Getting started](getting-started.md) — a guided walkthrough of your first config
- [Configuration](configuration.md) — the full KDL reference
- [Policies](policies.md) — every policy, with its own reference page
