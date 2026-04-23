---
name: resonate-external-system-of-record-pattern-rust
description: Maintain consistency across external systems in Rust Resonate workflows by treating one system as the source of truth and writing to it idempotently before any dependent effects. Uses type-dispatched ctx.get_dependency::<T>() for DB/client injection. Use when Resonate's durable promises coordinate writes to PostgreSQL, TigerBeetle, Stripe, or any external store with its own durability. v0.1.0 caveat: ctx.get_dependency + with_dependency are in sdk-rs source but not yet in rust.mdx.
license: Apache-2.0
---

# Resonate External System of Record Pattern — Rust

> **v0.1.0 caveat.** `ctx.get_dependency::<T>()` and `resonate.with_dependency::<T>(value)` are real `pub fn`s in the Rust SDK source (`resonate-sdk-rs:resonate/src/context.rs:115`, `resonate.rs:271`) but are not yet covered in `docs/develop/rust.mdx` as of April 2026. The APIs are safe to use; cite source paths when reviewers ask.

## Overview

When a Rust workflow touches an external system with its own durability (a database, a ledger, a message broker), that system often is or should be the *system of record* (SoR). Resonate coordinates the workflow and guarantees at-least-once execution; the SoR enforces consistency via its own primitives (transactions, idempotency keys, CAS operations).

Rust's type-dispatched dependency injection makes this pattern particularly clean: register typed resources once at process start with `resonate.with_dependency<T>(value)`, retrieve inside durable functions with `ctx.get_dependency::<T>()`.

For the language-agnostic framing, see `resonate-external-system-of-record-pattern-typescript`. Rust's shape adds: `Arc<T>` returns from DI, serde-derived input types, and idiomatic `sqlx` / ledger-crate usage.

## Core principle

> **Write to the system of record first. Read from it as ground truth. Never let Resonate's promise state contradict the SoR.**

Resonate stores workflow state (what step succeeded, what the result was). The SoR stores business state (the account balance, the order status). When they disagree, the SoR wins; Resonate's job is to converge toward it.

## Dependency setup (ephemeral world)

```rust
use resonate::prelude::*;
use sqlx::PgPool;
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<()> {
    let pool = PgPool::connect(&std::env::var("DATABASE_URL")?).await?;
    let stripe = stripe::Client::new(std::env::var("STRIPE_SECRET_KEY")?);

    let resonate = Resonate::new(ResonateConfig::default())
        .with_dependency(pool)
        .with_dependency(stripe);

    resonate.register(create_order).unwrap();
    resonate.register(charge_card).unwrap();

    tokio::signal::ctrl_c().await?;
    resonate.stop().await?;
    Ok(())
}
```

Dependencies are retrieved inside durable functions by type via `ctx.get_dependency::<T>()`, which returns `Arc<T>`.

## Basic shape

```rust
use resonate::prelude::*;
use serde::{Serialize, Deserialize};
use sqlx::PgPool;
use std::sync::Arc;

#[derive(Clone, Serialize, Deserialize)]
struct OrderInput {
    order_id: String,
    customer_id: String,
    amount_cents: i64,
}

#[resonate::function]
async fn create_order(ctx: &Context, input: OrderInput) -> Result<String> {
    // SoR write first (idempotent against retries via ON CONFLICT)
    ctx.run(insert_order_row, input.clone()).await?;

    // dependent effects only after the SoR write succeeds
    ctx.run(send_confirmation_email, input.customer_id.clone()).await?;
    ctx.rpc::<()>("enqueue_fulfillment", input.order_id.clone()).await?;

    Ok(input.order_id)
}

#[resonate::function]
async fn insert_order_row(info: &Info, input: OrderInput) -> Result<()> {
    let pool: Arc<PgPool> = info.get_dependency::<PgPool>();

    sqlx::query(
        "INSERT INTO orders (id, customer_id, amount_cents, status)
         VALUES ($1, $2, $3, 'created')
         ON CONFLICT (id) DO NOTHING",
    )
    .bind(&input.order_id)
    .bind(&input.customer_id)
    .bind(input.amount_cents)
    .execute(&*pool)
    .await
    .map_err(|e| Error::from_reason(e.to_string()))?;

    Ok(())
}
```

The leaf uses `info: &Info` (which also has `get_dependency`) instead of `ctx: &Context` because a pure write doesn't need sub-invocation capabilities. Returning `Ok(())` lets the parent's `ctx.run(...).await?` record the checkpoint.

## Idempotency keys — the external side

Use Resonate's deterministic invocation IDs (or derivations of them) as the idempotency key in the external system:

```rust
#[resonate::function]
async fn charge_card(info: &Info, order_id: String) -> Result<String> {
    let pool: Arc<PgPool> = info.get_dependency::<PgPool>();
    let stripe: Arc<stripe::Client> = info.get_dependency::<stripe::Client>();

    // skip if we already charged
    let existing: Option<(String,)> = sqlx::query_as(
        "SELECT charge_id FROM orders WHERE id = $1 AND charge_id IS NOT NULL",
    )
    .bind(&order_id)
    .fetch_optional(&*pool)
    .await
    .map_err(|e| Error::from_reason(e.to_string()))?;

    if let Some((charge_id,)) = existing {
        return Ok(charge_id); // already charged; Stripe dedupes on its side too
    }

    // create with idempotency key — Stripe dedupes by this
    let idempotency_key = format!("order:{}:charge", order_id);
    let charge = stripe
        .post(&format!("/charges"))
        .header("Idempotency-Key", &idempotency_key)
        .send_json(&serde_json::json!({ "amount": 100, "currency": "usd" }))
        .await
        .map_err(|e| Error::from_reason(e.to_string()))?;

    // update SoR with the charge result
    sqlx::query(
        "UPDATE orders SET charge_id = $1, status = 'paid' WHERE id = $2",
    )
    .bind(&charge.id)
    .bind(&order_id)
    .execute(&*pool)
    .await
    .map_err(|e| Error::from_reason(e.to_string()))?;

    Ok(charge.id)
}
```

The `ctx.run(charge_card, order_id)` that calls this leaf is durably checkpointed; Stripe's idempotency key dedupes on its side. Both layers see exactly one charge even if Resonate retries the step.

## Reading from the SoR on resumption

Checkpointed `ctx.run` calls return the stored value on replay — the external call is not re-executed. When workflow logic needs the *current* SoR state (not the checkpointed snapshot from first-run), read it explicitly inside a leaf:

```rust
#[derive(Serialize, Deserialize)]
struct OrderRow {
    id: String,
    sku: String,
    quantity: i64,
}

#[resonate::function]
async fn fulfill_order(ctx: &Context, order_id: String) -> Result<String> {
    // checkpointed; returns stored value on replay
    let order: OrderRow = ctx.run(load_order, order_id.clone()).await?;

    // current SoR read — use for fresh data even across replay
    let current_inventory: i64 = ctx
        .run(check_inventory_now, order.sku.clone())
        .await?;

    if current_inventory < order.quantity {
        ctx.run(backorder_flag, order_id).await?;
        return Ok("backorder".into());
    }

    ctx.run(reserve_inventory, (order_id.clone(), order.quantity)).await?;
    Ok("fulfilling".into())
}

#[resonate::function]
async fn load_order(info: &Info, order_id: String) -> Result<OrderRow> {
    let pool: Arc<PgPool> = info.get_dependency::<PgPool>();
    let row = sqlx::query_as::<_, (String, String, i64)>(
        "SELECT id, sku, quantity FROM orders WHERE id = $1",
    )
    .bind(&order_id)
    .fetch_one(&*pool)
    .await
    .map_err(|e| Error::from_reason(e.to_string()))?;
    Ok(OrderRow { id: row.0, sku: row.1, quantity: row.2 })
}

#[resonate::function]
async fn check_inventory_now(info: &Info, sku: String) -> Result<i64> {
    let pool: Arc<PgPool> = info.get_dependency::<PgPool>();
    let (qty,): (i64,) = sqlx::query_as(
        "SELECT quantity FROM inventory WHERE sku = $1",
    )
    .bind(&sku)
    .fetch_one(&*pool)
    .await
    .map_err(|e| Error::from_reason(e.to_string()))?;
    Ok(qty)
}
```

Both are `ctx.run` calls → both are checkpointed. The difference is intent: `load_order` is the workflow's snapshot of the order; `check_inventory_now` is a *fresh* read each retry. The SoR is the tie-breaker.

## Ledger-as-SoR (TigerBeetle shape)

For financial workflows, a dedicated ledger (TigerBeetle, double-entry tables) is a stronger SoR choice:

```rust
struct Ledger(tigerbeetle_rs::Client);

#[derive(Serialize, Deserialize)]
struct Transfer {
    from_account: u128,
    to_account: u128,
    amount: i64,
    transfer_id: u128,
}

#[resonate::function]
async fn transfer_funds(ctx: &Context, t: Transfer) -> Result<String> {
    // the ledger rejects duplicate transfer_ids as part of its contract
    ctx.run(post_ledger_transfer, t.clone()).await?;

    // after ledger commit, these are safe
    ctx.run(notify_both_parties, t.clone()).await?;

    Ok(format!("transfer:{}", t.transfer_id))
}

#[resonate::function]
async fn post_ledger_transfer(info: &Info, t: Transfer) -> Result<()> {
    let ledger: Arc<Ledger> = info.get_dependency::<Ledger>();
    ledger
        .0
        .create_transfer(t.from_account, t.to_account, t.amount, t.transfer_id)
        .await
        .map_err(|e| Error::from_reason(e.to_string()))?;
    Ok(())
}

async fn notify_both_parties(t: Transfer) -> Result<()> {
    // ...
    Ok(())
}
```

On replay, `post_ledger_transfer` returns the stored checkpoint value — the ledger is hit exactly once across all retries, and the ledger's own idempotency on `transfer_id` is the belt-and-suspenders.

## Distinct Rust idioms

- **Type-dispatched DI**: `ctx.get_dependency::<PgPool>()` or `info.get_dependency::<Ledger>()` — no string keys. Newtype-wrap when you need multiple values of the same logical type (`struct PrimaryDb(PgPool)`, `struct AnalyticsDb(PgPool)`)
- **`Arc<T>` return** — dependencies come back as `Arc<T>`; deref with `&*arc` when passing to query functions that take `&Pool`
- **`sqlx::query_as` + tuple/struct derivation** — typed query results without a separate ORM
- **`.map_err(|e| Error::from_reason(e.to_string()))`** — convert sqlx/stripe/etc. errors into Resonate's `Error` for `?` propagation. Verify the exact conversion with your SDK version; `Error::from_reason` matches master-branch v0.1.0
- **`ON CONFLICT DO NOTHING` / `ON CONFLICT DO UPDATE`** (Postgres) — natural idempotency on inserts
- **`info: &Info` for leaves that only need DI** — skip `&Context` when the leaf doesn't orchestrate sub-tasks; signals intent (pure write) and lets the SDK optimize

## Avoid

- **Instantiating DB clients inside a durable function** — each replay would reconstruct the client; dependencies must be created once in ephemeral-world and shared via `with_dependency`
- **Writing to two systems without a clear SoR** — if both fail partially, who wins? Pick one SoR; make the other a compensatable side-effect or best-effort secondary write
- **In-memory mutations outside `ctx.run`** — not durable; replay loses the mutation. Always wrap state changes in a checkpointed leaf
- **Mixing `ctx: &Context` on a pure leaf** — signals intent poorly; leaves that only hit the SoR should use `info: &Info`

## Related skills

- `resonate-basic-durable-world-usage-rust` — `ctx.get_dependency::<T>()`, `info: &Info` leaves, builder options
- `resonate-basic-ephemeral-world-usage-rust` — `with_dependency::<T>(value)` registration
- `resonate-saga-pattern-rust` — when the SoR doesn't cover all steps and you need compensation
- `resonate-human-in-the-loop-pattern-rust` — when the SoR update waits on external decision
- `durable-execution` — foundational replay semantics; this pattern is checkpoint-centric
