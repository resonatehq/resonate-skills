---
name: resonate-cli
description: Drive the Resonate server from the shell — start a server (`serve` / `dev`), create/resolve/search Durable Promises, create and delete cron schedules, manually invoke a function via `invoke`, walk a call-graph with `tree`, and inspect or recover tasks (`tasks`). Reach for this when the agent needs to poke a running Resonate server without writing SDK code: unblocking a stuck human-in-the-loop promise, triggering a workflow by hand to reproduce a bug, listing pending promises by tag, registering a one-off cron, or smoke-testing a deploy. Covers global flags, every subcommand and its flags as of `resonate 0.9.7`, configuration layering (`resonate.toml` + `RESONATE_*` env vars + flags), common recipes, and the small set of docs-vs-binary deltas an agent will trip on.
license: Apache-2.0
---

# resonate-cli

## Overview

The `resonate` binary is one program with three jobs:

1. **Run the Resonate server** — `resonate serve` (persistent storage) and `resonate dev` (in-memory, for development).
2. **Talk to a running server over HTTP** — every subcommand under `resonate promises`, `resonate schedules`, `resonate tasks`, plus the top-level `resonate invoke` and `resonate tree`.
3. **Expose an MCP shim** — `resonate mcp` is a stdio MCP server that wraps the same HTTP API for skill-aware agents (see `resonate-bash` for the durable-bash tool that ships through this shim).

This skill is the agent's reference for #2 and #3 — when you need to interact with a Resonate server from a shell. For long-form server deployment, see [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) and [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md).

The CLI is the source of truth. The hosted docs occasionally drift — see [Docs-vs-binary deltas](#docs-vs-binary-deltas) below. When in doubt: `resonate <cmd> --help`.

## When to use this skill

Reach for the CLI when **any** of the following is true:

- You need to resolve, reject, or cancel a Durable Promise from outside any worker — typically to unblock a human-in-the-loop workflow, simulate a webhook callback, or recover a stuck process.
- You need to inspect server state — `promises get`, `promises search`, `schedules search`, `tasks search`, `tree`.
- You need to trigger a function manually — `invoke` creates the promise and routes it to a worker target, no client SDK required.
- You need to register or remove a cron schedule.
- You're scripting a smoke test against a running server.

Prefer the SDK over the CLI in application code. The CLI is for operators, debuggers, and one-off tooling. Anything you'd commit as a workflow step should live in the SDK.

## Installation and version check

```shell
brew install resonatehq/tap/resonate   # macOS
resonate --version                     # expect 0.9.7 or newer
```

Linux: download from [GitHub releases](https://github.com/resonatehq/resonate/releases) and put the binary on `$PATH`.

All command surface in this skill is from `resonate 0.9.7`. Use `resonate <cmd> --help` to confirm against your installed version — the built-in help is always authoritative.

## Top-level command map

```
resonate
├── serve       Start the production server (persistent storage)
├── dev         Start the development server (in-memory)
├── promises    Promise operations (CRUD + search + callbacks/listeners)
├── tasks       Task operations (low-level; usually SDK-owned)
├── schedules   Schedule operations (cron-driven promise creation)
├── invoke      Invoke a function via a durable promise
├── tree        Display the call-graph tree rooted at a promise ID
└── mcp         Start the Resonate MCP server (stdio transport)
```

## Global flags

Every client subcommand (`promises`, `tasks`, `schedules`, `invoke`, `tree`, `mcp`) accepts:

| Flag | Description | Default |
|---|---|---|
| `-S`, `--server <URL>` | Resonate server URL | `http://localhost:8001` |
| `-T`, `--token <TOKEN>` | JWT bearer token for authenticated servers | — |
| `-h`, `--help` | Print help for this command | — |

`resonate --version` and `resonate --help` work at the top level.

There is **no** `--output json` flag in `resonate 0.9.7`, despite older docs mentioning one. Subcommand stdout is human-formatted; if you need structured output, call the HTTP API directly with `curl`.

## Server commands

### `resonate dev` — in-memory development server

```shell
resonate dev
```

Storage defaults to `:memory:` (SQLite in-RAM), so every restart is a clean slate. Useful for local development and tests.

Listens on:
- HTTP API: `http://localhost:8001`
- Prometheus metrics: `http://localhost:9090/metrics`
- Health probe: `http://localhost:8001/health` (200 when healthy — note: it is `/health`, not `/healthz`)

Common flags (full set: `resonate dev --help`):

| Flag | Default | Notes |
|---|---|---|
| `--server-port <PORT>` | `8001` | HTTP API port |
| `--server-host <HOST>` | `localhost` | Bind host |
| `--server-cors-allow-origin <ORIGIN>` | — | Repeatable; `*` for permissive |
| `--storage-sqlite-path <PATH>` | `:memory:` | Override for an on-disk SQLite file even in dev |
| `--transports-bash-exec-enabled <BOOL>` | `false` | Enable the `bash://` transport used by the [`resonate-bash`](../resonate-bash/SKILL.md) MCP tool |
| `--observability-metrics-port <PORT>` | `9090` | `0` disables the metrics endpoint |
| `--level <LEVEL>` | `info` | `debug`, `info`, `warn`, `error` |

`--transports-bash-exec-enabled` (and every `--transports-*-enabled` flag) requires an explicit `true` / `false`. A bare flag errors with `a value is required for '--transports-bash-exec-enabled <BOOL>'`.

### `resonate serve` — production server

```shell
export RESONATE_STORAGE__POSTGRES__URL="$DATABASE_URL"   # e.g. postgres://<user>:<password>@<host>:5432/resonate
resonate serve --storage-type postgres
```

Same flag surface as `dev` plus storage and auth options. The full reference is `resonate serve --help`; the highlights:

| Flag | Default | Notes |
|---|---|---|
| `--storage-type <sqlite\|postgres>` | `sqlite` | **Must be set to `postgres` explicitly** — passing only `--storage-postgres-url` silently leaves the server on SQLite |
| `--storage-postgres-url <URL>` | — | Required when `--storage-type postgres` |
| `--storage-postgres-pool-size <N>` | `10` | |
| `--storage-mysql-url <URL>` | — | MySQL is also supported |
| `--storage-mysql-pool-size <N>` | `10` | |
| `--auth-publickey <KEY>` | — | Path to PEM public key; enables JWT auth. Use the literal string `none` for unsigned mode |
| `--auth-iss <ISS>` | — | Required JWT `iss` claim |
| `--auth-aud <AUD>` | — | Required JWT `aud` claim |
| `--tasks-lease-timeout <MS>` | `15000` | How long a claimed task lease holds before redispatch |
| `--tasks-retry-timeout <MS>` | `30000` | Pending-task retry interval |
| `--transports-http-push-enabled <BOOL>` | `true` | Outbound webhook delivery |
| `--transports-http-push-auth-mode <none\|bearer\|gcp>` | `none` | Set to `gcp` for Cloud Run ID-token auth |
| `--transports-http-poll-enabled <BOOL>` | `true` | SSE long-poll transport |
| `--transports-gcps-enabled <BOOL>` | `false` | GCP Pub/Sub transport (`--transports-gcps-project` required) |
| `--transports-bash-exec-enabled <BOOL>` | `false` | See [`resonate-bash`](../resonate-bash/SKILL.md) |
| `--observability-otlp-endpoint <ENDPOINT>` | `localhost:4317` | OpenTelemetry OTLP target |
| `--storage-sqlite-path <PATH>` | `resonate.db` | On-disk by default in `serve` (`:memory:` in `dev`) |

**Storage gotcha:** `--storage-type` defaults to `sqlite`. If you set `--storage-postgres-url` alone, the server boots on SQLite and silently ignores the URL. Always pair them.

**Health route:** `/health` returns 200 when the server is up. `/healthz` does not exist on Resonate — don't wire that into your readiness probes.

## `resonate promises`

The most common subcommand tree. Every promise command takes `<ID>` as the trailing arg (or two IDs for callbacks/listeners) and respects the global `-S` / `-T` flags.

### `promises create <ID> --timeout <DURATION>`

```shell
resonate promises create approval-456 --timeout 24h
```

| Flag | Description |
|---|---|
| `--timeout <DURATION>` *(required)* | Promise deadline relative to now, e.g. `30s`, `5m`, `1h`, `24h` |
| `--param <JSON>` | Promise parameter, JSON-encoded: `--param '{"data":"hello"}'` |
| `--tags <JSON>` | Tag object: `--tags '{"project":"checkout"}'`. Used by `search --tags` later |

### `promises get <ID>`

```shell
resonate promises get order-123
```

Returns state, timestamps, param, value, and tags.

### `promises resolve <ID> [--value <JSON>]`

```shell
resonate promises resolve approval-456 --value '{"approved": true}'
```

The flag is `--value`, **not** `--data` (older docs sometimes show `--data` — the binary doesn't accept it).

### `promises reject <ID> [--value <JSON>]`

Same shape as `resolve` — the JSON ends up in the `value` field with promise state `rejected`.

### `promises cancel <ID> [--value <JSON>]`

Same shape; promise state becomes `rejected_canceled`.

### `promises search`

```shell
resonate promises search --state pending --limit 20
resonate promises search --tags '{"project":"checkout"}' --limit 50
```

| Flag | Description |
|---|---|
| `--state <STATE>` | One of `pending`, `resolved`, `rejected`, `rejected_canceled`, `rejected_timedout` |
| `--tags <JSON>` | Match on tag subset, JSON object |
| `--limit <N>` | Default `100` |
| `--cursor <CURSOR>` | Pagination cursor returned by the previous page |

`promises search` does **not** take a positional ID pattern — there is no `search "order-*"` form in `resonate 0.9.7`. Filter via `--tags` or paginate `--state` results client-side.

### `promises register-callback <AWAITED> <AWAITER>`

Wires a server-side callback so that resolving `AWAITED` notifies the `AWAITER` promise. Rarely used from the CLI — most callers register callbacks via the SDK.

### `promises register-listener <AWAITED> <ADDRESS>`

Registers a delivery address (e.g. `http://my-host/cb` or `poll://group@worker`) to be notified on settlement of `AWAITED`. Useful for ad-hoc webhook wiring during debugging.

## `resonate schedules`

Cron-driven promise creation. Every cron tick stamps out a new promise from a template.

### `schedules create <ID> --cron <EXPR> --promise-id <TEMPLATE> --promise-timeout <DURATION>`

```shell
resonate schedules create hourly-sync \
  --cron "0 * * * *" \
  --promise-id "sync-{{.timestamp}}" \
  --promise-timeout 30m \
  --promise-param '{"job":"sync"}' \
  --promise-tags '{"system":"etl"}'
```

| Flag | Required | Description |
|---|---|---|
| `--cron <EXPR>` | yes | Standard 5-field cron, e.g. `"0 * * * *"` for hourly |
| `--promise-id <TEMPLATE>` | yes | Promise ID template. `{{.timestamp}}` interpolates the tick's ms-since-epoch — use it to keep each tick's promise ID unique |
| `--promise-timeout <DURATION>` | yes | Per-tick promise deadline, e.g. `30m`, `1h` |
| `--promise-param <JSON>` | no | Param attached to every generated promise |
| `--promise-tags <JSON>` | no | Tags attached to every generated promise |

`--description` exists in older docs but not in `resonate 0.9.7`. Use `--promise-tags` to label.

### `schedules get <ID>` / `schedules search` / `schedules delete <ID>`

```shell
resonate schedules search --limit 50
resonate schedules get hourly-sync
resonate schedules delete hourly-sync
```

`schedules search` accepts `--tags` and `--limit`/`--cursor`. There is no positional pattern arg — same shape as `promises search`.

## `resonate invoke`

```shell
resonate invoke <PROMISE_ID> --func <FUNC_NAME> [--arg <ARG> | --json-args <JSON_ARRAY>]
```

Creates a Durable Promise and routes it to a worker that has registered `FUNC_NAME`. The agent uses this to trigger workflows manually — for reproducing a bug, smoke-testing a worker after deploy, or kicking off one-off jobs.

| Flag | Default | Description |
|---|---|---|
| `--func <FUNC_NAME>` *(required)* | — | Name the worker registered the function under |
| `--arg <ARG>` | — | Single argument; repeatable. Each value is a JSON scalar/object or a plain string |
| `--json-args <JSON_ARRAY>` | — | All arguments as a single JSON array (mutually exclusive with `--arg`) |
| `--version <N>` | `1` | Function version, when the worker registers multiple |
| `-t`, `--timeout <DURATION>` | `1h` | Promise timeout |
| `--target <TARGET>` | `poll://any@default` | Worker target address. Use `poll://any@<group>` to route to a specific worker group |
| `--delay <DURATION>` | — | Delay first delivery, e.g. `5s`, `1m` |

```shell
# Single arg, default worker group
resonate invoke order-123 --func processOrder --arg '{"orderId":"123"}'

# Multiple args
resonate invoke checkout-7 --func charge --arg '"customer_42"' --arg '99.50'

# JSON-array form
resonate invoke checkout-7 --func charge --json-args '["customer_42", 99.50]'

# Route to a specific worker group
resonate invoke daily-report --func runReport --arg '{}' --target poll://any@reports

# Delayed kickoff
resonate invoke recheck-cdn --func ping --arg '"edge-1"' --delay 5m
```

**Default target.** `poll://any@default` routes to any worker polling the `default` group. If your worker polls a non-default group, override with `--target poll://any@<group>` or the invoke sits pending forever. The same trap bites `ctx.detached()` / `ctx.Detached()` in the TypeScript and Go SDKs — both call into the same routing layer.

## `resonate tree`

```shell
resonate tree <ROOT_PROMISE_ID>
```

Prints the call-graph tree rooted at the given promise. Use this when a workflow is stuck and you want to see which child promise is blocking. No flags beyond `-S` / `-T`.

## `resonate tasks` (low-level)

Tasks are the leases workers claim to execute promises. The CLI exposes the full task lifecycle (`get`, `create`, `claim`, `fence`, `heartbeat`, `suspend`, `complete`, `release`, `halt`, `continue`, `search`), but **agents writing application code should not touch these** — the SDK owns task lifecycle. The two cases where the CLI task surface is useful:

1. **Diagnostics.** `resonate tasks search --state <STATE>` to inspect what's claimed / pending / suspended; `resonate tasks get <ID>` to look at one task in detail.
2. **Manual recovery.** `resonate tasks release <ID> --version <N>` to release a leaked lease back to pending so a fresh worker can claim it. `--version` is the optimistic-concurrency version returned by `tasks get`.

The full surface, for completeness:

| Subcommand | Required flags | Use |
|---|---|---|
| `get <ID>` | — | Read one task |
| `create <ID>` | `--pid`, `--ttl`, `--timeout` | Create a promise+task atomically (task starts acquired). Rare; SDK does this implicitly |
| `claim <ID>` | `--pid`, `--version`, `--ttl` | Claim a pending task. SDK-owned |
| `fence <ID>` | `--version`, `--action` | Run a fenced inner action (`--action` is a JSON envelope, e.g. `{"kind":"promise.create","data":{…}}`). SDK-owned |
| `heartbeat` | `--pid`, `--tasks` | Extend leases on multiple tasks: `--tasks '[{"id":"foo","version":1}]'`. SDK-owned |
| `suspend <ID>` | `--version`, `--actions` | Park a task while awaiting child promises. SDK-owned |
| `complete <ID>` | `--version`, `--action` | Settle the task's promise. SDK-owned |
| `release <ID>` | `--version` | Drop the lease, leaving the task pending. **Useful for manual recovery** |
| `halt <ID>` | — | Halt a task (block further claims) |
| `continue <ID>` | — | Reverse `halt` |
| `search` | optional `--state`, `--limit`, `--cursor` | Inspect tasks across the server |

If you find yourself writing CLI scripts that compose these task primitives, you almost certainly want the SDK instead — see the per-SDK basic-ephemeral-world and basic-durable-world skills.

## `resonate mcp`

```shell
resonate mcp --server http://localhost:8001
```

Starts a stdio MCP server that wraps the Resonate HTTP API. Skill-aware agents (Claude Code, Cursor) launch this as a child process and call its tools (`promise-create`, `promise-get`, `promise-listen`, `promise-search`, `promise-settle`, plus `resonate-bash` when the bash transport is enabled). Direct CLI use of `resonate mcp` is rare — agents wire it up via their MCP config. See [`resonate-bash`](../resonate-bash/SKILL.md) for the full setup story.

## Configuration layering

For `resonate serve` and `resonate dev`, configuration loads in this order — each layer overrides the previous:

1. **Defaults** — what `--help` shows.
2. **`resonate.toml`** — if present, picked up from the working directory.
3. **`RESONATE_*` environment variables** — every CLI flag has an env-var equivalent. The leading `--` becomes the `RESONATE_` prefix; nested config segments are joined by a **double underscore** `__` (a multi-word leaf key keeps a single `_`). E.g. `--storage-postgres-url` ⇄ `RESONATE_STORAGE__POSTGRES__URL`, `--server-url` ⇄ `RESONATE_SERVER__URL`, `--auth-publickey` ⇄ `RESONATE_AUTH__PUBLICKEY`, `--tasks-retry-timeout` ⇄ `RESONATE_TASKS__RETRY_TIMEOUT`. Run `resonate serve --help` for exact names.
4. **CLI flags** — highest precedence.

Example `resonate.toml`:

```toml
[auth]
publickey = "/etc/resonate/auth/public.pem"

[storage]
type = "postgres"

[storage.postgres]
url = "${DATABASE_URL}"   # postgres://<user>:<password>@<host>:5432/resonate
```

This layering applies to the server, not the client subcommands. `--server` / `--token` on `promises`, `schedules`, etc. are read from flags or `RESONATE_SERVER` / `RESONATE_TOKEN` in the environment if set — they don't read `resonate.toml`.

## Common workflows

### Unblock a human-in-the-loop promise

A workflow is parked on `ctx.promise({ id: "approval-456" })`. To resolve it from outside any worker:

```shell
resonate promises resolve approval-456 --value '{"approved": true}'
```

Or reject with a reason:

```shell
resonate promises reject approval-456 --value '{"reason": "denied by reviewer"}'
```

### Reproduce a bug by invoking a workflow manually

```shell
resonate invoke repro-$(date +%s) \
  --func processOrder \
  --arg '{"orderId":"123","forceFailure":true}' \
  --target poll://any@workers
```

The new promise gets a unique ID, the worker claims it, and you can `resonate tree repro-…` afterward to see the call graph.

### Find stuck promises by project tag

Tag every promise on creation:

```shell
resonate promises create order-123 --timeout 24h --tags '{"project":"checkout"}'
```

Later, find every pending checkout promise:

```shell
resonate promises search --state pending --tags '{"project":"checkout"}' --limit 100
```

### Audit what a schedule has produced

Tag promises emitted by a schedule and search by tag:

```shell
resonate schedules create nightly-report \
  --cron "0 2 * * *" \
  --promise-id "report-{{.timestamp}}" \
  --promise-timeout 1h \
  --promise-tags '{"source":"nightly-report"}'

# A week later:
resonate promises search --tags '{"source":"nightly-report"}' --limit 200
```

### Smoke-test a deploy

```shell
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8001/health   # → 200
resonate invoke smoke-$(date +%s) --func ping --arg '{}' --timeout 30s
```

### Connect to a remote server

```shell
resonate promises get order-123 \
  --server https://resonate.example.com \
  --token "$RESONATE_TOKEN"
```

Or export once for a shell session:

```shell
export RESONATE_SERVER=https://resonate.example.com
export RESONATE_TOKEN=eyJhbGciOi...
resonate promises search --state pending
```

## Docs-vs-binary deltas

Older copies of the hosted docs show flags that the `resonate 0.9.7` binary does not accept. When the docs disagree with `resonate <cmd> --help`, the binary wins. Known deltas at the time of writing:

| Docs say | `resonate 0.9.7` binary actually accepts |
|---|---|
| `resonate promises resolve … --data '…'` | `--value '…'` (no `--data`) |
| `resonate promises search "order-*" --limit 10` | No positional pattern; filter via `--state` and `--tags` |
| `resonate schedules create … --description "…"` | No `--description`; also requires `--promise-timeout` (not shown in older examples) |
| `resonate <anything> --output json` | No `--output` flag; pipe to the HTTP API directly if you need JSON |
| `resonate <client> --username / --password` | No basic-auth flags; JWT via `-T` / `--token` only |

Always check `resonate <cmd> --help` before scripting. The help text is generated from the binary and matches what the binary will accept.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `connect: connection refused` | Server not running, or wrong URL | `resonate dev` to start; `curl http://localhost:8001/health` to verify |
| `404 Not Found` on `promises get` | Wrong ID (case-sensitive), or promise timed out and was reaped | `resonate promises search --state rejected_timedout --tags '{…}'` to confirm |
| `401 Unauthorized` | Server has JWT auth on; client has no `--token` | Pass `-T <JWT>` or `export RESONATE_TOKEN=…` |
| Server boots on SQLite despite `--storage-postgres-url` | Missing `--storage-type postgres` | Pass both flags; the storage type defaults to `sqlite` |
| `/healthz returns 404` | The route is `/health` (single segment) | Use `/health` in probes |
| Schedule never fires | `--cron` expression invalid, or no worker claiming the generated promise's target | Check `resonate schedules get <ID>`; verify a worker polls the right group |
| `invoke` sits pending forever | Default `--target poll://any@default` but workers poll a different group | Override `--target poll://any@<group>` to match the worker's group |
| `a value is required for '--transports-bash-exec-enabled <BOOL>'` | Bare boolean flag | These flags require explicit `true` / `false` |
| Older doc example fails with `unrecognized argument` | Docs-vs-binary drift | `resonate <cmd> --help` is authoritative |
