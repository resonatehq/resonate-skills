---
name: resonate-basic-ephemeral-world-usage-python
description: Core patterns for using the Resonate Python SDK's Client APIs from the ephemeral world — initializing, registering durable functions, invoking them top-level (run / rpc / schedule-adjacent), setting dependencies, and managing external promises. Use when writing any process-level Python code that needs to launch or coordinate Resonate workflows.
license: Apache-2.0
---

# Resonate Basic Ephemeral World Usage — Python

## Overview

The ephemeral world is anywhere your Python process starts: a `main()`, a FastAPI route, an HTTP handler, a script, a worker entry point. You use the `Resonate` class to register durable functions and invoke them. Once an invocation starts, it crosses into the durable world and Resonate guarantees its completion — retries on failure, resumes after crashes, continues across restarts.

This skill covers the Client API surface. The Durable World (Context APIs) lives in `resonate-basic-durable-world-usage-python`.

## Preconditions

- Python ≥ 3.12 (the SDK uses recent typing features).
- `resonate-sdk` on PyPI.
- For remote mode: a running Resonate server. Local mode needs nothing.
- **Server compatibility caveat:** Python SDK v0.6.7 currently targets the legacy Resonate server protocol; v0.9.x-targeting Python support is coming in a future release. If you're running a server at v0.9.x, check SDK release notes before expecting remote mode to work.

## Install

```shell
uv add resonate-sdk
# or: pip install resonate-sdk
# or: poetry add resonate-sdk
```

## Initialize

Local (zero-dependency, everything in-process):

```py
from resonate import Resonate

resonate = Resonate()
# or identically:
resonate = Resonate.local()
```

Remote (connects to a Resonate server):

```py
from resonate import Resonate

resonate = Resonate.remote()
# or with explicit host/port:
resonate = Resonate.remote(
    host="localhost",
    store_port=8001,
    message_source_port=8001,
)
```

One `Resonate` instance per process. Use local mode for tests, scripts, and incremental adoption. Flip to remote when you need state persistence across restarts, multi-process coordination, or horizontal worker scaling.

## Register a durable function

Durable functions in Python are **generators**. Any `def` that uses `yield` internally is a generator; the SDK treats the yielded values as durable checkpoints. There is no `function*` keyword (TS) or `async def` requirement — just a regular `def` with `yield` inside.

Prefer the decorator form:

```py
@resonate.register
def process_order(ctx, order_id: str):
    # ctx is the Context; everything inside the function body runs in the durable world
    # ...
    return {"order_id": order_id, "status": "done"}
```

Or register imperatively after definition:

```py
def process_order(ctx, order_id: str):
    # ...
    return result

resonate.register(process_order)
```

Registration name defaults to the function name. To override (useful for RPC when the caller and callee are in different codebases):

```py
@resonate.register
def process_order_v2(ctx, order_id: str):
    # ...
    return result
# caller references "process_order_v2" by that exact string over RPC
```

## Share resources via dependencies

The ephemeral world is where you construct things that durable code shouldn't build itself — database connections, HTTP clients, config objects. Register them once:

```py
import psycopg

conn = psycopg.connect(DATABASE_URL)
resonate.set_dependency("db", conn)
```

Inside a durable function, access them via `ctx.get_dependency("db")`. Dependencies are *references*, not serialized state — keep them construction-heavy and usage-simple.

## Invoke a durable function

There are four shapes of invocation, corresponding to same-process vs remote and sync vs async:

```py
# same process, synchronous — blocks until the function returns
result = resonate.run("invocation-id", process_order, "order-123")

# same process, asynchronous — returns a handle to await later
handle = resonate.begin_run("invocation-id", process_order, "order-123")
# ... do other work ...
result = handle.result()

# remote process, synchronous — routes via Resonate server, waits for result
result = resonate.rpc("invocation-id", "process_order", "order-123")

# remote process, asynchronous — fire and await
handle = resonate.begin_rpc("invocation-id", "process_order", "order-123")
result = handle.result()
```

Invocation IDs are **yours to choose** and **deterministic**. Pick something stable that uniquely names this invocation across retries — `order:{order_id}` is better than `random.uuid()`, because the latter creates a new invocation every restart.

## Subscribe to an existing invocation

```py
handle = resonate.get("invocation-id")
result = handle.result()
```

Raises if the invocation ID doesn't exist. Useful for a separate process (e.g., a status-check HTTP endpoint) to block on a workflow started elsewhere.

## Options

Options configure the invocation. For ephemeral calls, options **precede** the call:

```py
from resonate.retry_policies import Exponential, Constant, Linear, Never

result = resonate.options(
    target="poll://any@workers",     # RPC routing group
    timeout=60.0,                    # seconds (not ms — Python uses seconds)
    retry_policy=Exponential(),      # or Constant(), Linear(), Never()
    tags={"tenant": "acme"},         # searchable metadata
    idempotency_key="custom-ikey",
    version=1,                       # function version for schema evolution
).run("invocation-id", process_order, "order-123")
```

Retry policies are importable from `resonate.retry_policies` — `Exponential`, `Constant`, `Linear`, `Never` are all public.

## External promises

External promises let a process outside the durable function resolve or reject it. This is the primitive behind human-in-the-loop workflows, webhooks, and any "wait for external event" pattern.

```py
import json, time

# create a promise that another process will resolve
resonate.promises.create(
    id="approval-ord-123",
    timeout=int(time.time() * 1000) + 30 * 60 * 1000,  # 30-minute deadline (ms since epoch)
)

# elsewhere (webhook, UI, CLI) resolves it
resonate.promises.resolve(id="approval-ord-123", data=json.dumps({"approved": True}))

# or rejects it
resonate.promises.reject(id="approval-ord-123", data=json.dumps({"reason": "denied"}))

# query state
p = resonate.promises.get("approval-ord-123")
```

Inside a durable function, `yield ctx.promise(id="approval-ord-123")` blocks until the promise settles — see `resonate-basic-durable-world-usage-python` for details.

## What belongs in the ephemeral world vs the durable world

**Ephemeral world (Client APIs, `Resonate` class):**
- Initialization, registration, top-level invocation
- Dependency construction (`set_dependency`)
- External promise management (`promises.create/resolve/reject/get`)
- Subscribing to running invocations (`get`)

**Durable world (Context APIs, used inside a registered function):**
- Sub-invocations (`ctx.run`, `ctx.rpc`, `ctx.beginRun`, `ctx.beginRpc`, `ctx.detached`)
- Sleeps, promises, deterministic time/random
- Accessing dependencies (`ctx.get_dependency`)

You cannot mix them. A Client API call inside a durable function raises; a Context API call outside a durable function has no context to bind to.

## Worker lifecycle

For remote mode, a process that calls `resonate.register(...)` becomes a worker for that function. Start the process and it begins long-polling for RPC invocations targeted at its group:

```py
from resonate import Resonate

resonate = Resonate.remote(host="localhost")

@resonate.register
def process_order(ctx, order_id: str):
    # ...
    return result

# block forever; the SDK's long-poller keeps the process alive
import time
while True:
    time.sleep(60)
```

Or let your framework's main loop keep the process alive (FastAPI, Flask, etc.).

## Common shapes

**Top-level script:**
```py
from resonate import Resonate

resonate = Resonate()

@resonate.register
def greet(ctx, name: str):
    return f"Hello, {name}!"

if __name__ == "__main__":
    print(resonate.run("greet:alice", greet, "Alice"))
```

**HTTP handler that starts a workflow:**
```py
@app.post("/orders")
def create_order(body: OrderBody):
    order_id = body.order_id
    # return immediately; workflow continues in the background
    resonate.begin_run(f"order:{order_id}", process_order, order_id)
    return {"status": "started", "order_id": order_id}
```

**Pure-ephemeral coordinator that calls workers over RPC:**
```py
resonate = Resonate.remote(host="resonate.example.com")
result = resonate.rpc(
    "batch:2026-04-16",
    "nightly_reconciliation",
    {"date": "2026-04-16"},
)
```

## Related skills

- `resonate-basic-durable-world-usage-python` — the Context API companion; use after a function is registered
- `resonate-basic-debugging-python` — what to do when workflows hang, replay unexpectedly, or raise
- `resonate-philosophy` and `durable-execution` — foundational concepts; read these first if you're new to Resonate
