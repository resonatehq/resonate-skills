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
- A running Resonate server (`resonate dev` starts one on `localhost:8001`). The Rust server v0.9.x runs on a single port — no separate store port.

## Install

```shell
uv add 'resonate-sdk>=0.7.0'
# or: pip install 'resonate-sdk>=0.7.0'
# or: poetry add 'resonate-sdk>=0.7.0'
```

## Initialize

```python
from __future__ import annotations
import os
from resonate.resonate import Resonate

url = os.environ.get("RESONATE_URL", "http://localhost:8001")

# Connect to a Resonate server
r = Resonate(url=url)

# With a named group (required for rpc routing to a specific worker pool)
r = Resonate(url=url, group="order-workers")

# With a default retry policy
from resonate.retry import Never
r = Resonate(url=url, retry_policy=Never())
```

One `Resonate` instance per process. The server is always required for durable execution across restarts and multi-process coordination.

## Register a durable function

Durable functions in the Python SDK v0.7.0 are `async def` coroutines. Register them explicitly after definition:

```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from resonate.context import Context

async def process_order(ctx: Context, order_id: str) -> dict:
    # ctx is the Context; everything inside the function body runs in the durable world
    return {"order_id": order_id, "status": "done"}

r.register(process_order)
```

Registration name defaults to the function name. To override (useful for RPC when caller and callee are in different codebases):

```python
r.register(process_order_v2, name="process_order", version=2)
```

With a per-function retry policy:

```python
from resonate.retry import Exponential

r.register(charge, retry_policy=Exponential())
```

## Share resources via dependencies

The ephemeral world is where you construct things that durable code shouldn't build itself — database connections, HTTP clients, config objects. Register them by type:

```python
import psycopg

conn = psycopg.connect(DATABASE_URL)
r.with_dependency(conn)
```

Inside a durable function, access them by type via `ctx.get_dependency(MyType)`. Dependencies are references, not serialized state — keep them construction-heavy and usage-simple.

## Invoke a durable function

```python
import asyncio
import time

async def main() -> None:
    # r.run is SYNC — returns a handle immediately; await the handle's .result()
    handle = r.run(f"order-{time.time_ns()}", process_order, "order-123")
    result = await handle.result()

    # One-liner form
    result = await r.run(f"order-{time.time_ns()}", process_order, "order-123").result()

    # Dispatch by name (rpc) — routes through the Resonate server to a worker group
    result = await r.rpc(f"order-{time.time_ns()}", "process_order", "order-123").result()
```

Invocation IDs are **yours to choose** and **deterministic**. Pick something stable that uniquely names this invocation across retries — `f"order:{order_id}"` is better than a random UUID, because the latter creates a new invocation every restart.

## Subscribe to an existing invocation

```python
async def main() -> None:
    handle = await r.get("my-invocation-id")
    result = await handle.result()
```

Returns a handle for the existing promise. Useful for a separate process (e.g., a status-check HTTP endpoint) to block on a workflow started elsewhere.

## Options

Options configure the invocation. For top-level calls, options come first on the Resonate instance:

```python
from resonate.retry import Exponential, Constant, Linear, Never

handle = r.options(
    target="backend",               # RPC routing group
    retry_policy=Exponential(),     # or Constant(), Linear(), Never()
    version=2,                      # function version for schema evolution
).rpc(f"order-{time.time_ns()}", "process_order", "order-123")

result = await handle.result()
```

## External promises

External promises let a process outside the durable function resolve or reject it. This is the primitive behind human-in-the-loop workflows, webhooks, and any "wait for external event" pattern.

```python
from datetime import timedelta
from resonate.types import Value

# resolve a promise (e.g. from a webhook handler)
await r.promises.resolve("approval-ord-123", Value(data={"approved": True}))

# or reject it
await r.promises.reject("approval-ord-123", Value(data={"reason": "denied"}))

# query state
record = await r.promises.get("approval-ord-123")
```

All `r.promises.*` methods are `async` — always `await` them.

## Schedules (cron)

```python
from datetime import timedelta

sched = await r.schedule(
    id="nightly-digest",
    cron="0 9 * * *",
    func_name="send_digest",
    args=("user-123",),
    promise_timeout=timedelta(minutes=30),
)
```

## Shutdown

Always stop the Resonate instance on exit:

```python
try:
    result = await r.run(id, fn, arg).result()
finally:
    await r.stop()
```

## What belongs in the ephemeral world vs the durable world

**Ephemeral world (Client APIs, `Resonate` class):**
- Initialization, registration, top-level invocation
- Dependency construction (`r.with_dependency(obj)`)
- External promise management (`r.promises.resolve/reject/get`)
- Subscribing to running invocations (`r.get`)

**Durable world (Context APIs, used inside a registered function):**
- Sub-invocations (`await ctx.run(...)`, `await ctx.rpc(...)`)
- Sleeps, promises, detached fire-and-forget
- Accessing dependencies (`ctx.get_dependency(MyType)`)

## Common shapes

**Top-level script:**

```python
from __future__ import annotations
import asyncio
import os
import time
from typing import TYPE_CHECKING
from resonate.resonate import Resonate

if TYPE_CHECKING:
    from resonate.context import Context

async def greet(ctx: Context, name: str) -> str:
    return f"Hello, {name}!"

async def main() -> None:
    r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))
    r.register(greet)
    try:
        result = await r.run(f"greet-{time.time_ns()}", greet, "Alice").result()
        print(result)
    finally:
        await r.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

**HTTP handler that starts a workflow:**

```python
@app.post("/orders")
async def create_order(body: OrderBody):
    order_id = body.order_id
    # Return immediately; workflow continues durably in the background
    r.run(f"order:{order_id}", process_order, order_id)
    return {"status": "started", "order_id": order_id}
```

**Pure-ephemeral coordinator that dispatches to workers over RPC:**

```python
handle = r.options(target="order-workers").rpc(
    f"batch-{time.time_ns()}",
    "nightly_reconciliation",
    {"date": "2026-04-16"},
)
result = await handle.result()
```

## Related skills

- `resonate-basic-durable-world-usage-python` — the Context API companion; use after a function is registered
- `resonate-basic-debugging-python` — what to do when workflows hang, replay unexpectedly, or raise
- `resonate-philosophy` and `durable-execution` — foundational concepts; read these first if you're new to Resonate
