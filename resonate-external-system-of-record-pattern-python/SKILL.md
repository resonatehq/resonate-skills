---
name: resonate-external-system-of-record-pattern-python
description: Maintain consistency across external systems in Python Resonate workflows by treating one system as the source of truth and writing to it idempotently before any dependent effects. Use when Resonate's durable promises coordinate writes to PostgreSQL, TigerBeetle, Kafka, or any external store that has its own durability contract.
license: Apache-2.0
---

# Resonate External System of Record Pattern — Python

## Overview

When a Resonate workflow touches an external system that has its own durability (a database, a ledger, a message broker), that external system often is or should be the *system of record* (SoR). Resonate's role is to coordinate the steps, guarantee at-least-once execution, and make each step idempotent against the SoR.

The pattern is language-agnostic in intent; the Python expression uses standard DB-API connections, TigerBeetle / similar clients via `set_dependency`, and the `ctx.run` envelope to ensure each write is checkpointed exactly once.

## Core principle

> **Write to the system of record first. Read from it as ground truth. Never let Resonate's promise state contradict the SoR.**

Resonate stores *its* state — what step succeeded, what the result was, which promise is pending. The SoR stores the business state — the account balance, the order status, the ledger entry. When these contradict, the SoR wins; Resonate's job is to converge toward it.

## Basic shape

```py
from resonate import Resonate
import psycopg

resonate = Resonate()
db = psycopg.connect(DATABASE_URL, autocommit=True)
resonate.set_dependency("db", db)

@resonate.register
def create_order(ctx, order_id: str, customer_id: str, amount: float):
    # write to SoR first with idempotency
    yield ctx.run(insert_order_row, order_id, customer_id, amount)

    # dependent effects only after SoR write succeeds
    yield ctx.run(send_confirmation_email, customer_id, order_id)
    yield ctx.run(enqueue_fulfillment, order_id)

    return {"order_id": order_id, "status": "created"}


def insert_order_row(ctx, order_id: str, customer_id: str, amount: float):
    db = ctx.get_dependency("db")
    # INSERT ... ON CONFLICT DO NOTHING — idempotent against retries
    db.execute(
        """
        INSERT INTO orders (id, customer_id, amount, status)
        VALUES (%s, %s, %s, 'created')
        ON CONFLICT (id) DO NOTHING
        """,
        (order_id, customer_id, amount),
    )
```

## Idempotency keys — the external side

Resonate's deterministic invocation ID gives you a stable identity across retries. Use it (or a derivation of it) as the idempotency key in the external system:

```py
def charge_card(ctx, order_id: str, amount: float):
    db = ctx.get_dependency("db")
    stripe = ctx.get_dependency("stripe")

    # skip if we already charged
    row = db.execute("SELECT charge_id FROM orders WHERE id = %s", (order_id,)).fetchone()
    if row and row[0]:
        return {"charge_id": row[0], "status": "already_charged"}

    # create with idempotency key — Stripe dedupes by this
    charge = stripe.charges.create(
        amount=int(amount * 100),
        currency="usd",
        idempotency_key=f"order:{order_id}:charge",
    )

    db.execute(
        "UPDATE orders SET charge_id = %s, status = 'paid' WHERE id = %s",
        (charge.id, order_id),
    )
    return {"charge_id": charge.id, "status": "charged"}
```

The Stripe call is durably checkpointed by Resonate; Stripe's own idempotency key dedupes on its side. Both sides see exactly one charge even if Resonate retries the step.

## Reading from the SoR on resumption

When a durable function replays, `ctx.run` returns the stored promise value for completed steps — the external call is NOT re-executed. But if your workflow logic needs the *current* SoR state (not the checkpointed value), read it explicitly:

```py
@resonate.register
def fulfill_order(ctx, order_id: str):
    # checkpointed; returns the stored value on replay
    order = yield ctx.run(load_order, order_id)

    # current SoR read — use this when the downstream step needs fresh data
    current_inventory = yield ctx.run(check_inventory_now, order.sku)

    if current_inventory < order.quantity:
        yield ctx.run(backorder_flag, order_id)
        return {"status": "backorder"}

    yield ctx.run(reserve_inventory, order_id, order.quantity)
    return {"status": "fulfilling"}
```

Both are inside `ctx.run` envelopes, so both are checkpointed. The difference is *which value* you treat as authoritative for downstream logic.

## TigerBeetle or a ledger-as-SoR

For financial systems, a dedicated ledger (TigerBeetle, double-entry tables) is a strong SoR choice. Resonate coordinates the surrounding workflow but delegates consistency to the ledger:

```py
@resonate.register
def transfer_funds(ctx, from_account: str, to_account: str, amount: int, transfer_id: str):
    # the ledger is the SoR; it rejects double-posts by transfer_id
    result = yield ctx.run(post_ledger_transfer, from_account, to_account, amount, transfer_id)

    if result.status != "posted":
        raise ValueError(f"ledger rejected transfer: {result.reason}")

    # after ledger commit, these are safe to do
    yield ctx.run(notify_both_parties, from_account, to_account, amount)
    return {"transfer_id": transfer_id, "status": "complete"}


def post_ledger_transfer(ctx, from_acc: str, to_acc: str, amount: int, transfer_id: str):
    tb = ctx.get_dependency("tigerbeetle")
    # TigerBeetle rejects duplicate transfer IDs as part of its API contract
    return tb.create_transfer(from_acc, to_acc, amount, id=transfer_id)
```

On replay, `post_ledger_transfer` returns the stored checkpoint value — the ledger is hit exactly once across all retries.

## Anti-patterns to avoid

**In-memory state outside `ctx.run`:**

```py
# BAD
@resonate.register
def bad(ctx, order_id: str):
    order = fetch_from_cache(order_id)  # ← not durable; cache may be gone on replay
    yield ctx.run(process, order)
```

Wrap cache reads (or any I/O) in a `ctx.run` helper so the value is checkpointed.

**Writing to two systems without a clear SoR:**

```py
# BAD — unclear which source of truth wins on partial failure
@resonate.register
def bad(ctx, order_id: str):
    yield ctx.run(write_to_postgres, order_id)    # step 1
    yield ctx.run(write_to_elasticsearch, order_id)  # step 2
```

If the Elasticsearch write fails and you retry, Resonate will skip the Postgres write (already checkpointed) but re-attempt ES — that's fine, as long as ES is idempotent. If it isn't, you have a consistency problem. The fix: pick the SoR (Postgres), make the other (ES) a compensatable side effect or a best-effort secondary write.

## Distinct Python idioms

- **`psycopg.connect(..., autocommit=True)`** for implicit transactions per statement; avoid the explicit `COMMIT` dance in durable functions (transactions are single-statement at the `ctx.run` level by convention).
- **`%s` parameter binding** (psycopg style) or `$1`-style for asyncpg; parameterize everything — SQL injection is a real concern in durable functions that take string inputs.
- **Resource dependencies via `ctx.get_dependency`** — never instantiate DB clients inside a durable function; they'd be recreated on every replay.
- **`ON CONFLICT DO NOTHING` / `ON CONFLICT DO UPDATE`** (Postgres) for natural idempotency on inserts.

## Related skills

- `resonate-basic-durable-world-usage-python` — `ctx.run`, `ctx.get_dependency`, options chain
- `resonate-saga-pattern-python` — when the SoR doesn't cover all steps and you need compensation
- `resonate-human-in-the-loop-pattern-python` — when the SoR update waits on external decision
- `durable-execution` — foundational replay semantics; this pattern depends on checkpoint semantics
