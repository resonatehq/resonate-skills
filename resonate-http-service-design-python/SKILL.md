---
name: resonate-http-service-design-python
description: Design HTTP services in Python that use Resonate durable functions behind FastAPI / Flask / Django route handlers — routing patterns, workflow boundaries, RPC to specialized worker groups, webhook-driven promise resolution. Use when building or refactoring Python HTTP APIs that trigger durable workflows.
license: Apache-2.0
---

# Resonate HTTP Service Design — Python

## Overview

Route handlers in a Python web framework (FastAPI, Flask, Django) are the entrypoint for durable workflows; they start or await invocations via the `Resonate` client. Actual business logic runs in worker processes that register durable functions. Downstream services (a DB worker, a search worker) expose their own durable functions and are invoked via RPC.

This skill covers the shape of that split: what runs where, how routes interact with workers, and how webhooks resolve durable promises.

## Architecture model

```
 client → HTTP routes (FastAPI/Flask)
            → Resonate client (r.run / r.rpc / r.promises.resolve)
                → worker group A (business logic)
                    → ctx.rpc → worker group B (specialized, e.g. DB)
```

- **HTTP server:** FastAPI/Flask/Django app handling routes; maps HTTP requests to Resonate invocations
- **Worker service:** a process (same or different host) with `r.register(...)`-ed functions; connects to the Resonate server
- **Resonate server:** durable promise store + coordination hub (Rust v0.9.x, single port 8001)

## Rules

- **Route handlers are ephemeral.** Use Client APIs (`r.run`, `r.rpc`, `r.promises.*`). Never use Context APIs in a route handler.
- **Durable functions run in workers.** `async def` with `await ctx.run/rpc` + `r.register(fn)`.
- **All external effects (DB, HTTP, filesystem) run inside `ctx.run` envelopes** — never at the top of a durable function, never at module import time.
- **Stable invocation IDs.** Derive from request inputs (`f"order:{order_id}"`) — not `uuid.uuid4()` at route-handler time unless you persist the ID in the response so the client can poll.

## Route design patterns

### 1. Submit-and-poll (async HTTP)

Client submits a job, gets an ID, polls for status:

```python
from __future__ import annotations
import os, time
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from resonate.resonate import Resonate

app = FastAPI()
r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))


class JobSubmission(BaseModel):
    order_id: str
    items: list[str]


@app.post("/jobs")
async def submit_job(body: JobSubmission) -> dict:
    job_id = f"order:{body.order_id}"
    # Fire and forget — workflow runs durably in the background
    r.options(target="order-workers").rpc(job_id, "process_order", body.order_id, body.items)
    return {"job_id": job_id, "status": "started"}


@app.get("/jobs/{job_id}")
async def get_job(job_id: str) -> dict:
    try:
        record = await r.promises.get(job_id)
        state = record.state if hasattr(record, "state") else "pending"
        if state == "resolved":
            return {"job_id": job_id, "status": "done", "result": record.value}
        elif state == "rejected":
            return {"job_id": job_id, "status": "failed", "reason": record.value}
        else:
            return {"job_id": job_id, "status": "running"}
    except Exception:
        raise HTTPException(status_code=404, detail="job not found")
```

### 2. Submit-and-callback (webhook resolves promise)

Client submits; workflow blocks on `ctx.promise()`; external system resolves via webhook:

```python
from resonate.types import Value

# Worker
async def approve_then_ship(ctx, order_id: str) -> dict:
    approval = ctx.promise()
    approval_id = await approval.id()
    await ctx.run(notify_approver, order_id, approval_id)
    decision = await approval
    if not decision.get("approved"):
        return {"status": "rejected"}
    await ctx.run(ship_order, order_id)
    return {"status": "shipped"}


# Route — start the workflow
@app.post("/orders/{order_id}/approve-and-ship")
async def start(order_id: str) -> dict:
    r.options(target="order-workers").rpc(
        f"order:{order_id}", "approve_then_ship", order_id
    )
    return {"status": "pending_approval"}


# Route — webhook resolves the durable promise
@app.post("/webhooks/approval/{order_id}")
async def approval_webhook(order_id: str, payload: dict) -> dict:
    await r.promises.resolve(
        f"approval:{order_id}",
        Value(data={"approved": payload.get("approved", False)}),
    )
    return {"ok": True}
```

### 3. Synchronous HTTP wrapping a fast workflow

If the workflow completes quickly, block on it and return the result directly:

```python
@app.post("/price-quote")
async def price_quote(body: QuoteRequest) -> dict:
    quote_id = f"quote:{body.sku}:{body.qty}"
    result = await r.options(target="pricing-workers").rpc(
        quote_id,
        "compute_quote",
        body.sku,
        body.qty,
    ).result()
    return {"quote_id": quote_id, "price": result["price"]}
```

`r.rpc(...)` returns a handle; calling `.result()` and awaiting it blocks the route handler. Keep timeouts tight — a slow worker holds the HTTP request open.

## Worker-side lifecycle

A worker process:
1. Instantiates `Resonate(url=..., group=...)`
2. Registers its durable functions
3. Stays alive (via its own event loop or a `while True` sleep)

```python
# workers/order_worker.py
from __future__ import annotations
import asyncio, os
from typing import TYPE_CHECKING
from resonate.resonate import Resonate

if TYPE_CHECKING:
    from resonate.context import Context

r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"), group="order-workers")

async def process_order(ctx: Context, order_id: str, items: list[str]) -> dict:
    await ctx.run(validate_order, order_id, items)
    await ctx.run(charge_payment, order_id)
    await ctx.run(create_shipment, order_id)
    return {"order_id": order_id, "status": "done"}

r.register(process_order)

if __name__ == "__main__":
    async def main() -> None:
        try:
            # Block until Ctrl-C; SDK long-polls for work
            await asyncio.Event().wait()
        finally:
            await r.stop()

    asyncio.run(main())
```

## Separating worker groups

For larger services, split specialization across worker groups:

```python
# db worker group
r_db = Resonate(url=url, group="db")

async def query_orders(ctx: Context, user_id: str) -> list:
    db = ctx.get_dependency(psycopg.Connection)
    return db.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,)).fetchall()

r_db.register(query_orders)

# business worker group
r_biz = Resonate(url=url, group="business")

async def place_order(ctx: Context, user_id: str, cart: list[str]) -> dict:
    # Reach into the DB group via RPC
    history = await ctx.options(target="db").rpc("query_orders", user_id)
    # ... business logic ...
    return {"user_id": user_id, "order_id": "..."}

r_biz.register(place_order)
```

Each group runs in its own process (or cluster of processes). RPC routes through Resonate's promise store with the server handling fair dispatch.

## Combining FastAPI + in-process worker

Small services often put the HTTP server and the worker in the same process:

```python
from __future__ import annotations
import asyncio, os
from fastapi import FastAPI
from resonate.resonate import Resonate

app = FastAPI()
r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))

async def process_order(ctx, order_id: str) -> dict:
    await ctx.run(charge_payment, order_id)
    return {"order_id": order_id, "status": "done"}

r.register(process_order)

@app.post("/orders/{order_id}")
async def start(order_id: str) -> dict:
    r.run(f"order:{order_id}", process_order, order_id)
    return {"status": "started"}

@app.on_event("shutdown")
async def shutdown() -> None:
    await r.stop()
```

Because `r.register` registers in the same process, both RPC-routing and in-process `r.run(...)` work. For higher throughput, move workers to dedicated processes.

## Distinct Python idioms

- **FastAPI** is the leading modern web framework; patterns use FastAPI `async def` route handlers.
- **Pydantic models** (`BaseModel`) for request bodies — idiomatic + validates at parse time.
- **`r.run(...)` is SYNC** — it returns a handle immediately; the actual async work is awaiting `handle.result()`. You can fire-and-forget by not awaiting the handle.
- **`r.promises.*` are all `async`** — always `await` them in async route handlers.
- **`Value(data=...)`** wraps the payload for `r.promises.resolve/reject`.

## Common gotchas

- **Don't use Context APIs in a route handler.** `ctx` only exists inside a `r.register()`-ed function body. A route handler gets a `Resonate` client, not a `Context`.
- **Don't rely on FastAPI request lifecycle for workflow timing.** A durable function can outlive an HTTP request by hours or days. Return a job ID and let the client poll.
- **Always `await r.stop()` on shutdown.** Register a shutdown handler (`@app.on_event("shutdown")`) to drain in-flight work gracefully.
- **Don't forget `target`.** When the route handler calls `r.rpc(...)`, specify `.options(target="group-name")` so the invocation routes to a worker group.

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — `Resonate`, `r.run`, `r.rpc`, `r.options(target=)`, promise management
- `resonate-basic-durable-world-usage-python` — `r.register`, `ctx.run`, `ctx.rpc`, `ctx.promise`
- `resonate-human-in-the-loop-pattern-python` — webhook-driven promise resolution (pattern 2 in this skill, expanded)
- `resonate-saga-pattern-python` — multi-step workflows suitable for pattern 3 (sync HTTP)
