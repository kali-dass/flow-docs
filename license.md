# Licensing

> **New here?** Start with [Getting Started](getting-started.md) or [Technical Overview](technical-overview.md)
> **Looking for something specific?** [Command line](cli.md) • [Operations](operations.md)

Flow is commercial software. Every instance needs a valid license file to start — this page
explains what that means in practice: where the file goes, what Flow checks, and what happens
if it's missing, invalid, or expired.

## What you need

A **license file** delivered to you (portal download, email, or as part of provisioning). You
don't create or edit this file; you just place it somewhere Flow can find it. Flow checks that
it's genuine and unaltered entirely **offline** — no license server, no network call required.

## Where Flow looks for the license file

Flow checks three places, in this order, and uses the **first one that's set**:

1. **`--license-file <path>`** — an explicit path passed on the command line. Highest
   precedence: if you pass this, it's used, full stop.
2. **`FLOW_LICENSE_FILE`** environment variable — checked if the flag wasn't passed. This is
   the natural choice for containers/orchestrators: set it once in the deployment spec and
   every invocation picks it up without needing the flag repeated.
3. **`./license.json`**, next to wherever Flow is run from — the zero-config fallback if
   neither of the above is set.

```bash
# Explicit flag — wins over everything else
flow --config-kdl config.kdl --license-file /etc/flow/license.json

# Env var — convenient for containers
export FLOW_LICENSE_FILE=/etc/flow/license.json
flow --config-kdl config.kdl

# Neither set — Flow looks for ./license.json in the current directory
flow --config-kdl config.kdl
```

**Why this order:** it's the same convention most CLI tools follow — the thing you typed for
*this specific run* (the flag) beats the thing set once in your environment (the env var),
which beats an implicit convention (the default path). If you're debugging "why isn't my
`--license-file` being picked up," this order is why an env var can never silently win over a
flag you explicitly passed — the flag always takes priority.

## What Flow checks, and in what order

When Flow starts, it validates the license **before opening any listener** — before it does
anything else with your configuration:

1. **The file exists and is readable.** No file at the resolved path → Flow refuses to start.
2. **It's in the expected format.** Malformed or unreadable → refuses to start.
3. **It's a format this Flow build recognizes.** An unrecognized license format → refuses to
   start (this exists so a future format change fails clearly rather than being silently
   misread).
4. **It's genuine and unaltered.** If the file has been changed in any way since it was
   issued — even a single character in the expiration date — Flow detects this and refuses to
   start.
5. **It hasn't expired.** Flow checks the license's expiration date plus its grace period (see
   below). Past that point, Flow refuses to start.

Any failure at any step produces a clear log message explaining which check failed, and Flow
exits without binding a single port. It never starts in a partially-licensed or degraded state.

## Expiration and the grace period

Every license carries its own expiration date and its own **grace period** — a number of days
past expiration during which Flow keeps working normally. This isn't a Flow setting you
configure; it's part of the license itself.

- **Before expiration + grace period:** Flow runs normally.
- **After expiration + grace period, at startup:** Flow refuses to start, same as any other
  invalid license.

> **Current limitation.** Today, Flow only checks the license **at startup**. If a license
> expires *while Flow is already running*, Flow keeps serving traffic until the next
> restart — it does not notice mid-run. A continuous runtime check (so an expired license
> stops accepting *new* requests without waiting for a restart, while letting in-flight
> requests finish normally) is planned but not yet implemented.

## Common problems

**"license file at ... could not be read"** — the file doesn't exist at the path Flow resolved.
Double check which of the three locations above you expected Flow to use, and that the path is
correct for wherever Flow is actually running from (a relative default path depends on the
current working directory).

**"license signature verification FAILED"** — Flow could not confirm the file is genuine and
unaltered. This happens if the file was edited, reformatted, corrupted in transit, or
truncated — even a change that looks harmless (like re-saving it with different spacing) can
trigger this. Get a fresh copy of the file rather than trying to fix it by hand.

**"license expired on ..."** — self-explanatory. Contact whoever issued your license for a
renewal.

**Flow starts but a fresh copy of the license doesn't seem to help** — check you're not
accidentally overriding the file location with a stale `FLOW_LICENSE_FILE` env var or
`--license-file` flag pointing somewhere other than where you just updated the file.

## Related

- [Command line](cli.md#--license-file-path) — the `--license-file` flag reference
- [Operations](operations.md) — running Flow in production, including license file placement
  alongside other deployment concerns
- [Getting Started](getting-started.md) — the license file as part of your first run
