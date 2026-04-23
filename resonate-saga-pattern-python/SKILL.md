---
name: resonate-saga-pattern-python
description: Implement saga patterns for distributed transactions in Python with Resonate — forward steps with compensating actions that unwind on failure. Use when coordinating multi-step Python processes that need consistency across failures without distributed locks or two-phase commits.
license: Apache-2.0
---

# Resonate Saga Pattern — Python

## Overview

A saga is a long-running transaction split into smaller steps, each with a compensating action. Forward steps run top-to-bottom; on failure, compensations run bottom-to-top to restore consistency. Python's native `try/except` expresses this cleanly when the durable function is a Resonate-registered generator.

For the TS version's mental model, see `resonate-saga-pattern-typescript`. The Python expression differs in syntax (no generic `Generator` type annotation, `try/except` instead of `try/catch`, plain `yield` not `yield*`) and in how compensation lists are collected.

## When to use

- Multi-step workflow where intermediate state is visible to other consumers (charge a card → create shipment)
- Each step is idempotent or cheap to retry individually
- Compensation logic exists for every committed step
- You need "all or nothing" consistency without a distributed transaction

**Don't use** when steps are a pure in-memory pipeline (use sequential `ctx.run` calls without compensation), or when you have access to a single DB transaction that covers all steps.

## Basic shape

```py
from resonate import Resonate

resonate = Resonate()

@resonate.register
def place_order(ctx, order_id: str):
    completed: list[str] = []

    try:
        yield ctx.run(reserve_inventory, order_id)
        completed.append("inventory")

        yield ctx.run(charge_payment, order_id)
        completed.append("payment")

        yield ctx.run(create_shipment, order_id)
        completed.append("shipment")

        return {"status": "success", "order_id": order_id}

    except Exception as err:
        # compensate in reverse order
        for step in reversed(completed):
            yield ctx.run(compensate, step, order_id)

        return {
            "status": "failed",
            "order_id": order_id,
            "reason": str(err),
            "compensated": completed,
        }


def compensate(ctx, step: str, order_id: str):
    if step == "shipment":
        cancel_shipment(order_id)
    elif step == "payment":
        refund_payment(order_id)
    elif step == "inventory":
        release_inventory(order_id)
```

Each forward step is a checkpoint — if the worker crashes mid-saga, Resonate resumes from the last successful `yield`. Compensations run only when the `except` block is entered, so a partially-completed saga's compensations are themselves durable.

## Explicit step objects

For sagas with many steps, an explicit list is easier to reason about than the flat `try/except`:

```py
from dataclasses import dataclass
from typing import Callable

@dataclass
class SagaStep:
    name: str
    forward: Callable
    backward: Callable

STEPS = [
    SagaStep("inventory", reserve_inventory, release_inventory),
    SagaStep("payment", charge_payment, refund_payment),
    SagaStep("shipment", create_shipment, cancel_shipment),
]

@resonate.register
def place_order(ctx, order_id: str):
    completed: list[SagaStep] = []

    try:
        for step in STEPS:
            yield ctx.run(step.forward, order_id)
            completed.append(step)
        return {"status": "success", "order_id": order_id}

    except Exception as err:
        for step in reversed(completed):
            yield ctx.run(step.backward, order_id)
        return {"status": "failed", "order_id": order_id, "reason": str(err)}
```

## Compensation must itself be retryable

A compensation that fails stalls the saga in an inconsistent state. Guard against this:

```py
from resonate.retry_policies import Exponential

@resonate.register
def place_order(ctx, order_id: str):
    completed: list[str] = []
    try:
        # ... forward steps ...
    except Exception:
        for step in reversed(completed):
            # aggressive retry: compensations MUST eventually succeed
            yield ctx.run(compensate, step, order_id).options(
                retry_policy=Exponential(),
                timeout=600.0,  # 10 minutes per compensation
            )
        raise
```

If a compensation exhausts its retry policy and still fails, you have a real inconsistency — log it, alert on-call, and resolve manually. The saga durable-function re-raises the original error so the caller sees the failure.

## Distinct Python idioms

- **No `yield*`** — every forward and compensation step is a plain `yield ctx.run(...)`. The generator-delegation concept from TS (`yield*`) doesn't apply in Python; `yield` is always single-value.
- **`try/except` over `try/catch`** — catch block is `except Exception as err:` (not `catch (error)`).
- **`reversed(completed)`** is a Python built-in that returns a reverse iterator without mutating the list. Prefer it over `completed[::-1]` (creates a copy) or `completed.reverse()` (mutates in place, which may surprise).
- **Dataclasses** (or pydantic) are the idiomatic way to model step objects, not interfaces or generics.
- **`Exception`** catches everything; if you want narrower handling, catch specific exception types defined in your domain (`PaymentDeclined`, `InventoryUnavailable`).

## Avoid: compensation that itself mutates in-flight state

Bad:

```py
def refund_payment(ctx, order_id: str):
    payment = PAYMENTS_DB.get(order_id)
    payment.status = "refunded"  # ← in-memory mutation; invisible to the system
    PAYMENTS_DB.save(payment)
```

The in-memory `status` flip is not durable; a subsequent replay sees an inconsistent state if the DB write fails. Make the compensation a pure operation against the system of record:

```py
def refund_payment(ctx, order_id: str):
    payments_api.refund(order_id)
    # idempotent by order_id; safe to retry
```

## Common variants

### Nested sub-saga

A single step can be its own saga via RPC:

```py
@resonate.register
def place_order(ctx, order_id: str):
    try:
        yield ctx.rpc("reserve_inventory_saga", order_id)  # ← sub-saga in another worker
        yield ctx.run(charge_payment, order_id)
        yield ctx.run(create_shipment, order_id)
    except Exception:
        # compensation logic; nested sagas compensate themselves from within their own except
        ...
```

The sub-saga's own compensation logic runs within the sub-saga; the parent doesn't know or care.

### Parallel fan-out inside a saga

```py
@resonate.register
def place_order(ctx, order_id: str):
    try:
        yield ctx.run(reserve_inventory, order_id)

        # kick off two independent steps in parallel
        p1 = yield ctx.begin_run(charge_payment, order_id)
        p2 = yield ctx.begin_run(send_confirmation_email, order_id)
        yield p1
        yield p2

        yield ctx.run(create_shipment, order_id)
    except Exception:
        ...
```

Both `charge_payment` and `send_confirmation_email` are durable; if either fails, the `except` block runs and compensations include both.

## Related skills

- `resonate-basic-durable-world-usage-python` — `ctx.run`, `ctx.begin_run`, options chain
- `resonate-recursive-fan-out-pattern-python` — parallel patterns inside or alongside sagas
- `resonate-external-system-of-record-pattern-python` — when the "saga" is actually just writing to one authoritative system with idempotency
- `durable-execution` — foundational concept; replay semantics make compensation safe
