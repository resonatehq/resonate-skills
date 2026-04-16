---
name: resonate-human-in-the-loop-pattern-python
description: Implement human-in-the-loop workflows in Python — durable functions that block on ctx.promise until an external actor (webhook, UI, CLI, operator) resolves or rejects. Use when a workflow must wait on an action that doesn't originate from another worker, such as approval gates, manual review, or out-of-band data.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern — Python

## Overview

A human-in-the-loop workflow blocks on a promise that is resolved (or rejected) by something outside the Resonate worker set — a person clicking an Approve button, a webhook from a third-party system, an operator running a CLI command. The worker doesn't poll or sleep; it yields on `ctx.promise(id=...)` and Resonate wakes it when the promise settles.

## When to use

- Approval gates in business workflows (expense approval, content moderation, deploy gate)
- Waiting on third-party callbacks (Stripe webhooks, DocuSign signature events)
- Operator-driven unblock steps (break-glass in incident runbooks)
- Any workflow step where the data or decision comes from outside the Resonate worker

## Basic shape

```py
from resonate import Resonate

resonate = Resonate()

@resonate.register
def expense_approval(ctx, expense_id: str, amount: float):
    # create or resume a promise keyed by the expense
    decision_promise = yield ctx.promise(id=f"approval:{expense_id}")

    # block here — worker goes to sleep; Resonate wakes us when someone resolves
    decision = yield decision_promise

    if decision.get("approved"):
        yield ctx.run(process_reimbursement, expense_id, amount)
        return {"expense_id": expense_id, "status": "approved"}
    else:
        yield ctx.run(notify_rejection, expense_id, decision.get("reason"))
        return {"expense_id": expense_id, "status": "rejected"}
```

From outside the worker (a webhook, a UI route, a CLI):

```py
# resolve the promise from the ephemeral world
import json

resonate.promises.resolve(
    id=f"approval:{expense_id}",
    data=json.dumps({"approved": True, "approver": "alice@acme.io"}),
)

# or reject
resonate.promises.reject(
    id=f"approval:{expense_id}",
    data=json.dumps({"reason": "over-budget"}),
)
```

## Promise IDs must be deterministic

The workflow and the external resolver need to agree on the promise ID. Derive it from stable workflow inputs:

```py
@resonate.register
def moderate_post(ctx, post_id: str, author_id: str):
    # ID is derivable from workflow inputs — resolver in the moderation UI
    # constructs the same ID and calls promises.resolve
    decision = yield ctx.promise(id=f"moderation:{post_id}")
    result = yield decision
    return result
```

Bad: `ctx.promise(id=f"approval:{uuid.uuid4()}")` — the ID is random per invocation; the external resolver has no way to construct it.

## Passing context to the resolver

Pass data to the resolver via the promise's `data` parameter when creating it. The resolver sees it when it calls `resonate.promises.get(id)`:

```py
@resonate.register
def approve_travel(ctx, traveler_id: str, destination: str):
    promise = yield ctx.promise(
        id=f"travel-approval:{traveler_id}:{destination}",
        data={"needs_approval_from": "manager", "destination": destination},
    )
    decision = yield promise
    return decision
```

Useful for routing in the approval UI: "which group does this approval belong to?" lives on the promise rather than in a separate DB.

## Timeouts

Promises have a deadline; if nobody resolves or rejects by the deadline, the yield raises. Use timeouts to bound how long a workflow waits:

```py
import time

@resonate.register
def approve_with_sla(ctx, order_id: str):
    promise = yield ctx.promise(
        id=f"approval:{order_id}",
        # 24-hour SLA on manual approvals
        timeout=int(time.time() * 1000) + 24 * 60 * 60 * 1000,
    )

    try:
        decision = yield promise
        return {"status": "decided", "decision": decision}
    except Exception as err:
        # promise timed out; auto-escalate
        yield ctx.run(escalate_to_senior, order_id)
        return {"status": "auto-escalated", "reason": str(err)}
```

**Note:** promise timeouts are absolute ms-since-epoch (from the Python SDK API), not relative seconds. The inconsistency between `ctx.sleep(float_seconds)` and `ctx.promise(timeout=int_ms)` is in the SDK; the skill reflects it accurately.

## Webhook-driven resolution

A typical FastAPI webhook route that resolves a promise:

```py
from fastapi import FastAPI, Request
import json

app = FastAPI()

@app.post("/webhooks/docusign")
async def docusign_callback(req: Request):
    body = await req.json()
    envelope_id = body["envelopeId"]
    status = body["status"]

    if status == "completed":
        resonate.promises.resolve(
            id=f"docusign:{envelope_id}",
            data=json.dumps({"signed": True, "signed_at": body["completedAt"]}),
        )
    elif status in {"declined", "voided"}:
        resonate.promises.reject(
            id=f"docusign:{envelope_id}",
            data=json.dumps({"reason": status}),
        )

    return {"ok": True}
```

The worker that registered `envelope_id` is unblocked as soon as DocuSign's webhook hits.

## Multiple human approvers (parallel)

Fan out to independent approvers; wait for the first to respond (or all, depending on policy):

```py
@resonate.register
def multi_approver(ctx, request_id: str, approvers: list[str]):
    promises = [
        (yield ctx.promise(id=f"approve:{request_id}:{a}"))
        for a in approvers
    ]

    # all-approvers policy: wait for everyone
    decisions = [(yield p) for p in promises]
    approved_count = sum(1 for d in decisions if d.get("approved"))

    if approved_count == len(approvers):
        return {"status": "approved"}
    else:
        return {"status": "rejected", "approved_count": approved_count}
```

For "first approver wins" use `begin_run` on a small helper that races them — Python doesn't have a built-in `Promise.race` equivalent; see `resonate-recursive-fan-out-pattern-python` for racing primitives.

## Distinct Python idioms

- **`json.dumps(...)` on resolve data** — the Python SDK's `data` parameter expects a string; JSON is the conventional format. The TS skill is ambiguous about encoding; Python's stdlib `json` makes the serialization explicit.
- **FastAPI / Flask route for webhooks** — the typical entry point; the skill uses FastAPI's async syntax and notes that `resonate.promises.resolve` from a sync context works fine (no `await` needed on the Python SDK).
- **`try/except` on `yield promise`** — Python's exception model makes timeout/reject handling natural.
- **`int(time.time() * 1000)` for ms-since-epoch** — Python uses seconds by default; promise timeouts are ms. Skills call this out explicitly.

## Avoid: polling

Bad:

```py
@resonate.register
def bad_approval(ctx, order_id: str):
    # ← polling defeats the point
    while True:
        status = yield ctx.run(check_approval_status, order_id)
        if status:
            break
        yield ctx.sleep(30.0)
    return {"status": "approved"}
```

`ctx.promise` gives you wake-on-event; polling wastes checkpoints and loses the ability to pass decision data into the workflow.

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — `promises.resolve/reject/get` from the ephemeral world (webhook handler side)
- `resonate-basic-durable-world-usage-python` — `ctx.promise` details (id, data, timeout)
- `resonate-saga-pattern-python` — sagas with human-gated steps
- `durable-execution` — foundational replay semantics; promise-wait survives crashes
