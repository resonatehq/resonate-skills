---
name: resonate-human-in-the-loop-pattern-python
description: Implement human-in-the-loop workflows in Python — durable functions that suspend on ctx.promise() until an external actor (webhook, UI, CLI, operator) resolves or rejects. Use when a workflow must wait on an action that doesn't originate from another worker, such as approval gates, manual review, or out-of-band data.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern — Python

## Overview

A human-in-the-loop workflow suspends on a durable promise that is resolved (or rejected) by something outside the Resonate worker set — a person clicking an Approve button, a webhook from a third-party system, an operator running a CLI command. The worker doesn't poll or sleep; it opens a `ctx.promise()` and awaits it, and Resonate wakes it when the promise settles.

## When to use

- Approval gates in business workflows (expense approval, content moderation, deploy gate)
- Waiting on third-party callbacks (Stripe webhooks, DocuSign signature events)
- Operator-driven unblock steps (break-glass in incident runbooks)
- Any workflow step where the data or decision comes from outside the Resonate worker

## Basic shape

```python
from __future__ import annotations
import asyncio, os, time
from typing import TYPE_CHECKING
from resonate.resonate import Resonate
from resonate.types import Value

if TYPE_CHECKING:
    from resonate.context import Context

r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))

async def notify_reviewer(ctx: Context, order_id: str, amount: int, approval_id: str) -> str:
    # Side effects live in leaves — this prints/notifies exactly once
    print(f"  order {order_id} (${amount}) needs review; promise id: {approval_id!r}")
    return approval_id

async def ship_order(ctx: Context, order_id: str) -> str:
    return f"shipped-{order_id}"

async def cancel_order(ctx: Context, order_id: str) -> str:
    return f"canceled-{order_id}"

async def fulfill_order(ctx: Context, order_id: str, amount: int) -> str:
    # Open the approval promise; its id is deterministic (derived from the workflow id)
    approval = ctx.promise()
    approval_id = await approval.id()

    # Notify the reviewer via a leaf (side effects belong in leaves)
    await ctx.run(notify_reviewer, order_id=order_id, amount=amount, approval_id=approval_id)

    # Suspend here — the worker holds no state while waiting
    # This can be seconds, hours, or days; no process needs to stay alive
    decision_raw = await approval

    if decision_raw.get("approve"):
        return await ctx.run(ship_order, order_id=order_id)
    return await ctx.run(cancel_order, order_id=order_id)
```

From outside the worker (a webhook, a UI route, a CLI):

```python
from resonate.types import Value

# Resolve the promise — wakes the suspended workflow
await r.promises.resolve(approval_id, Value(data={"approve": True, "note": "looks good"}))

# Or reject it
await r.promises.reject(approval_id, Value(data={"reason": "policy violation"}))
```

`r.promises.resolve/reject` are `async` — always `await` them.

## The canonical SDK example

The SDK's `examples/human-in-the-loop` shows the full pattern with a `ReviewerInbox` dependency (an `asyncio.Future`) so the simulated reviewer learns the promise id the instant the leaf creates it — without polling:

```python
import asyncio
from resonate.resonate import Resonate
from resonate.types import Value

class ReviewerInbox:
    def __init__(self) -> None:
        self.approval_id: asyncio.Future[str] = asyncio.get_running_loop().create_future()

    def publish(self, approval_id: str) -> None:
        if not self.approval_id.done():
            self.approval_id.set_result(approval_id)

async def notify_reviewer(ctx, order_id: str, amount: int, approval_id: str) -> str:
    print(f"  order {order_id} (${amount}) pending; id: {approval_id!r}")
    ctx.get_dependency(ReviewerInbox).publish(approval_id)
    return approval_id

async def fulfill_order(ctx, order_id: str, amount: int) -> str:
    approval = ctx.promise()
    approval_id = await approval.id()
    await ctx.run(notify_reviewer, order_id=order_id, amount=amount, approval_id=approval_id)
    decision_raw = await approval
    if decision_raw.get("approve"):
        return await ctx.run(ship_order, order_id=order_id, note=decision_raw.get("note", ""))
    return await ctx.run(cancel_order, order_id=order_id, note=decision_raw.get("note", ""))

async def main() -> None:
    r = Resonate(url="http://localhost:8001")
    inbox = ReviewerInbox()
    r.with_dependency(inbox)
    r.register(fulfill_order)
    try:
        wid = f"fulfill-{time.time_ns()}"
        handle = r.run(wid, fulfill_order, "order-42", 199)

        # Simulated reviewer: wait for the inbox, then resolve
        async def simulate_reviewer() -> None:
            approval_id = await inbox.approval_id
            await r.promises.resolve(approval_id, Value(data={"approve": True, "note": "ok"}))

        reviewer = asyncio.create_task(simulate_reviewer())
        out = await handle.result()
        await reviewer
        print(f"OK: {out}")
    finally:
        await r.stop()

asyncio.run(main())
```

## Promise IDs are deterministic

The orchestrator and the external resolver need to agree on the promise ID. When you call `ctx.promise()` without an explicit id, the SDK assigns a deterministic id derived from the workflow's own promise id (e.g., `{workflow_id}.1` for the first promise). Your leaf captures and publishes this id so the external resolver knows where to settle.

If you need a portable, human-readable id, pass a keyword argument (not yet in the v0.7.0 public API for `ctx.promise`; derive it from workflow inputs and embed it in the leaf instead). The safe pattern is always: open the promise, await its id, pass the id to a leaf for notification.

## Passing context to the resolver

Pass data to the resolver via a leaf that publishes the promise id alongside any routing metadata:

```python
async def notify_for_travel(
    ctx: Context, traveler_id: str, destination: str, approval_id: str
) -> str:
    # In a real system: Slack message, email, dashboard row
    print(f"  travel approval needed: {traveler_id} → {destination} [{approval_id}]")
    return approval_id
```

The reviewer sees the id and calls `r.promises.resolve(approval_id, Value(data=decision))`.

## Webhook-driven resolution

A typical FastAPI webhook route that resolves a promise:

```python
from fastapi import FastAPI, Request
from resonate.resonate import Resonate
from resonate.types import Value

app = FastAPI()
r = Resonate(url="http://localhost:8001")

@app.post("/webhooks/docusign")
async def docusign_callback(req: Request) -> dict:
    body = await req.json()
    envelope_id = body["envelopeId"]
    status = body["status"]

    if status == "completed":
        await r.promises.resolve(
            f"docusign:{envelope_id}",
            Value(data={"signed": True, "signed_at": body["completedAt"]}),
        )
    elif status in {"declined", "voided"}:
        await r.promises.reject(
            f"docusign:{envelope_id}",
            Value(data={"reason": status}),
        )

    return {"ok": True}
```

Note: the worker that is awaiting `ctx.promise()` is woken as soon as the webhook resolves.

## Multiple human approvers (parallel)

Fan out to independent approvers; wait for all to respond:

```python
async def multi_approver(ctx: Context, request_id: str, count: int) -> dict:
    # Open N approval promises in sequence (each gets a deterministic id)
    approvals = [ctx.promise() for _ in range(count)]
    ids = [await a.id() for a in approvals]

    # Notify each approver via a leaf
    for i, approval_id in enumerate(ids):
        await ctx.run(notify_approver, request_id, i, approval_id)

    # Await all decisions
    decisions = [await a for a in approvals]
    approved_count = sum(1 for d in decisions if d.get("approve"))

    if approved_count == count:
        return {"status": "approved"}
    return {"status": "rejected", "approved": approved_count, "of": count}
```

## Distinct Python idioms

- **`Value(data=...)`** — `r.promises.resolve/reject` wrap the payload in a `Value` object. Import from `resonate.types`.
- **`r.promises.*` are all `async`** — always `await` them.
- **FastAPI `async def` routes** work naturally with `await r.promises.resolve(...)`.
- **`ctx.get_dependency(ReviewerInbox)`** — dependencies are fetched by type, not string key.
- **`try/except` on `await approval`** — handles timeout/rejection naturally with Python's exception model.

## Avoid: polling

Bad:

```python
async def bad_approval(ctx: Context, order_id: str) -> dict:
    # Polling defeats the point
    while True:
        status = await ctx.run(check_approval_status, order_id)
        if status:
            break
        await ctx.sleep(timedelta(seconds=30))
    return {"status": "approved"}
```

`ctx.promise()` gives you wake-on-event; polling wastes checkpoints and loses the ability to pass decision data into the workflow.

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — `r.promises.resolve/reject/get` from the ephemeral world (webhook handler side)
- `resonate-basic-durable-world-usage-python` — `ctx.promise`, `ctx.get_dependency`
- `resonate-saga-pattern-python` — sagas with human-gated steps
- `durable-execution` — foundational replay semantics; promise-wait survives crashes
