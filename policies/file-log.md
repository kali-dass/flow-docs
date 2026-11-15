# File-log

> **Preview:** shipped and usable, may change before it is final. See [Roadmap](../roadmap.md#observability).

Writes one JSON line per request to a file, stdout, or stderr — status, timing, bytes
transferred, and (optionally) specific request headers or query parameters you name. This is
Flow's per-request access log, separate from [the application log](../configuration.md#loggingapp-log)
(Flow's own startup/warning/error messages).

## Quick start

```kdl
system {
    logging {
        file-log-destination name="access" output="file" path="/var/log/flow/access.log"
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
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

Every request on this service writes one line like:

```json
{"timestamp":1732400000000,"hostname":"unknown","version":"0.2.0","service":"Api","route_matched":"orders","request_id":null,"remote_addr":"10.0.0.1","host":null,"method":"GET","path":"/orders","protocol":"H1","status":200,"bytes_in":0,"bytes_out":128,"duration_upstream_us":1420,"duration_flow_us":35,"upstream_addr":"10.0.0.5:8080","captures":{}}
```

## Usage

`logging` is a block on a **service** or a **route**, next to `routes` — not inside
`path-control`. Both levels can carry a `filter kind="file-log"`; if both do, **both run** (a
service-level catch-all and a more specific route-level copy, writing one record each,
possibly to different destinations) — attaching it twice is not an error, and nothing
deduplicates between them.

`destination` names a block declared under `system.logging.file-log-destination` (see
[Configuration](../configuration.md#loggingfile-log-destination) for its settings). Omit it to
use the built-in default, a file at `flow-access.log` in the working directory.

### At the service level

A `logging` block directly on a service applies to **every route in that service** — the
blanket, catch-all placement:

```kdl
system {
    logging {
        file-log-destination name="access" output="file" path="/var/log/flow/access.log"
    }
}

services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        logging {
            filter kind="file-log" destination="access" {
                capture-headers {
                    header name="X-Request-Id" as="request_id"
                }
            }
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
            }
            route {
                name "catalog"
                match {
                    path-prefix "/catalog"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.6:8080"
                }
            }
        }
    }
}
```

Both `orders` and `catalog` are logged here — neither route declares its own `logging` block,
so both inherit the service-level one.

### At the route level

A `logging` block inside a specific `route { }` applies **only to that route** — everything
else in the service is unaffected:

```kdl
system {
    logging {
        file-log-destination name="access" output="file" path="/var/log/flow/access.log"
    }
}

services {
    Api {
        listeners {
            "0.0.0.0:8080"
        }
        routes {
            route {
                name "orders"
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
                logging {
                    filter kind="file-log" destination="access" {
                        capture-queries {
                            param name="client_version" as="client_version"
                        }
                    }
                }
            }
            route {
                name "health"
                match {
                    path-prefix "/health"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
            }
        }
    }
}
```

Only `orders` is logged here — `health` has no `logging` block of its own, and there's no
service-level one to inherit, so it isn't logged at all. This is the way to log a handful of
routes precisely rather than a whole service, or to give one route its own capture
configuration independent of the rest.

### Both together, and multiple destinations

Declare as many `file-log-destination` blocks as you need — each gets its own queue, buffer,
and writer thread, entirely independent of the others. A common pattern: a full-fidelity
catch-all at the service level, plus a second, separately-tuned destination for a specific
route's errors:

```kdl
system {
    logging {
        file-log-destination name="access" output="file" path="/var/log/flow/access.log"
        file-log-destination name="errors" output="file" path="/var/log/flow/errors.log"
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
                match {
                    path-prefix "/orders"
                }
                allowed-methods "GET"
                allowed-protocols "http" "h1"
                connectors {
                    "10.0.0.5:8080"
                }
                logging {
                    filter kind="file-log" destination="errors" sampling="errors-only"
                }
            }
        }
    }
}
```

`sampling` always lives on the `filter kind="file-log"` line, never on `file-log-destination`
itself — a destination block only takes `name`/`output`/`path` and the
reliability/queue/overflow attributes in [Configuration](../configuration.md#loggingfile-log-destination).

Every request on this route now writes **two** records: the service-level `access` filter's
catch-all copy (every request, since it has no `sampling` of its own — defaults to `"all"`),
and, only for a non-2xx/3xx response or one Flow never got any response for, a second copy to
`errors`, via the route-level filter. Two filters can also share the *same* destination —
nothing requires a route-level filter to use a different one from its service — in which case
both just add to that one destination's queue independently; if that shared destination's
circuit opens or its queue backs up, [adaptive sampling](#adaptive-sampling) forces *both*
filters into `errors-only` together, since they're reacting to the one destination's shared
health, not to each other.

## What's in a record

Every record carries these fields, always:

| Field | Meaning |
|---|---|
| `timestamp` | Unix epoch milliseconds. |
| `hostname`, `version` | The host and the running Flow build. `hostname` is currently always `"unknown"` — resolving the real host name is unimplemented. |
| `service`, `route_matched` | The matched service and route name; `route_matched` is `null` for a request that matched no route. |
| `request_id` | The [correlation ID](correlation-id.md) for this request, or `null` if none is attached. |
| `remote_addr`, `host` | Client address and the `Host`/SNI value matched. |
| `method`, `path`, `protocol`, `status` | Request method and path, connection protocol, response status. |
| `bytes_in`, `bytes_out` | Request and response body bytes. |
| `duration_upstream_us`, `duration_flow_us` | Backend time and Flow's own overhead, in microseconds; either can be `null` (a request Flow answered itself has no upstream time). |
| `upstream_addr` | The backend that answered, or `null`. |
| `captures` | Named captures (below); `{}` when none are declared. |
| `req_headers`, `resp_headers`, `querystring` | Capture-all (below); each key appears only when that tier is enabled. |

## Named capture

`capture-headers`/`capture-queries` name specific request headers or query parameters to
include under `captures`, each under the JSON key given by `as`:

```json
"captures":{"request_id":"abc-123","client_version":"4.2.0"}
```

A capture whose header or query parameter was absent on a given request is left out of
`captures` entirely — not written as `null`. A value longer than `max-bytes` (default 256) is
truncated, silently, to that length. Up to 8 captures total (headers and query parameters
combined) per `filter kind="file-log"` instance. Query parameters are matched by exact name
with no percent-decoding — capture a parameter you know is a plain token (an ID, a version
string), not one that may contain `%`-encoded characters.

## Capture-all

For headers you can't name ahead of time — a backend's own response headers vary by
upstream and by request — `capture-all` captures every request header, every response
header, or the whole query string, up to a byte budget per tier:

```kdl
filter kind="file-log" destination="access" \
    capture-all-request-headers=true capture-all-request-headers-max-bytes=2048 \
    capture-all-response-headers=true capture-all-response-headers-max-bytes=2048 \
    capture-all-querystring=true capture-all-querystring-max-bytes=512
```

| Attribute | Default | Meaning |
|---|---|---|
| `capture-all-request-headers` | `false` | Capture every request header into `req_headers`. |
| `capture-all-request-headers-max-bytes` | 2048 | Byte budget for the tier above. |
| `capture-all-response-headers` | `false` | Capture every response header into `resp_headers` — covers a backend's response and a response Flow generated itself (a rate limit, a redirect, ...) alike. |
| `capture-all-response-headers-max-bytes` | 2048 | Byte budget for the tier above. |
| `capture-all-querystring` | `false` | Capture every query parameter into `querystring`. |
| `capture-all-querystring-max-bytes` | 512 | Byte budget for the tier above. |
| `capture-all-pool-size` | 256 | Pre-allocated buffers reused across requests on a filter with at least one tier enabled, instead of allocating one per request. |

Each enabled tier appears as its own object, only when enabled — `req_headers`, `resp_headers`,
`querystring`. Stops adding headers once a tier's budget is reached (the rest of that tier's
headers for this request are left out, in header order — not retried smaller); a request on a
route where every tier is disabled pays nothing for this feature at all.

Buffers are drawn from a small, fixed-size pool instead of allocating fresh on every request;
if every pooled buffer is in use, Flow falls back to a one-off allocation for that request
rather than skipping capture-all for it — tracked by
`flow_file_log_capture_all_pool_exhausted_total`, below.

Named capture (`captures`) and capture-all can both be enabled on the same filter — named
capture for the things you always want under a short, stable key; capture-all for everything
else.

## Reliability

### Buffering and flush triggers

Each destination has its own bounded queue (`queue-capacity`) and double-buffered in-memory
accumulation (`buffer-bytes` each): one buffer fills while the other, once full, flushes in
the background — so a flush in progress doesn't by itself stall new records from being
accepted. A buffer flushes on whichever of three triggers comes first: filling up,
`flush-interval-ms` elapsing since its first record, or `inactive-timeout-ms` elapsing since
the *most recent* one — the last one flushes a batch's tail promptly once traffic goes quiet,
rather than waiting out the rest of `flush-interval-ms`'s window. A request is never blocked by
logging. The one case the writer thread can still briefly block on is both buffers being full
and flushing at once (the second one slower than a full refill of the first) — rare, since
local writes are normally far faster than that.

**Tuning `inactive-timeout-ms` for low-throughput destinations.** Because it resets on every
record, a destination whose records arrive on average slower than `inactive-timeout-ms` apart
(the default, 200ms, is under roughly 5 requests/second) will flush after nearly every single
record instead of an amortized batch — one disk write per record, not a batch of them. That's
not a malfunction; it's the setting doing what it's configured to do. For a destination that
sees that little traffic and cares more about write amortization than how quickly the last
record of a quiet period reaches disk, raise `inactive-timeout-ms` well above the expected gap
between records, or set it close to `flush-interval-ms` to effectively disable it.

### Queue overflow

What happens when the queue itself is full is `overflow-policy`'s choice:

| Value | When the queue is full and a new record arrives |
|---|---|
| `"reject-newest"` (default) | The new record is dropped — the existing backlog is kept intact, oldest first. |
| `"evict-oldest"` | The newest activity is kept at the cost of the backlog's own history. Not one record evicted per push — once occupancy reaches `evict-high-watermark` (a fraction of `queue-capacity`, default `0.9`), a whole batch of the oldest records is dropped in one pass, down to `evict-low-watermark` (default `0.7`), so most pushes between batches pay no eviction cost at all. Takes a brief lock on every push (`reject-newest` stays lock-free) — the trade-off of choosing this policy. |

Either way, every lost record is counted (`flow_file_log_dropped_records_total`), never waited
for.

### Capacity warnings

An operator can watch a destination's queue filling up before it starts dropping:
`capacity-warning-threshold` (a fraction of `queue-capacity`, default `0.8`) logs a `WARN` once
when occupancy crosses it and an `INFO` once when it drops back below — never more than once
per crossing, and never a drop itself. Unlike `high-watermark` (see
[Adaptive sampling](#adaptive-sampling)), crossing this never changes what gets logged; it's a
heads-up, not a policy change. `flow_file_log_capacity_warning_active{destination}` (0/1) tracks
the same crossing on the [Prometheus metrics endpoint](../metrics.md).

### Write failures: retry and circuit breaker

A write that fails (not one that hangs) is retried up to `max-retries` times. If it's still
failing, the destination's writer stops draining new records — they queue up behind the bound
above — and probes the destination periodically (every `retry-delay-ms`) until it recovers,
at which point normal writing resumes automatically. `flow_file_log_circuit_open` is `1` while
a destination is in this state.

### Write hangs: the watchdog pool

A write that hangs rather than fails — a stuck network filesystem mount, for example — is
watched by `write-timeout-ms`. If a write hasn't returned within that time, Flow races it with
a second, independent attempt (its own fresh file handle, not the stuck one) rather than
waiting indefinitely; whichever finishes first is used. `io-pool-size` (default `2`) bounds how
many such attempts can ever be concurrently stuck for one destination — the primary attempt
counts toward this too, so the default means "the primary, plus one fresh attempt if it
hangs." Once every slot is permanently used up by attempts that never returned, further writes
to that destination go back to the plain, unwatched behavior (no hang protection left to give).

**One trade-off this can't fully avoid**: a "stuck" attempt that's merely slow, not actually
dead, can still succeed *after* a fresh attempt already recovered the destination — and
because both attempts hold the same content, the destination can end up with that batch
written twice. A blocked write can't be cancelled once issued, so this can't be eliminated,
only bounded and surfaced: Flow logs it plainly (`app-log`, `WARN`) whenever it happens, rather
than letting a duplicate line appear with no trace of why. In practice this needs a write that
recovers *after* a timeout but isn't permanently hung — rare, and bounded by how often writes
actually stall at all.

`flow_file_log_io_pool_active{destination}` (a gauge, 0 to `io-pool-size`) tracks how many
attempts are currently in flight for a destination on the
[Prometheus metrics endpoint](../metrics.md) — reaching `io-pool-size` means at least one
attempt is permanently stuck.

### Shutdown

A graceful shutdown flushes every destination before Flow exits — queued and buffered records
are written out, not lost; a buffer already flushing in the background is given up to
`write-timeout-ms` to finish before shutdown proceeds. An abrupt kill (a crash, `SIGKILL`, a
power loss) still loses whatever hadn't reached disk yet, since nothing can run after that. The
one exception even on a graceful shutdown: a destination whose circuit is already open (see
above) can't be flushed — there's nothing to deliver to, by definition — so whatever it's
holding is lost in that case either way.

## Sampling

```kdl
filter kind="file-log" sampling="errors-only"
```

| Value | Writes a record for |
|---|---|
| `"all"` (default) | Every request. |
| `"errors-only"` | A response with status `400` and above, and a request Flow never got any response for at all (no matching route, a failed upstream connection) — not a `2xx`/`3xx`. |
| `"slow-only"` | A request whose total duration is at or above `slow-threshold-ms` (default `1000`). On the rare request Flow has no duration for at all, treated as not slow, not logged. |
| `"sample-rate"` | Roughly 1 in N requests, where N is derived from `sample-rate` (a fraction over `0` and at most `1` — e.g. `0.1` logs about 10%). Deterministic (every Nth request by a per-filter counter), not a random draw — not something to rely on to pick specific requests. |

```kdl
filter kind="file-log" sampling="slow-only" slow-threshold-ms=500
filter kind="file-log" sampling="sample-rate" sample-rate=0.1
```

A request excluded by `sampling` is not counted anywhere (not as a drop — `sampling` is a
policy decision, not loss) and costs nothing beyond reading the response status (and, for
`"slow-only"`, the request's own duration, already computed for other purposes; for
`"sample-rate"`, one atomic counter increment): no capture, no allocation, nothing pushed to
the writer thread.

A service-level and a route-level `file-log` filter can use different `sampling` values on the
same request — both still run independently (see [Usage](#usage)): a service-wide
`sampling="all"` catch-all and a route's own `sampling="errors-only"` copy, writing a second
record only for that route's errors. Each filter instance keeps its own `sample-rate` counter —
a service-level and route-level filter counting independently is expected, not a bug.

## Adaptive sampling

Two independent conditions force every `filter kind="file-log"` pointing at a destination into
`sampling="errors-only"`, regardless of what each one is individually configured with:

- The destination's circuit is open (see [Write failures: retry and circuit breaker](#write-failures-retry-and-circuit-breaker)) — it's actively failing
  writes.
- The destination's queue occupancy is at or above `high-watermark` (a fraction of
  `queue-capacity`, default `0.9`) — it's falling behind, whether or not any write has failed.

Either condition holds for `overflow-cooldown-ms` after it clears, so a brief recovery doesn't
immediately flap back. A destination that's failing or backing up is exactly when flooding it
with full-fidelity logging matters least and a backlog matters most; this tightens
automatically and reverts automatically once both conditions have been clear for the whole
cooldown window.

`flow_file_log_adaptive_sampling_active{destination}` (0/1) tracks the combined state;
`flow_file_log_queue_pressure_active{destination}` (0/1) tracks the queue-occupancy condition
on its own, so an operator can tell traffic-volume pressure apart from a destination-specific
failure. Both are on the [Prometheus metrics endpoint](../metrics.md).

## Refused when the configuration loads

- `file-log-destination` without `name`, or two destinations sharing a `name`.
- `output="file"` without `path`, or `path` set with any other `output`.
- An unknown attribute on `file-log-destination`, or on `filter kind="file-log"` itself.
- `filter kind="file-log" destination="..."` naming a destination that isn't declared.
- A combined `capture-headers`/`capture-queries` count over 8, or a `max-bytes` over 256.
- A `capture-headers`/`capture-queries` entry missing `name` or `as`.
- A `capture-all-*-max-bytes` attribute that isn't a positive integer, or a `capture-all-*`
  toggle set to anything other than `true`/`false`.
- `sampling` set to anything other than `"all"`, `"errors-only"`, `"slow-only"`, or
  `"sample-rate"`.
- `slow-threshold-ms` that isn't a positive integer.
- `sampling="sample-rate"` without a `sample-rate` attribute, or one that isn't a number over
  `0` and at most `1`.
- `queue-capacity=0` — a zero-capacity queue can't buffer any records.
- `high-watermark` set to `0` or above `1`.
- `capacity-warning-threshold` set to `0` or above `1`.
- `overflow-policy` set to anything other than `"reject-newest"` or `"evict-oldest"`.
- `evict-high-watermark` or `evict-low-watermark` set while `overflow-policy` isn't
  `"evict-oldest"` — they have no effect under `"reject-newest"`, the default.
- `evict-high-watermark` or `evict-low-watermark` set to `0` or above `1`, or
  `evict-low-watermark` not less than `evict-high-watermark` (after defaults are applied, if
  only one of the pair is set explicitly).

Flow starts with a warning when:

- `system.logging.app-log` and a `file-log-destination` both write to the same standard
  stream (`stdout` or `stderr`) — their lines interleave, one human-readable, one JSON, with
  no separator.
- A declared `file-log-destination` is never referenced by any `filter kind="file-log"`.

## Observability

`flow_file_log_dropped_records_total{destination}` (a counter),
`flow_file_log_circuit_open{destination}` (0/1),
`flow_file_log_capture_all_pool_exhausted_total{destination}` (a counter),
`flow_file_log_adaptive_sampling_active{destination}` (0/1),
`flow_file_log_queue_pressure_active{destination}` (0/1),
`flow_file_log_capacity_warning_active{destination}` (0/1), and
`flow_file_log_io_pool_active{destination}` (a gauge, 0 to `io-pool-size`) are available on the
[Prometheus metrics endpoint](../metrics.md) when it's configured, labeled by destination
(the output stream name, or the file path).
