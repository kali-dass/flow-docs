# Getting Started

> **New here?** You're in the right place
> **Want more context?** See [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)

This guide takes you from an empty directory to a working proxy in a few minutes.

## Prerequisites

- The `flow` binary
- A backend service to proxy to — anything that answers HTTP will do

Check that the binary runs:

```bash
flow --help
```

## Your first config

Flow is configured with a single [KDL](https://kdl.dev) file. Create `myconfig.kdl`:

```kdl
system {
    threads-per-service 2
}

services {
    MyGateway {
        listeners {
            "0.0.0.0:8080"
        }

        routes {
            route {
                match {
                    path-prefix "/"
                }
                allowed-methods "GET" "POST"
                allowed-protocols "http"

                connectors {
                    load-balance {
                        selection "RoundRobin"
                    }
                    "127.0.0.1:3000"
                }
            }
        }
    }
}
```

This listens on port 8080 in plain HTTP and forwards everything to a backend on port 3000.

## Check the config before you run it

Flow can validate a config and exit without binding any ports:

```bash
flow --config-kdl myconfig.kdl --validate-configs
```

If the config is malformed, Flow prints the offending line with a caret pointing at the
problem. Fix it and re-run. This is worth doing in CI.

## Run it

```bash
flow --config-kdl myconfig.kdl
```

Then send a request:

```bash
curl -v http://localhost:8080/hello
```

You should see the response from your backend on port 3000.

## Add TLS

To terminate HTTPS, give the listener a certificate and key. To also offer HTTP/2, set
`offer-h2=true` (HTTP/2 on the listener requires TLS):

```kdl
listeners {
    "0.0.0.0:8080"
    "0.0.0.0:8443" cert-path="./certs/test.crt" key-path="./certs/test.key" offer-h2=true
}
```

Update the route to accept the new protocols:

```kdl
allowed-protocols "http" "https" "h1" "h2"
```

Need a self-signed certificate for testing?

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout certs/test.key -out certs/test.crt -days 365 \
  -subj "/CN=localhost"
```

Then:

```bash
curl -kv https://localhost:8443/hello
```

(`-k` tells curl to accept the self-signed certificate.)

## Route to more than one backend

List several connectors and Flow will load balance across them:

```kdl
connectors {
    load-balance {
        selection "RoundRobin"
    }
    "10.0.0.1:3000"
    "10.0.0.2:3000"
    "10.0.0.3:3000"
}
```

## Route by path

Routes are matched most-specific-first, so you can layer them:

```kdl
routes {
    route {
        match { path-prefix "/api" }
        allowed-methods "GET" "POST"
        allowed-protocols "http"
        connectors { "10.0.0.1:3000" }
    }
    route {
        match { path-prefix "/" }
        allowed-methods "GET"
        allowed-protocols "http"
        connectors { "10.0.0.9:8080" }
    }
}
```

Requests to `/api/users` hit the first route (longer prefix wins); everything else falls
through to the second.

> **Careful:** all routes must live inside a **single** `routes { }` block. If you write
> two separate `routes { }` blocks, only the first is used and the rest are silently
> ignored.

## Where to go next

- [Configuration](configuration.md) — every option, including upstream TLS and tuning
- [Routing](routing.md) — header matching, SNI, regex, and how ties are scored
- [Policies](policies.md) — rate limiting, IP blocking, header rewriting
- [Performance](performance.md) — how fast Flow goes, and how to keep it there
