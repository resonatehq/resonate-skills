---
name: resonate-basic-durable-world-usage-python
description: Core patterns for writing Resonate durable functions in Python — async/await syntax, Context API reference (run, rpc, detached, promise, sleep, dependency injection), and the determinism rules that make recovery work. Use when writing any function body registered with r.register().
license: Apache-2.0
---

# Resonate Basic Durable World Usage — Python

## Overview

Durable functions are the payload of Resonate. They are `async def` coroutines that the SDK treats as a sequence of durable checkpoints. Every `await ctx.run(...)` or `await ctx.rpc(...)` is a save point: if the process crashes, the function resumes from the last successful checkpoint on another process or a restart.

This skill covers the Context API (`ctx.*`) that you use inside those functions. The ephemeral-world counterpart (`Resonate` class, registration, top-level invocation) lives in `resonate-basic-ephemeral-world-usage-python`.

## The contract

Every durable function:

1. Is registered with `r.register(fn)` in the ephemeral world.
2. Takes `ctx` (the Context) as its first parameter; any other parameters after that are the invocation args.
3. Uses `await` to interact with `ctx`. Each awaited `ctx.*` call is a durable checkpoint.
4. Is deterministic when replayed: no direct `time.time()`, no `random.random()`, no raw network I/O outside `ctx.run`.

Minimal shape:

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from resonate.context import Context

async def process_order(ctx: Context, order_id: str) -> dict:
    # step 1 — durable: its result is persisted
    order = await ctx.run(load_order, order_id)

    # step 2 — durable: runs locally in the same worker
    charge = await ctx.run(charge_card, order)

    # step 3 — ordinary Python control flow; free to branch, loop, raise
    if not charge["ok"]:
        raise RuntimeError(f"payment failed: {charge['reason']}")

    return {"order_id": order_id, "status": "paid"}
```

If this function crashes after step 1 but before step 2, it resumes at step 2 on restart — `load_order` is NOT re-run. Its result is read from the promise store.

## Same-process invocation: `ctx.run`

`ctx.run` invokes another function in the same process durably. The durable function awaits it and the result is checkpointed:

```python
async def foo(ctx: Context, arg: str) -> str:
    result = await ctx.run(bar, arg)
    return result

async def bar(ctx: Context, arg: str) -> str:
    # bar can be any async def; it runs as a durable leaf
    return arg.upper()
```

To fire off multiple children and await them in parallel, launch them before awaiting:

```python
async def enrich_batch(ctx: Context, order_ids: list[str]) -> list[dict]:
    # Launch all children (returns futures)
    futures = [ctx.run(enrich_one, oid) for oid in order_ids]
    # Await them all
    results = [await f for f in futures]
    return results
```

Note: unawaited `ctx.run(...)` futures are still tracked by the runtime via structured concurrency — the parent cannot settle until all spawned children complete. But explicitly awaiting them gives you the results.

## Remote invocation: `ctx.rpc` and `ctx.detached`

`ctx.rpc` dispatches a registered function to a remote worker process by name:

```python
async def orchestrator(ctx: Context, batch_id: str) -> dict:
    # "worker_fn" must be registered in a reachable worker group
    result = await ctx.rpc("worker_fn", batch_id)
    return result
```

`ctx.detached` is fire-and-forget — the parent does NOT wait for the result:

```python
async def place_order(
    ctx: Context, customer: str, sku: str, qty: int, amount: int
) -> tuple[str, str, str]:
    stock_ref = await ctx.run(reserve_stock, sku, qty)
    charge_ref = await ctx.run(charge_card, customer, amount)

    # Dispatch audit as a detached workflow; get back the promise id only
    audit_future = ctx.detached(
        "write_audit_log",
        customer=customer,
        sku=sku,
        amount=amount,
        stock_ref=stock_ref,
        charge_ref=charge_ref,
    )
    audit_id = await audit_future.id()   # await the id; do NOT await the result
    return stock_ref, charge_ref, audit_id
```

Use `detached` when the side effect should survive independently of the parent's success — post-commit hooks, notifications, audit logs.

## Options

Options configure an individual call. For Context invocations, options come first via `ctx.options(...)`:

```python
from resonate.retry import Exponential, Never

async def foo(ctx: Context, arg: str) -> str:
    result = await ctx.options(
        retry_policy=Exponential(),
        version=2,
    ).run(bar, arg)
    return result
```

To target a specific worker group for `rpc`:

```python
result = await ctx.options(target="db-workers").rpc("query_orders", user_id)
```

## External promises: `ctx.promise`

`ctx.promise` creates a durable promise the function suspends on until externally resolved. This is the primitive for human-in-the-loop workflows.

```python
async def fulfill_order(ctx: Context, order_id: str, amount: int) -> str:
    # Create the promise; its id is deterministic (derived from the workflow id)
    approval = ctx.promise()
    approval_id = await approval.id()

    # Notify whoever needs to resolve (via a leaf — side effects in leaves only)
    await ctx.run(notify_reviewer, order_id, amount, approval_id)

    # Suspend here until an external party resolves or rejects
    decision_raw = await approval

    if decision_raw.get("approve"):
        return await ctx.run(ship_order, order_id)
    return await ctx.run(cancel_order, order_id)
```

The external resolver calls `await r.promises.resolve(approval_id, Value(data=decision))` from outside the worker (a webhook handler, a CLI, a separate process).

## `ctx.sleep`

`ctx.sleep` suspends for a `timedelta`. The Resonate server holds the continuation; no process needs to stay alive during the sleep:

```python
from datetime import timedelta

async def daily_digest(ctx: Context, user_id: str) -> None:
    while True:
        await ctx.sleep(timedelta(hours=24))
        await ctx.rpc("send_digest", user_id)
```

Pass a `timedelta`, not a raw float. For fake async work inside a leaf, use `await asyncio.sleep(...)` directly instead.

## Dependencies

`ctx.get_dependency(MyType)` retrieves a dependency registered via `r.with_dependency(obj)`:

```python
import psycopg

async def write_to_db(ctx: Context, record: dict) -> None:
    db = ctx.get_dependency(psycopg.Connection)
    # Wrap the actual I/O in a helper via ctx.run so the side effect is checkpointed
    await ctx.run(_insert_row, db, record)

async def _insert_row(ctx: Context, db: psycopg.Connection, record: dict) -> None:
    db.execute("INSERT INTO orders VALUES (...)", record)
```

Never instantiate DB clients or HTTP sessions inside a durable function body — they would be recreated on every replay.

## Side effects and the `ctx.run` envelope

Any side effect — HTTP call, DB write, file write, print — belongs inside a function that the durable function reaches via `ctx.run`. This ensures the effect is executed exactly once: the orchestrator awaits the checkpoint, the leaf runs, the result is persisted, and on replay the stored result is returned instead of re-executing.

Bad (replays re-execute the side effect):

```python
async def foo(ctx: Context, arg: str) -> str:
    requests.post(WEBHOOK_URL, json=arg)  # runs on every replay
    return "done"
```

Good (side effect is checkpointed exactly once):

```python
async def send_webhook(ctx: Context, arg: str) -> str:
    requests.post(WEBHOOK_URL, json=arg)
    return "sent"

async def foo(ctx: Context, arg: str) -> str:
    await ctx.run(send_webhook, arg)
    return "done"
```

## Determinism: time and randomness

Never call `time.time()` or `random.random()` directly in a durable function — on replay, the values change and the function branches differently. Wrap them in a leaf via `ctx.run`:

```python
import time, random

async def _get_time(ctx: Context) -> float:
    return time.time()

async def _get_random(ctx: Context) -> float:
    return random.random()

async def foo(ctx: Context) -> None:
    t = await ctx.run(_get_time)
    r = await ctx.run(_get_random)
```

The leaf settles once and its value is persisted; replay returns the stored value.

## Common shapes

**Sequential pipeline with retries:**

```python
from resonate.retry import Exponential

async def ingest(ctx: Context, source_url: str) -> dict:
    data = await ctx.options(retry_policy=Exponential()).run(fetch, source_url)
    parsed = await ctx.run(parse, data)
    await ctx.run(persist, parsed)
    return {"count": len(parsed)}
```

**Parallel fan-out within a single function:**

```python
async def enrich_batch(ctx: Context, ids: list[str]) -> list[dict]:
    futures = [ctx.run(enrich_one, i) for i in ids]
    return [await f for f in futures]
```

**Recursive workflow (dispatching itself via rpc):**

```python
async def crawl(ctx: Context, url: str, depth: int) -> dict:
    page = await ctx.run(fetch, url)
    if depth > 0:
        for link in page["links"]:
            ctx.detached("crawl", link, depth - 1)
    return {"url": url, "id": page["id"]}
```

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — the Client API companion
- `resonate-basic-debugging-python` — diagnosing stuck workflows, missing registrations, non-determinism regressions
- `durable-execution` — the foundational concept
- `resonate-philosophy` — "do not build these things" mindset
