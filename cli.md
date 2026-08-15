# Command-Line Reference

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Making a decision?** See [Design Decisions](design-decisions.md)
> **Understanding limitations?** See [Limitations](limitations.md)
> **Looking for something specific?** [Configuration Reference](configuration.md) • [Operations](operations.md) • [Performance](performance.md)

Flow is started from the command line and configured from a KDL file. The command line
handles *how this process runs*; the config file describes *what the proxy does*.

```bash
flow --config-kdl /etc/flow/config.kdl
```

```
Options:
      --config-kdl <CONFIG_KDL>                  Path to the configuration file in KDL format
      --validate-configs                         Validate all configuration data and exit
      --threads-per-service <THREADS_PER_SERVICE>  Worker threads per service
      --daemonize                                Run in the background after starting
      --upgrade                                  Take over an existing server's listeners
      --upgrade-socket <UPGRADE_SOCKET>          Path to the upgrade socket
      --pidfile <PIDFILE>                        Path to the pidfile, used for upgrade
  -h, --help                                     Print help
```

---

## `--config-kdl <PATH>`

The configuration file to load. This is the flag you will use every time.

```bash
flow --config-kdl /etc/flow/config.kdl
```

Flow reads the file, parses it, builds every listener, route, and connector, and starts
serving.

> **If you omit it, Flow starts with an empty default configuration** — no listeners, no
> routes. It will not serve anything. There is no error, only a log line saying
> `No configuration file provided`. If Flow appears to start but nothing responds, check
> that you actually passed this flag.

If the file is missing, malformed, or semantically invalid, Flow **exits immediately** with
an error pointing at the problem. It does not start in a degraded state.

## `--validate-configs`

Parse and check the configuration, then exit — **without binding any ports**.

```bash
flow --config-kdl /etc/flow/config.kdl --validate-configs
```

This runs the entire config pipeline: KDL syntax, every route, every connector attribute,
every filter. If it exits cleanly, the config is loadable.

Use it:

- **In CI**, on every change to a config file.
- **Before a deploy**, on the target host — it catches a bad certificate path or an
  unreadable CA file that a syntax check on your laptop would miss.

This is the cheapest way to avoid a failed rollout.

## `--threads-per-service <N>`

Worker threads for each service.

```bash
flow --config-kdl config.kdl --threads-per-service 4
```

**Set this to your CPU core count.** Flow is CPU-bound; hyperthreading does not help, so
counting hyperthreads rather than physical cores buys you nothing and adds scheduling
overhead.

Normally you set `threads-per-service` in the `system` block of the config file instead.
Use the flag when the same config is deployed to machines with different core counts.

## `--daemonize`

Detach and run in the background.

```bash
flow --config-kdl config.kdl --daemonize --pidfile /var/run/flow.pid
```

**A pidfile is required** when daemonizing — set `--pidfile` or `pid-file` in the config.
The path must be absolute.

If you run under systemd, Docker, or any other supervisor, **do not use this**. Leave Flow
in the foreground and let the supervisor manage the process — that is what it is for.

## `--upgrade`

Take over the listening sockets of an already-running Flow, with no dropped connections.

```bash
flow --config-kdl config.kdl --upgrade
```

The new process connects to the upgrade socket and waits — for about six seconds — for the
old process to hand over its listeners. **It will not receive them until you send `SIGQUIT`
to the old process.** That is the step people miss:

```bash
# 1. new process starts and waits
flow --config-kdl config.kdl --upgrade &

# 2. tell the old one to hand over
kill -SIGQUIT <old_pid>
```

If you see `No incoming socket transfer` repeating, followed by
`Giving up reading socket ... EAGAIN`, you did not send the signal, or you sent it too late.

**Linux only** — the file-descriptor transfer does not work on macOS.

This flag is deliberately **not** settable in the config file: it describes what *this
invocation* should do, not what the service is.

See [Operations](operations.md) for the full procedure.

## `--upgrade-socket <PATH>`

The Unix socket used to pass listening sockets between the old and new process during
`--upgrade`. Must be an absolute path, and **both processes must agree on it**.

Usually set once in the config file's `system` block.

## `--pidfile <PATH>`

Where to write the process ID. Required when `--daemonize` is used. Must be an absolute
path.

---

## How the command line and the config file interact

This is worth understanding, because it is **not** a simple "the CLI wins" rule.

### Values: the CLI replaces the file

`--threads-per-service` overrides whatever the config file says.

### Switches: the CLI can only turn things **on**

`--validate-configs`, `--daemonize`, and `--upgrade` are **combined** with the config file,
not substituted for it.

So if your config file has `daemonize true`, you **cannot** turn it off from the command
line — there is no `--no-daemonize`. The flag can enable a switch, never disable one. To run
in the foreground, remove `daemonize` from the config file.

### Paths: the CLI and the file must **agree**

`--pidfile` and `--upgrade-socket` are the exception, and the behavior is strict:

- Set in **only one** place → that value is used.
- Set in **both**, and they **match** → fine.
- Set in **both**, and they **differ** → **Flow refuses to start**, with a
  `Mismatched commanded PID files` error.

This is deliberate. Two processes disagreeing about where the pidfile or upgrade socket
lives is exactly how an upgrade silently fails to hand over — so Flow treats a conflict as a
fatal error rather than quietly picking one.

**In practice:** define `pid-file` and `upgrade-socket` in the config file, and don't pass
the flags at all.

---

---

## Gotchas

Four behaviors that surprise people. None of them are bugs — but none of them are what you'd
guess either.

### 1. No `--config-kdl` is not an error

```bash
flow          # starts, serves nothing, exits 0 on shutdown
```

Flow does **not** fail when you forget the config flag. It starts with an empty default
configuration — no listeners, no routes — logs `No configuration file provided`, and sits
there. The process is healthy; it just has nothing to do.

So **"Flow is running but nothing responds"** usually means the config flag never arrived —
check your unit file, entrypoint, or wrapper script before you debug the network.

Note the asymmetry: a config path that **exists but is bad** stops Flow immediately with an
error. A config path that is **absent entirely** is silently accepted.

### 2. You cannot turn a switch off from the command line

There is no `--no-daemonize`, and adding one wouldn't help: the CLI switches are **combined**
with the config file (logically OR'd), not substituted for it.

```kdl
system { daemonize true }
```

With that in your config, **every** invocation daemonizes. No flag will stop it. To run in
the foreground, remove the line from the config file.

The rule is: **a switch can be turned on from the CLI, never off.** This applies to
`--daemonize`, `--upgrade`, and `--validate-configs`.

### 3. A pidfile/socket disagreement is fatal, not a precedence contest

```bash
# config.kdl says:  pid-file "/var/run/flow.pid"
flow --config-kdl config.kdl --pidfile /tmp/other.pid
```

This does **not** override. Flow **refuses to start**:

```
Mismatched commanded PID files. CLI: "/tmp/other.pid", Config: "/var/run/flow.pid"
```

The same applies to `--upgrade-socket`. Set them in **one** place — normally the config file
— and don't pass the flags.

This is deliberate. If the old and new process disagree about where the upgrade socket
lives, the handoff silently fails and you drop connections. Flow would rather not start than
let that happen quietly.

### 4. `--upgrade` waits, then gives up

`--upgrade` does not itself perform the upgrade. It starts a process that **waits ~6 seconds**
for the old one to hand over its listeners — and the old one only does that when it receives
`SIGQUIT`.

If you start the new process and do nothing else, you get:

```
ERROR transfer_fd: No incoming socket transfer, sleep 1s and try again
ERROR transfer_fd: Giving up reading socket from: /tmp/flow-upgrade.sock, error: EAGAIN
```

That is not a broken socket — **it means nobody sent the signal.** See
[Operations](operations.md).

---

## Common invocations

```bash
# Normal start
flow --config-kdl /etc/flow/config.kdl

# Check a config without starting (do this in CI)
flow --config-kdl /etc/flow/config.kdl --validate-configs

# Match a 4-core host
flow --config-kdl /etc/flow/config.kdl --threads-per-service 4

# Zero-downtime upgrade (then SIGQUIT the old process)
flow --config-kdl /etc/flow/config.kdl --upgrade &
kill -SIGQUIT <old_pid>
```

## Logging

Flow writes to stdout. Control the level with the `RUST_LOG` environment variable:

```bash
RUST_LOG=info flow --config-kdl config.kdl     # default
RUST_LOG=debug flow --config-kdl config.kdl    # verbose
RUST_LOG=warn flow --config-kdl config.kdl     # quiet
```

At startup Flow logs each service it configures and warns about risky settings — a connector
with certificate verification disabled, a cleartext upstream, or a tuning option set on a
protocol that ignores it. **Read the startup warnings on a new deployment.**
