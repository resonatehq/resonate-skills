---
name: resonate-server-nats
description: Run the Resonate protocol on NATS JetStream using resonate-on-nats — a separate Go server where NATS is both storage and transport, with no HTTP interface at all. Covers dev vs serve, partitioning across instances, wiring workers with the NatsNetwork client in the TypeScript and Python SDKs, the subject layout, the lowercase group/pid trap, the BUSL-1.1 constraint, and the current gaps. Use when a user already runs NATS and asks to run Resonate on it.
license: Apache-2.0
---

# Resonate on NATS

## Overview

[`resonatehq/resonate-on-nats`](https://github.com/resonatehq/resonate-on-nats) is a **Go** server that implements the Resonate protocol on NATS JetStream. JetStream KV holds the state and NATS carries the messages. The server needs no separate database, and it exposes no HTTP interface.

Three things an agent must internalize before writing any code:

1. **It is not a backend of the core server.** There is no flag on [`resonatehq/resonate`](https://github.com/resonatehq/resonate) that produces this. It is a different binary from a different repository.
2. **It is not drop-in.** Unlike the ScyllaDB provider, this server has **no HTTP interface whatsoever** — there is no `net/http` in it. `RESONATE_URL` is meaningless here. Workers connect over NATS via the `NatsNetwork` client.
3. **Only TypeScript and Python can talk to it.** `NatsNetwork` ships in those two SDKs. Go, Rust, and Java have no NATS client, so teams on those languages cannot use this provider. Say so early rather than after they've committed.

Related: [`resonate-server-scylladb`](../resonate-server-scylladb/SKILL.md) (the drop-in provider), [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) (core server on systemd).

## When to use this skill

- The user already runs NATS and does not want to add a database beside it.
- The user asks whether Resonate can run on NATS or JetStream.
- The user is wiring a worker to an existing `resonate-on-nats` deployment.

## When NOT to use this skill

- **The user does not already run NATS.** Recommend the core server on Postgres — reference implementation, Apache 2.0, what production runs today.
- **The user is on Go, Rust, or Java.** No NATS client exists in those SDKs. Do not suggest writing one as a workaround.
- **The user wants the smallest possible change.** The ScyllaDB provider speaks the same HTTP/JSON protocol as the core server, so worker wiring is untouched. This one changes it.
- The user needs promise or task search, or server-side authentication — neither exists here (see Gaps).

## Licensing — say this correctly

`resonate-on-nats` is **source-available under BUSL-1.1**, not Apache 2.0. Always describe it as "source-available (BUSL-1.1)" and never with a broader license label.

- All non-production use is free, including modifying and redistributing.
- The LICENSE sets `Additional Use Grant: None`, so **production use requires a commercial license** from Resonate HQ (`licensing@resonatehq.io`).
- Each released version converts to Apache 2.0 on the Change Date, `2030-07-01`.

## Requirements

- **nats-server 2.14.x with JetStream enabled.** Durable timers use JetStream message schedules (the stream is created with `AllowMsgSchedules`), and cron-style recurrence landed in 2.14. The repository builds against 2.14.2, so treat 2.14 as the supported floor — check this before anything else.
- Go 1.25+ to build the server from source.

## Running the server

```bash
git clone https://github.com/resonatehq/resonate-on-nats
cd resonate-on-nats
go build -o resonate-on-nats .
```

**Development** — embedded NATS server, fresh store in a temp directory that is discarded on exit (it is file-backed, not in-memory), nothing external:

```bash
./resonate-on-nats dev
```

**Production** — against a NATS deployment the user already runs:

```bash
./resonate-on-nats serve --nats-url nats://localhost:4222
```

### Flags

| Flag | `serve` default | `dev` default | Purpose |
|---|---|---|---|
| `--nats-url` | `nats://127.0.0.1:4222` | — | NATS server to connect to |
| `--partitions` | `16` | `1` | Total partition count for the stream |
| `--subscribe` | all | all | Partition indices this instance handles |
| `--debug` | `false` | `true` | Deterministic testing endpoints |
| `--port` | — | `4222` | Port for the embedded NATS server |
| `--log-level` | `info` | `info` | Persistent root flag, applies to every subcommand |

`--debug` enables `debug.start`, `debug.stop`, `debug.reset`, `debug.tick`, `debug.snap`, which let tests drive logical time. When it is off those five kinds return `501` rather than being absent. **It defaults on in `dev` and off in `serve`.**

**Never enable `--debug` on `serve`, and do not describe it as a harmless observability toggle.** The flag itself destroys nothing at startup, but two of the kinds it unlocks are destructive: `debug.reset` purges the entire stream and every non-debug key in the KV bucket, and `debug.start` sets a flag that halts outbox delivery server-wide. The server has no authentication, so anyone able to publish to the request subject can invoke both.

The binary also ships client subcommands — `promises`, `tasks`, `schedules`, `invoke` — that take `--nats-url` and `--origin` (default `default`). Useful for poking at a running server without writing code.

## Partitioning across instances

State lives in JetStream KV, partitioned across instances. Each instance subscribes to a subset of partitions and processes messages **serially per origin**, using optimistic concurrency (CAS) so multiple instances don't conflict.

```bash
./resonate-on-nats serve --partitions 16 --subscribe 0,1,2,3
./resonate-on-nats serve --partitions 16 --subscribe 4,5,6,7
```

**Every instance must pass the same `--partitions`.** The stream is created with `CreateOrUpdateStream`, so a mismatched instance rewrites the subject transform for everyone. Pick the number with headroom, because changing it later means resharding. Note the defaults differ between `dev` (1) and `serve` (16), so a value that worked locally is not what you get in production.

## Wiring workers

`NatsNetwork` lives inside the main SDKs, not in a separate package. There is no `resonate-transport-nats` package to install.

### TypeScript

`@resonatehq/sdk` with the `@nats-io/transport-node` peer dependency (`^3.0.0`). Import from the `/nats` subpath — it is not exported from the package root.

```ts
import { Resonate } from "@resonatehq/sdk";
import { NatsNetwork } from "@resonatehq/sdk/nats";
import { connect } from "@nats-io/transport-node";

const conn = await connect({ servers: "nats://localhost:4222" });
const resonate = new Resonate({ network: new NatsNetwork({ conn }) });
```

`NatsNetworkConfig`: `{ conn, pid?, group?, serverTopic?, workerTopic?, requestTimeout?, logger? }`.

### Python

```bash
uv add "resonate-sdk[nats]"   # pulls in nats-py
```

```python
import asyncio

import nats
from resonate.resonate import Resonate
from resonate.network.nats import NatsNetwork


async def main():
    conn = await nats.connect("nats://localhost:4222")
    resonate = Resonate(network=NatsNetwork(conn))
    ...


asyncio.run(main())
```

Two things an agent gets wrong by pattern-matching: `Resonate` is **not** re-exported from the package root (`from resonate import Resonate` raises `ImportError` — the shipped `__init__.py` exports only `PROTOCOL_VERSION` and `now_ms`), and it must be constructed **inside a running event loop**, because `__init__` calls `asyncio.create_task`. Never write this snippet with a bare top-level `await`.

Signature: `NatsNetwork(conn, pid=None, group=None, *, server_topic=..., worker_topic=..., request_timeout=30.0)`.

The connection lifecycle stays **outside** the SDK. `stop()` tears down this network's subscriptions only and leaves the connection for the caller to drain and close.

### Two traps that fail silently

1. **Python: `url` beats `network`.** Network selection resolves `url` > `network` > `RESONATE_URL` env var. Pass both a `url` and a `network` and the URL wins — the `NatsNetwork` is ignored with no error. Pass one.
2. **Keep custom `group` and `pid` lowercase.** NATS subjects are case-sensitive end to end and nothing normalizes case, so a publisher and a subscriber that disagree on capitalization never meet. Default pids are lowercase uuid hex, so staying lowercase keeps hand-set values consistent with the defaults. (Do **not** repeat the claim in the SDK source comments that Go's URL parser lowercases the host — it does not; `net/url` lowercases only the scheme.)

## Subject layout

Relevant when the NATS deployment has subject-level permissions.

- Requests → `resonate.requests.{base64url(origin)}`, where the encoding is unpadded base64url (`base64.RawURLEncoding`), with a `Resonate-Reply-To` header naming a private inbox. The reply lands on that inbox; the server ignores the NATS reply subject.
- Workers receive on a unicast `resonate.recv.{group}.{pid}` and an anycast `resonate.recv.{group}`, queue-subscribed on the group so exactly one member handles each anycast message.
- Task addresses use the `nats://` scheme; the server maps `nats://{subject}` back to a subject.

Prefixes are configurable per client (`serverTopic` / `server_topic`, `workerTopic` / `worker_topic`); the request timeout defaults to 30 seconds.

## Gaps — state these, don't discover them in production

- **No production reference deployments.** Nobody is running this in production yet.
- **No search at all.** Neither `promise.search` nor `task.search` is a recognized kind; both fall through to the default branch and return `400 bad request`. Anything querying promises by tag will not work.
- **Lighter test coverage than the ScyllaDB provider.** Unit tests cover the store, schedules, and tasks, but there are no oracle-diff, crash, or linearizability suites. If a user is choosing between the two providers on evidence of correctness, say so.
- **No authentication.** The server has no auth layer. Everything protecting it must come from the NATS deployment — accounts, users, subject permissions. Raise this whenever the user describes a shared or multi-tenant broker.
- **No HTTP interface.** No REST surface, no way to point an HTTP client at it, no health endpoint.
- **TypeScript and Python only.** No Go, Rust, or Java client.
- **The layout is not settled.** This is a young repository; the subject scheme and partition model can still change. Check open pull requests before building tooling against them.

## Checklist before recommending this to a user

1. Do they already run NATS? If not → core server on Postgres.
2. Is their broker nats-server 2.14+ with JetStream? If not → it will not run.
3. Are they on TypeScript or Python? If not → no client exists.
4. Have you told them it is BUSL-1.1 and production needs a license?
5. Have you told them there is no server-side auth, and that NATS permissions are the only control?
6. Do they need promise or task search? If yes → it will not work.
7. Have you stated that there are no production reference deployments, and that test coverage here is lighter than the ScyllaDB provider's?
8. If they want the smallest change, have you mentioned that [`resonate-server-scylladb`](../resonate-server-scylladb/SKILL.md) is drop-in and this is not?

## Examples

[`resonatehq/nats-demos`](https://github.com/resonatehq/nats-demos) — runnable TypeScript and Python demos against this server.
