# mcp-cluster-health-monitor

A health canary for a fleet of AtScale clusters: it reads a cluster registry, TCP-probes every cluster at once, and prints one JSON report with an exit code you can gate on.

## Why it exists

When a gateway fronts several AtScale clusters, the first question on any incident is the dullest one: which clusters are actually reachable right now? Answering it by hand — one `nc` or `psql` per endpoint — is slow and easy to get half-right. This tool turns that question into a single command that returns a machine-readable answer and an exit code, so a cron job, a deploy gate, or a person at 3am all get the same verdict.

The probe is deliberately shallow. It opens a TCP connection to each cluster's endpoint and reports whether the connection succeeds, times out, or is refused. That is enough to distinguish "the cluster is gone" from "the cluster is up," which is the distinction that matters most in the first minute of an incident. It does not authenticate, and it does not run a query — see [What it does not do](#what-it-does-not-do).

## Install

Build from source with Cargo. The crate depends on a sibling crate, [`mcp-cluster-registry`](https://github.com/joeyen-atscale/mcp-cluster-registry), via a relative path, so clone both into the same parent directory:

```
git clone https://github.com/joeyen-atscale/mcp-cluster-registry
git clone https://github.com/joeyen-atscale/mcp-cluster-health-monitor
cd mcp-cluster-health-monitor
cargo build --release
# binary at target/release/mcp-cluster-health-monitor
```

Requires Rust 1.70 or newer.

## Quickstart

Write a registry listing the clusters to watch:

```toml
[[clusters]]
name = "prod"
endpoint = "mcp-aws.atscaleinternal.com:15432"
required = true
priority = 1
supported_backends = ["sql"]

[clusters.auth]
type = "direct"
pg_user = "PG_USER"
pg_pass_env = "PG_PASS"

[[clusters]]
name = "staging"
endpoint = "mcp-staging.atscaleinternal.com:15432"
required = false
priority = 2
supported_backends = ["sql", "dax"]

[clusters.auth]
type = "oidc"
token_url = "https://auth.example.com/token"
client_id = "my-client"
realm = "atscale"
client_secret_env = "OIDC_SECRET"
```

Run the probe:

```
mcp-cluster-health-monitor --registry registry.toml --timeout-ms 2000
```

If `prod` is reachable and `staging` is refusing connections, you get:

```json
{
  "timestamp_ms": 1749432000000,
  "overall": "degraded",
  "clusters": [
    {
      "name": "prod",
      "status": "healthy",
      "latency_ms": 12
    },
    {
      "name": "staging",
      "status": "unhealthy",
      "error": "Connection refused (os error 111)"
    }
  ]
}
```

`staging` is optional (`required = false`), so the fleet is `degraded`, not `critical`, and the process exits `1`. `latency_ms` and `error` are omitted when they don't apply.

For a quick eyeball during an incident, `--format human` prints one line per cluster instead of JSON.

## Flags

| Flag | Default | Meaning |
|---|---|---|
| `--registry <PATH>` | `registry.toml` | Registry file. Parsed as JSON if the extension is `.json`, otherwise as TOML. |
| `--timeout-ms <MS>` | — | Per-cluster connect timeout in milliseconds. Overrides `--timeout-secs`. |
| `--timeout-secs <S>` | `5` | Per-cluster connect timeout in seconds. Kept for compatibility; prefer `--timeout-ms`. |
| `--format <FMT>` | `json` | `json` or `human`. |
| `--cluster <NAME>` | — | Probe only this cluster. Repeat the flag to probe a named subset. |

## Statuses and exit codes

Each cluster lands in one of three states, from the outcome of a single TCP connect:

| Status | Meaning |
|---|---|
| `healthy` | Connect succeeded within the timeout; `latency_ms` is reported. |
| `unreachable` | Connect timed out. |
| `unhealthy` | Connect failed before the timeout — refused, reset, or another connection error. The OS error is in `error`. |

The fleet's `overall` status rolls those up against each cluster's `required` flag:

| Overall | Condition |
|---|---|
| `healthy` | Every cluster is `healthy`. |
| `degraded` | Every `required` cluster is `healthy`; at least one optional cluster is not. |
| `critical` | At least one `required` cluster is not `healthy`. |

The exit code is meant to be gated on by a caller:

| Code | Meaning |
|---|---|
| `0` | `overall` is `healthy`. |
| `1` | `overall` is `degraded` or `critical`. |
| `2` | Registry could not be read or parsed, or no cluster matched `--cluster`. |

## What it does not do

The probe is TCP-connect only. It does **not** authenticate to a cluster, and it does **not** run a query against any backend. The `auth`, `supported_backends`, and `priority` fields are read from the registry — they're part of the shared registry schema — but this tool does not act on them. A cluster that completes a TCP handshake is reported `healthy` even if it would reject every query.

Backend round-trips — actually issuing a DAX or MDX query and confirming a result comes back — are a deeper level of health that this tool leaves to a future iteration.

## Where it fits

Part of the AtScale federation gateway toolchain, alongside [`mcp-cluster-registry`](https://github.com/joeyen-atscale/mcp-cluster-registry) (the typed registry this tool reads), [`mcp-cross-cluster-diff`](https://github.com/joeyen-atscale/mcp-cross-cluster-diff), and [`mcp-federated-catalog`](https://github.com/joeyen-atscale/mcp-federated-catalog). The registry is the shared definition of what the fleet is; this is the tool that asks whether the fleet is up.

## License

MIT OR Apache-2.0
