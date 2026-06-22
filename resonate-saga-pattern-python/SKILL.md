---
name: resonate-saga-pattern-python
description: Implement saga patterns for distributed transactions in Python with Resonate — forward steps with compensating actions that unwind on failure. Use when coordinating multi-step Python processes that need consistency across failures without distributed locks or two-phase commits.
license: Apache-2.0
---

# Resonate Saga Pattern — Python

## Overview

A saga is a long-running transaction split into smaller steps, each with a compensating action. Forward steps run top-to-bottom; on failure, compensations run bottom-to-top to restore consistency. Python's native `try/except` expresses this cleanly when the durable function uses `await ctx.run(...)` for each step.

For the TS version's mental model, see `resonate-saga-pattern-typescript`. The Python expression differs in syntax (`try/except` instead of `try/catch`, `async def` + `await` instead of `function*` + `yield`) and in how compensation lists are collected.

## When to use

- Multi-step workflow where intermediate state is visible to other consumers (charge a card → create shipment)
- Each step is idempotent or inexpensive to retry individually
- Compensation logic exists for every committed step
- You need "all or nothing" consistency without a distributed transaction

**Don't use** when steps are a pure in-memory pipeline (use sequential `ctx.run` calls without compensation), or when you have access to a single DB transaction that covers all steps.

## Basic shape

```python
from __future__ import annotations
import asyncio, os, time
from typing import TYPE_CHECKING
from resonate.resonate import Resonate
from resonate.retry import Never

if TYPE_CHECKING:
    from resonate.context import Context

r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"), retry_policy=Never())

async def place_order(ctx: Context, order_id: str) -> dict:
    # Step 1: reserve inventory
    flight = await ctx.run(reserve_flight, order_id)

    # Step 2: charge (compensate flight on failure)
    try:
        hotel = await ctx.run(reserve_hotel, order_id)
    except Exception:
        await ctx.run(release_flight, flight)
        raise

    # Step 3: charge (compensate hotel + flight on failure)
    try:
        charge = await ctx.run(charge_card, order_id)
    except Exception:
        await ctx.run(release_hotel, hotel)
        await ctx.run(release_flight, flight)
        raise

    return {"status": "success", "order_id": order_id,
            "flight": flight, "hotel": hotel, "charge": charge}
```

Each forward step is a durable checkpoint — if the worker crashes mid-saga, Resonate resumes from the last successful `await ctx.run(...)`. Compensations run only when the `except` block is entered, so a partially-completed saga's compensations are themselves durable.

## The canonical SDK example (trip booking)

The SDK's `examples/saga` shows a three-step trip: `reserve_flight` → `reserve_hotel` → `charge_card`, with per-step compensations and a `Never()` retry policy on the top-level Resonate instance (so a step that deterministically fails doesn't spin forever):

```python
from __future__ import annotations
import asyncio, os, time
from typing import TYPE_CHECKING
from resonate.resonate import Resonate
from resonate.retry import Never

if TYPE_CHECKING:
    from resonate.context import Context

class BookingError(Exception): ...

async def reserve_flight(ctx: Context, customer: str, frm: str, to: str) -> str:
    ref = f"FL-{customer}-{frm}-{to}"
    print(f"  reserved {ref}")
    return ref

async def reserve_hotel(ctx: Context, customer: str, city: str, fail: bool) -> str:
    if fail:
        raise BookingError(f"no rooms available in {city}")
    ref = f"HT-{customer}-{city}"
    print(f"  reserved {ref}")
    return ref

async def release_flight(ctx: Context, ref: str) -> str:
    print(f"  released {ref}")
    return ref

async def release_hotel(ctx: Context, ref: str) -> str:
    print(f"  released {ref}")
    return ref

async def book_trip(
    ctx: Context, customer: str, frm: str, to: str, amount: int, fail_at: str
) -> tuple[str, str]:
    flight = await ctx.run(reserve_flight, customer=customer, frm=frm, to=to)

    try:
        hotel = await ctx.run(
            reserve_hotel, customer=customer, city=to, fail=fail_at == "hotel"
        )
    except BookingError:
        await ctx.run(release_flight, ref=flight)
        raise

    return flight, hotel

async def main() -> None:
    r = Resonate(
        url=os.environ.get("RESONATE_URL", "http://localhost:8001"),
        retry_policy=Never(),
    )
    r.register(book_trip)
    try:
        handle = r.run(f"saga-{time.time_ns()}", book_trip, "alice", "SFO", "JFK", 850, "")
        flight, hotel = await handle.result()
        print(f"OK: flight={flight} hotel={hotel}")
    finally:
        await r.stop()

asyncio.run(main())
```

## Explicit step objects

For sagas with many steps, an explicit list is easier to reason about:

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class SagaStep:
    name: str
    forward: Callable
    backward: Callable

STEPS = [
    SagaStep("flight", reserve_flight, release_flight),
    SagaStep("hotel", reserve_hotel, release_hotel),
    SagaStep("charge", charge_card, refund_payment),
]

async def place_order(ctx: Context, order_id: str) -> dict:
    completed: list[tuple[SagaStep, str]] = []

    try:
        for step in STEPS:
            ref = await ctx.run(step.forward, order_id)
            completed.append((step, ref))
        return {"status": "success", "order_id": order_id}

    except Exception as err:
        for step, ref in reversed(completed):
            await ctx.run(step.backward, ref)
        return {"status": "failed", "order_id": order_id, "reason": str(err)}
```

## Compensation must itself be retryable

A compensation that fails stalls the saga in an inconsistent state. Guard against this:

```python
from resonate.retry import Exponential

async def place_order(ctx: Context, order_id: str) -> dict:
    completed: list[tuple[SagaStep, str]] = []
    try:
        # ... forward steps ...
        pass
    except Exception:
        for step, ref in reversed(completed):
            # Aggressive retry: compensations MUST eventually succeed
            await ctx.options(
                retry_policy=Exponential(),
            ).run(step.backward, ref)
        raise
```

If a compensation exhausts its retry policy, you have a real inconsistency — log it, alert on-call, and resolve manually.

## Distinct Python idioms

- **`async def` + `await`** — durable functions are coroutines, not generators. Every step uses `await ctx.run(...)`.
- **`try/except` over `try/catch`** — catch block is `except BookingError:` or `except Exception:`. Catch specific domain exceptions rather than bare `Exception` when you can, so internal SDK exceptions (like `SuspendedError`) are not accidentally swallowed.
- **`reversed(completed)`** is a Python built-in that returns a reverse iterator without mutating the list.
- **Dataclasses** (or pydantic) are the idiomatic way to model step objects, not interfaces or generics.

## Avoid: compensation that mutates in-flight state

Bad:

```python
async def refund_payment(ctx: Context, order_id: str) -> None:
    payment = PAYMENTS_DB.get(order_id)
    payment.status = "refunded"          # in-memory mutation; invisible to the system
    PAYMENTS_DB.save(payment)
```

Make the compensation a pure operation against the system of record:

```python
async def refund_payment(ctx: Context, order_id: str) -> None:
    payments_api.refund(order_id)        # idempotent by order_id; safe to retry
```

## Common variants

### Nested sub-saga

A single step can be its own saga dispatched via rpc:

```python
async def place_order(ctx: Context, order_id: str) -> dict:
    try:
        await ctx.rpc("reserve_inventory_saga", order_id)   # sub-saga in another worker
        await ctx.run(charge_payment, order_id)
        await ctx.run(create_shipment, order_id)
    except Exception:
        # Parent compensation; nested saga compensates itself internally
        await ctx.run(cancel_order, order_id)
        raise
```

### Parallel fan-out inside a saga

```python
async def place_order(ctx: Context, order_id: str) -> dict:
    try:
        await ctx.run(reserve_inventory, order_id)

        # Kick off two independent steps in parallel
        p1_future = ctx.run(charge_payment, order_id)
        p2_future = ctx.run(send_confirmation_email, order_id)
        await p1_future
        await p2_future

        await ctx.run(create_shipment, order_id)
    except Exception:
        await ctx.run(compensate_all, order_id)
        raise
```

Both `charge_payment` and `send_confirmation_email` are durable; if either fails, the `except` block runs and compensates.

## Related skills

- `resonate-basic-durable-world-usage-python` — `ctx.run`, `ctx.options`, `ctx.rpc`
- `resonate-recursive-fan-out-pattern-python` — parallel patterns inside or alongside sagas
- `resonate-external-system-of-record-pattern-python` — when the "saga" is actually just writing to one authoritative system with idempotency
- `durable-execution` — foundational concept; replay semantics make compensation safe
