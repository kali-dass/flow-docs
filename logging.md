# Logging

What Flow writes, where it goes, and how to configure it.

> **Running Flow?** See [Operations](operations.md)
> **Config syntax?** See [Configuration](configuration.md#loggingapp-log)
> **Per-request fields?** See [file-log](policies/file-log.md)

## Flow has two logs

They answer different questions, are configured separately, and never share a destination.

| | Application log | Request records |
|---|---|---|
| What it carries | Flow talking about itself | one record per request |
| Content | startup and shutdown, configuration faults, DNS changes, background tasks, upstream faults | status, timing, bytes, named headers and query parameters |
| Format | plain text | JSON, one object per line |
| On by default | yes, to stdout | **no** |
| Where it goes | stdout, or `<log-dir>/flow-application.log` | wherever the [`file-log`](policies/file-log.md) policy points |
| Volume controlled by | `level` | the policy's own sampling |

The dividing line between them: **nothing a client sends can change how much Flow writes to
the application log.** Anything that happens once per request belongs in a request record.

That has a consequence worth knowing before you need it. A refused request — a `429` from
[rate limiting](policies/rate-limiting.md), a `403` from [IP restriction](policies/ip-restriction.md),
a `413` from a [size limit](policies/request-size-limit.md) — does **not** appear in the
application log at `info`. It is a fact about one request, not about Flow.

## The application log

Two kinds of thing end up here.

### Configuration and lifecycle

What Flow did as it started, and what it is doing as it stops. Written at `info`: the
configuration loading and validating, the license check, each service being configured and its
listeners, DNS resolution for hostname connectors, and the shutdown sequence.

This is the log to read when Flow will not start, or started differently from how you expected.

Warnings worth watching for at startup:

- `verify-cert=false` — certificate validation is disabled on a connector. Never in production.
- `allow-h2c=true` — the upstream hop is cleartext. Only acceptable on a trusted network.
- A tuning option set where its protocol does not use it — `max-h2-streams` or
  `h2-ping-interval-secs` on an HTTP/1.1-only connector, for example.

### Runtime faults and degradation

What has gone wrong, or is going wrong, while Flow serves. Not per-request outcomes: things
about Flow's own state and the state of what it depends on.

- A backend becoming unreachable.
- A DNS answer that changes or empties a backend pool.
- A JWKS refresh failing, so [token authentication](policies/token-auth.md) is running on
  last-known-good keys.
- Records being dropped because a request-record destination cannot keep up.

### What each level means

| Level | What it tells you |
|---|---|
| `error` | Flow will not start, an upstream a route depends on is failing, or Flow hit an internal error |
| `warn` | Flow is serving, but degraded, falling back, or configured in a way that will surprise someone |
| `info` | Lifecycle: started, listening, configuration loaded, license valid, DNS set changed |
| `debug` | Per-request detail, including every refused request |
| `trace` | Everything, for narrow investigation |

`info` is the default and the right level to run at. `debug` is for developing a configuration,
not for production: on a busy gateway it turns on detail for every request.

### Configuring it

<!-- check-docs: skip -->
```kdl
system {
    logging {
        app-log {
            level "info"
            output "file"
            log-dir "/var/log/flow"
        }
    }
}
```

`app-log` is a child of `system.logging`, **not** of `system`. Placed directly under `system` it
is refused, with a message saying where it belongs.

With `output "file"`, Flow writes `<log-dir>/flow-application.log`. You choose the directory;
the filename is fixed. `log-dir` must **already exist** — Flow does not create it, so a
misspelled path cannot quietly become a new empty log directory somewhere you will not look.
Flow refuses to start if it is missing, is not a directory, or cannot be written to.

An existing `flow-application.log` is appended to, never truncated. Flow does not rotate it;
use `logrotate` or your platform's equivalent.

The level can be overridden per invocation with `--log-level`, which outranks the file. See
[Configuration](configuration.md#loggingapp-log) for the full reference and
[CLI](cli.md) for the flag.

**Two things cannot reach the file**, whatever you configure: command-line parsing, and a
configuration file that fails to parse at all. Neither can be routed by a configuration Flow has
not read yet, so both report on stdout.

`output "file"` works with or without `daemonize`. The two are independent — but `daemonize
true` does **not** imply it: leave `output` at `stdout` while daemonizing and the logs go to a
detached stdout, which is to say nowhere. See
[Running in the background](operations.md#running-in-the-background).

## Request records

One JSON object per request, written by attaching the [`file-log`](policies/file-log.md) policy
to a service or a route:

```kdl
system {
    logging {
        file-log-destination name="access" output="file" path="/var/log/flow/flow-access.log"
    }
}

services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        logging {
            filter kind="file-log" destination="access"
        }
        routes {
            route {
                name "orders"
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                match {
                    path-prefix "/orders"
                }
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

Note that `logging` sits beside `routes`, not inside `path-control`: it runs after the response
is complete, so it can only observe a request, never change or delay one.

When Flow generated the response itself, the record's `denied_by` field names what refused the
request. The status alone is ambiguous: `403` is shared by IP restriction and token
authentication, and `502` by a request matching no route, a route with no reachable backend, an
`h2-only` connector whose backend negotiated HTTP/1.1, a response that exceeded a size limit,
and an upstream that could not be reached.

[file-log](policies/file-log.md) is the full reference: every field, sampling, capturing named
or all headers and query parameters, destinations, and what happens under write pressure.

> **Preview:** request records may change before they are final. **Planned:** records to external sinks (HTTP, Kafka, syslog), and masking for secrets and personal data. See [Roadmap](roadmap.md#observability).

## There is no request record by default

Until you attach `file-log`, nothing records individual requests. The account of what happened
is the [metrics endpoint](metrics.md), which is aggregate: `flow_requests_total` counts by status
*class* (`4xx`), not by exact status, though its `source` label separates responses Flow
generated itself from those an upstream returned.

So on a default install, "which requests were refused, and why" has no answer. Attach `file-log`
before you need it, not during an incident.

## When a backend fails

A backend going down is reported in the application log, not left to per-request records. How
depends on whether anything is monitoring it:

- With [active health checking](policies/health-checking.md) or a
  [circuit breaker](policies/circuit-breaker.md) configured, the state change is reported when it
  happens — once, not per affected request.
- With **neither** configured, Flow has no notion of backend state, so there is no state change
  to report. Each failed request logs its own `error` instead, naming the route and saying that
  no health check is configured. Otherwise a dead backend would be invisible in the application
  log.

In that second case the volume is proportional to the failure rate, which is a further reason to
configure health checking in production.

A DNS change that shrinks a backend pool is reported at `info` — Flow cannot tell a fleet
scaling down from half a fleet failing, so it reports the change rather than judging it. A pool
emptied after repeated empty answers is an `error`, because every request to that route then
fails. See [DNS Resolution](dns.md).

## What is not logged

Request and response bodies, by either log. Named captures in a request record can carry
specific headers or query parameters you ask for, never a body.

## Beyond logs

- The [Prometheus metrics endpoint](metrics.md) for request counts, latency, and in-flight requests
- The [`timing-header`](policies/timing-header.md) filter for per-request latency in response headers
- `--log-level debug` for startup diagnostics

> **Planned:** OpenTelemetry tracing, and per-backend and per-policy metrics. See [Roadmap](roadmap.md#observability).
