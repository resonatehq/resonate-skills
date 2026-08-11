---
name: resonate-server-scylladb
description: Run the Resonate protocol on ScyllaDB using resonate-on-scylladb — a separate Go server, not a storage backend of the core server. Covers local and existing-cluster setup, the SCYLLADB_/SERVER_/TIMEOUTS_/WORKER_ environment model (which is NOT the core server's RESONATE_ model), the six-table schema, the BUSL-1.1 licensing constraint, and the current gaps. Use when a user already runs ScyllaDB or another wide-column store and asks to run Resonate on it.
license: Apache-2.0
---

# Resonate on ScyllaDB

## Overview

[`resonatehq/resonate-on-scylladb`](https://github.com/resonatehq/resonate-on-scylladb) is a **Go** server that implements the Resonate protocol against ScyllaDB. It is an independent implementation of the published protocol — **not** a storage backend of the core Rust server ([`resonatehq/resonate`](https://github.com/resonatehq/resonate)).

Two consequences an agent must internalize before writing any config:

1. **You cannot get ScyllaDB by configuring the core server.** There is no `RESONATE_STORAGE__TYPE=scylladb`. You run a different binary from a different repository.
2. **It speaks the same HTTP/JSON protocol.** Application code, SDK choice, and worker code are unchanged. The only thing that changes is what `RESONATE_URL` points at.

Related: [`resonate-server-nats`](../resonate-server-nats/SKILL.md) (the other provider — NATS-native, not drop-in), [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) (core server on systemd), [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md) (core server on Cloud Run).

## When to use this skill

- The user already operates a ScyllaDB cluster and does not want to add a database for durable execution.
- The user asks whether Resonate can run on ScyllaDB or another wide-column store.
- The user is evaluating durable execution against storage they already run.

## When NOT to use this skill

- **The user does not already run ScyllaDB.** Recommend the core server on Postgres — it is the reference implementation, it is Apache 2.0, and it is what production deployments run today. Do not steer someone onto ScyllaDB for novelty.
- The user needs promise or task **search** — it is unimplemented here (see Gaps).
- The user is deploying **workers**, not the server. Worker deployment is unchanged; use the relevant per-SDK skill.

## Licensing — say this correctly

`resonate-on-scylladb` is **source-available under BUSL-1.1**, not Apache 2.0. Always describe it as "source-available (BUSL-1.1)" and never with a broader license label.

- All non-production use is free, including modifying and redistributing.
- The LICENSE sets `Additional Use Grant: None`, so **production use requires a commercial license** from Resonate HQ (`licensing@resonatehq.io`).
- Each released version converts to Apache 2.0 on the Change Date, `2030-07-01`.

If a user's plan implies production use, surface the license before they invest in the migration — not after.

## Configuration model — the biggest gotcha

The core server reads `RESONATE_`-prefixed variables with `__` nesting (figment). **This server does not.** It uses flat, component-prefixed variables. An agent that pattern-matches from the core server will write config that is silently ignored.

Priority order: CLI flags → environment variables → optional `resonate.yaml` → built-in defaults.

| Env var | Purpose |
|---|---|
| `SERVER_ADDR` | Listen address (default `:8001`) |
| `SERVER_DEBUG` | Debug mode — **drops and recreates the keyspace on connect** |
| `SERVER_LOG_LEVEL` | `debug`, `info`, `warn`, `error` (default `info`); any other value is a fatal startup error |
| `SCYLLADB_HOSTS` | Comma-separated seed hosts |
| `SCYLLADB_PORT` | CQL port |
| `SCYLLADB_USERNAME` | Username |
| `SCYLLADB_PASSWORD` | Password |
| `SCYLLADB_TLS_ENABLED` | Enable TLS |
| `SCYLLADB_TLS_INSECURE` | Skip certificate verification |
| `SCYLLADB_KEYSPACE` | Keyspace name |
| `SCYLLADB_REPLICATION` | Replication clause used when creating the keyspace |
| `TIMEOUTS_BUCKET_WIDTH` | Timeout bucket width (e.g. `1h`, `30m`) |
| `TIMEOUTS_BUCKET_LOOKBACK` | Past buckets to scan each tick |
| `TIMEOUTS_SHARDS` | Shard count for the timeout tables — must match across all server instances |
| `WORKER_TTL` | Worker row TTL (e.g. `15s`) |
| `WORKER_TICK_INTERVAL` | Coordinator tick interval (e.g. `1s`) |

The `resonate.yaml` equivalents nest under `server:`, `scylladb:`, `timeouts:`, and `worker:`.

```yaml title="resonate.yaml"
server:
  addr: ":8001"
  log-level: info
scylladb:
  hosts:
    - node-0.example.com
  keyspace: resonate
  tls:
    enabled: true
timeouts:
  bucket-width: 1h
  bucket-lookback: 1
  shards: 1
worker:
  ttl: 15s
  tick-interval: 1s
```

## Local setup

```bash
git clone https://github.com/resonatehq/resonate-on-scylladb
cd resonate-on-scylladb
docker compose --profile server up
```

**The `--profile server` is required.** The `server` service in `docker-compose.yaml` declares `profiles: [server]`, so a bare `docker compose up` starts ScyllaDB alone and nothing listens on `:8001`. Never hand a user the bare form.

With the profile, this brings up ScyllaDB plus the server on `:8001`. Point a worker at it exactly as you would the core server:

```bash
RESONATE_URL=http://localhost:8001
```

Requires Go 1.25.4+ (the `go` directive in `go.mod` is a minimum, not a floor to round down) and Docker to build from source.

## Schema provisioning — the destructive gotcha

**The server only applies DDL when debug mode is on, and debug mode drops the keyspace first.**

`CreateSchema` is wired directly to `Server.Debug`. With `--debug` (or `SERVER_DEBUG=true`) the server issues `DROP KEYSPACE IF EXISTS`, recreates the keyspace, and applies the tables — on every connect. Without it, the server applies no DDL at all and assumes the schema is already provisioned.

The bundled `docker-compose.yaml` runs `serve --debug`. That is why a local `docker compose up` works with no setup, and also why local state does not survive a restart.

Two rules follow, and an agent must never get these wrong:

1. **Never suggest `--debug` or `SERVER_DEBUG` against a cluster holding data.** It will destroy the keyspace. It is for local development and the test suites only.
2. **For any real deployment, the user provisions the schema themselves.** Do not tell them the server will handle it.

## Existing-cluster setup

Apply the schema out of band rather than letting the server create the keyspace. Do **not** hand the user a bare `cqlsh -f internal/dbms/schema.cql`: the file's first three lines are `CREATE KEYSPACE IF NOT EXISTS resonate;` (with no replication clause, so it takes ScyllaDB's defaults) and `USE resonate;`. That hardcodes the keyspace name and silently contradicts the advice to choose replication deliberately.

Have them create the keyspace themselves, then apply only the table DDL:

```bash
cqlsh -e "CREATE KEYSPACE resonate WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3};"

tail -n +4 internal/dbms/schema.cql > tables.cql
cqlsh -k resonate -f tables.cql
```

The keyspace name must match `SCYLLADB_KEYSPACE`. Then build the binary and point it at the cluster, with debug off:

```bash
go build -o resonate ./cmd/resonate

SCYLLADB_HOSTS=node-0.example.com,node-1.example.com,node-2.example.com \
SCYLLADB_KEYSPACE=resonate \
SCYLLADB_TLS_ENABLED=true \
./resonate serve
```

**The provider's CLI is also named `resonate` and also takes `serve`.** It is built from this repository, not from the core server repo. When writing instructions, make the build step explicit so the user cannot end up running the core server binary by accident.

`SCYLLADB_REPLICATION` is only honored on the keyspace-creation path, which is debug mode only. Since the user provisions the keyspace themselves, that setting does nothing on this path — do not include it in existing-cluster instructions.

## The data model

The schema has six tables. Knowing the shape matters for anyone operating this.

| Table | Primary key | Notes |
|---|---|---|
| `promises` | `(origin, id)` | One row per durable promise. Task state lives here too — the `task_*` columns — so acquire, fence, and complete are single-row operations. |
| `schedules` | `(origin, id)` | Recurring schedules. |
| `promise_timeouts` | `((bucket, shard), timeout_at, origin, promise_id)` | Time-ordered, bucketed, sharded. |
| `task_timeouts` | `((bucket, shard), timeout_at, timeout_type, origin, task_id)` | Same shape. |
| `schedule_timeouts` | `((bucket, shard), timeout_at, origin, schedule_id, create_token)` | Same shape. |
| `workers` | `(worker_id)` | Liveness, with a row TTL set by `WORKER_TTL`. |

Two properties worth explaining to a user weighing this:

- **State fans out.** One row per promise means state spreads across many small partitions rather than concentrating into a fixed few.
- **There is no event log.** State is the current row, not a sequence to replay, so nothing accumulates without bound and a long-running execution has no history-size ceiling. A long workflow is an ordinary loop; there is no history to truncate and no restart step to plan for.

`TIMEOUTS_SHARDS` sets how many partitions the expiry scan spreads across. Bucketing means each scan is a clustering-key range read inside one partition.

**`TIMEOUTS_SHARDS` must be identical on every instance and must not change after data lands.** The shard is a hash of the record id baked into the partition key, so a changed value orphans existing timeout rows in partitions no server scans, and durable timers stop firing without an error.

## Verifying a deployment

The repository ships three suites, all runnable under Docker Compose:

```bash
# Diff — every step checked against an in-memory oracle
docker compose -f docker-compose.test.yml -p resonate-diff --profile diff \
  up --build --abort-on-container-exit --exit-code-from tester-diff

# Kill — crash mid-transaction, assert invariants on what survives
docker compose -f docker-compose.test.yml -p resonate-kill --profile kill \
  up --build --abort-on-container-exit --exit-code-from tester-kill

# Linearizability — concurrent interleavings, model-checked
docker compose -f docker-compose.test.yml -p resonate-linz --profile linearizability \
  up --build --abort-on-container-exit --exit-code-from tester-linearizability
```

The kill tests abort an operation at every cooperative yield checkpoint — reads, cursor scans, non-transactional pre-inserts, lightweight-transaction commits, rollbacks, cleanups and batches — a thousand iterations by default, and check named state invariants on the remains. Nine invariants describe *accepted* orphans, because the recovery design writes the timeout entry before the transaction that commits the state change — an abort in that gap leaves a stale entry to clean up rather than losing state.

To smoke-check a running server, the routes are `GET /health`, `GET /poll/{group}/{id}` for long-polling clients, and a single `POST /` carrying the JSON RPC envelope:

```bash
curl -sf localhost:8001/health

curl -s localhost:8001 -X POST -H 'content-type: application/json' -d '{
  "kind": "promise.create",
  "head": { "corrId": "1", "version": "2026-04-01" },
  "data": { "id": "smoke", "timeoutAt": 9999999999999, "param": {}, "tags": {} }
}'
```

**`param` and `tags` are required and must be present**, even when empty — the validation layer rejects the payload with envelope status `400` and message `param is required` or `tags is required` respectively, before it reaches the database. The SDKs always send both, so this only bites hand-written requests like the one above.

Response status lives in the envelope head, not the HTTP status — the HTTP response is `200` even for a protocol-level `400`. `501` means the operation is recognized but unimplemented; a `400` whose data reads `unknown kind: <kind>` means the server does not recognize it at all. Match on the `unknown kind:` prefix, not on a bare `unknown kind`. See Gaps.

## Gaps — state these, don't discover them in production

- **No production reference deployments.** Nobody is running this in production yet. Say so.
- **Search is unimplemented.** `task.search` returns `501`. `promise.search` and `schedule.search` aren't recognized kinds at all and return `400` with `unknown kind: <kind>` — so don't probe for `501` to detect capability. Anything relying on querying promises by tag will not work.
- **No authentication.** An auth hook exists on the handler, but nothing ever sets it and its `Check` is an unimplemented stub that returns nil. Every request that reaches the server is served. Say so whenever the user describes a shared network.
- **Database-layer failure is uncharacterized.** A repair path exists in the code but is not wired into the running server, so a lost node or a network partition is not something the current tests cover.
- **Known bug:** deleting a schedule can leave stale rows in `schedule_timeouts`.
- **The schema is not frozen.** This is a young repository; check open pull requests before hard-coding assumptions about column layout.

## Checklist before recommending this to a user

1. Do they already run ScyllaDB? If not → core server on Postgres.
2. Have you told them it is BUSL-1.1 and production needs a license?
3. Do they need promise or task search? If yes → it will not work.
4. Have you stated that there are no production reference deployments?
5. Have you used `SCYLLADB_*` variables rather than the core server's `RESONATE_*` model?
6. Have you kept `--debug` / `SERVER_DEBUG` away from every command that touches real data, and told them they provision the schema themselves?
7. Have you told them there is no server-side auth, so network controls are the only protection?
8. If they want a NATS-native deployment instead, have you pointed at [`resonate-server-nats`](../resonate-server-nats/SKILL.md) and said it is not drop-in?
