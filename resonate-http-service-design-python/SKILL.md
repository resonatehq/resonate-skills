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
            → Resonate client (begin_run / rpc / promises.resolve)
                → worker group A (business logic)
                    → ctx.rpc → worker group B (specialized, e.g. DB)
```

- **HTTP server:** FastAPI/Flask/Django app handling routes; maps HTTP requests to Resonate invocations
- **Worker service:** a process (same or different host) with `@resonate.register`-ed functions; long-polls for RPC invocations
- **Resonate server:** durable promise store + coordination hub (in-memory local mode, or remote server for multi-process)

## Rules

- **Route handlers are ephemeral.** Use Client APIs (`resonate.run`, `begin_run`, `rpc`, `begin_rpc`, `promises.*`). Never use Context APIs in a route handler.
- **Durable functions run in workers.** Regular `def` with `yield` + `@resonate.register` decorator.
- **All external effects (DB, HTTP, filesystem) run inside `ctx.run` envelopes** — never at the top of a durable function, never at module import time.
- **Stable invocation IDs.** Derive from request inputs (`order:{order_id}`) — not `uuid.uuid4()` at route-handler time unless you persist the ID in the response so the client can poll.

## Route design patterns

### 1. Submit-and-poll (async HTTP)

Client submits a job, gets a job ID, polls for status.

```py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from resonate import Resonate

app = FastAPI()
resonate = Resonate.remote(host="localhost")


class JobSubmission(BaseModel):
    order_id: str
    items: list[str]


@app.post("/jobs")
def submit_job(body: JobSubmission):
    job_id = f"order:{body.order_id}"

    # start the workflow; return immediately
    resonate.begin_rpc(
        job_id,
        "process_order",
        body.order_id,
        body.items,
    ).options(target="poll://any@order-workers")

    return {"job_id": job_id, "status": "started"}


@app.get("/jobs/{job_id}")
def get_job(job_id: str):
    try:
        handle = resonate.get(job_id)
        # non-blocking status check: a handle.result() would block;
        # instead, query the promise store for state
        promise = resonate.promises.get(job_id)
        if promise.state == "pending":
            return {"job_id": job_id, "status": "running"}
        elif promise.state == "resolved":
            return {"job_id": job_id, "status": "done", "result": promise.value}
        else:
            return {"job_id": job_id, "status": "failed", "reason": promise.value}
    except Exception:
        raise HTTPException(status_code=404, detail="job not found")
```

### 2. Submit-and-callback (webhook resolves promise)

Client submits; workflow blocks on a `ctx.promise`; external system resolves via webhook.

```py
# worker
@resonate.register
def approve_then_ship(ctx, order_id: str):
    p = yield ctx.promise(id=f"approval:{order_id}")
    decision = yield p
    if not decision.get("approved"):
        return {"status": "rejected"}
    yield ctx.run(ship_order, order_id)
    return {"status": "shipped"}


# route — start
@app.post("/orders/{order_id}/approve-and-ship")
def start(order_id: str):
    resonate.begin_rpc(f"order:{order_id}", "approve_then_ship", order_id).options(
        target="poll://any@order-workers"
    )
    return {"status": "pending_approval"}


# route — webhook resolves promise
@app.post("/webhooks/approval/{order_id}")
def approval_webhook(order_id: str, payload: dict):
    import json
    resonate.promises.resolve(
        id=f"approval:{order_id}",
        data=json.dumps(payload),
    )
    return {"ok": True}
```

### 3. Synchronous HTTP wrapping a fast workflow

If the workflow completes quickly (seconds, not minutes), block on it and return the result directly:

```py
@app.post("/price-quote")
def price_quote(body: QuoteRequest):
    quote_id = f"quote:{body.sku}:{body.qty}"
    result = resonate.rpc(
        quote_id,
        "compute_quote",
        body.sku,
        body.qty,
    ).options(target="poll://any@pricing-workers", timeout=10.0)
    return {"quote_id": quote_id, "price": result["price"]}
```

`rpc()` blocks the route handler. Keep timeouts tight — a slow worker holds the HTTP request open.

## Worker-side lifecycle

A worker process is just a Python program that:
1. Instantiates `Resonate.remote(host=...)`
2. Registers its durable functions
3. Long-polls forever

```py
# workers/order_worker.py
from resonate import Resonate

resonate = Resonate.remote(host="resonate.internal")

@resonate.register
def process_order(ctx, order_id: str, items: list[str]):
    yield ctx.run(validate_order, order_id, items)
    yield ctx.run(charge_payment, order_id)
    yield ctx.run(create_shipment, order_id)
    return {"order_id": order_id, "status": "done"}


if __name__ == "__main__":
    import time
    while True:
        time.sleep(60)  # block forever; SDK's long-poller keeps the process alive
```

Or let your framework's main loop keep the process alive (uvicorn, gunicorn, hypercorn, etc.) if the same process serves HTTP too.

## Separating worker groups

For larger services, split specialization across worker groups:

```py
# db worker group — "poll://any@db"
@resonate.register
def query_orders(ctx, user_id: str):
    db = ctx.get_dependency("db")
    return db.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,)).fetchall()

# business worker group — "poll://any@business"
@resonate.register
def place_order(ctx, user_id: str, cart: list[str]):
    # reach into the DB group via RPC
    history = yield ctx.rpc("query_orders", user_id).options(target="poll://any@db")
    # ... business logic ...
    return {"user_id": user_id, "order_id": "..."}
```

Each group runs in its own process (or cluster of processes). RPC routes through Resonate's promise store with the server handling fair dispatch.

## Combining FastAPI + in-process worker

Small services often put the HTTP server and the worker in the same process:

```py
from fastapi import FastAPI
from resonate import Resonate
import asyncio

app = FastAPI()
resonate = Resonate.remote(host="localhost")


@resonate.register
def process_order(ctx, order_id: str):
    yield ctx.run(charge_payment, order_id)
    return {"order_id": order_id, "status": "done"}


@app.post("/orders/{order_id}")
def start(order_id: str):
    resonate.begin_run(f"order:{order_id}", process_order, order_id)
    return {"status": "started"}
```

Because `resonate.register` registers in the same process, both RPC-routing and in-process `begin_run` both work. For higher throughput, move workers to dedicated processes.

## Distinct Python idioms

- **FastAPI** is the leading modern web framework; patterns use FastAPI syntax. Flask + Django patterns translate directly — substitute `@app.route` for `@app.post`.
- **Pydantic models** (`BaseModel`) for request bodies — idiomatic + validates at parse time.
- **`int(time.time() * 1000)`** for ms-since-epoch timeouts on promises (vs TS's `Date.now()`).
- **Resonate Python SDK is synchronous.** Unlike TS where you'd `await resonate.run(...)`, Python's `resonate.run(...)` blocks directly. No `async`/`await` on the Resonate side (though your FastAPI route handler can still be `async def` for other async I/O).
- **`json.dumps(...)`** on promise data — explicit; TS is often implicit about encoding.

## Common gotchas

- **Don't use Context APIs in a route handler.** `ctx` only exists inside a `@resonate.register`-ed function body. A route handler gets a `Resonate` client, not a `Context`.
- **Don't rely on FastAPI request lifecycle for workflow timing.** A durable function can outlive an HTTP request by hours or days. Return a job ID to the client and let them poll (pattern 1) or subscribe via webhook (pattern 2).
- **Don't share Resonate client instances across processes without care.** Each process (HTTP server, worker) instantiates its own `Resonate.remote(...)`; they coordinate via the Resonate server, not shared memory.
- **Don't forget `target`.** When the route handler calls `rpc`/`begin_rpc`, specify `target="poll://any@worker-group"` so the invocation routes to a worker process and not back to the HTTP process.

## What's NOT in this skill (and why)

- **Deployment-specific guidance** (GCP Cloud Run + Python, Supabase + Python, AWS Lambda + Python) — deferred or skipped per the skills-expansion initiative's non-goal on new deployment targets. See `resonate-server-deployment` for server-side install, or the TS deployment skills for patterns that may translate by analogy.
- **Token-based authentication on the Python client** — the Python SDK does not yet support token auth (`docs/deploy/security.mdx` is explicit about this). When the SDK adds it, a dedicated `resonate-token-authentication-python` skill lands.

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — `Resonate.remote`, `begin_run`, `rpc`, `options(target=)`, promise management
- `resonate-basic-durable-world-usage-python` — `@resonate.register`, `ctx.run`, `ctx.rpc`, `ctx.promise`
- `resonate-human-in-the-loop-pattern-python` — webhook-driven promise resolution (pattern 2 in this skill, expanded)
- `resonate-saga-pattern-python` — multi-step workflows suitable for pattern 3 (sync HTTP)
