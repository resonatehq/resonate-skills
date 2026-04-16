---
name: resonate-saga-pattern-rust
description: Implement saga patterns for distributed transactions in Rust with Resonate — forward steps with compensating actions that unwind on failure using Result<T> and match-based compensation dispatch. Use when coordinating multi-step Rust workflows that need consistency across failures. v0.1.0 caveat: API surface may change between Rust SDK releases.
license: Apache-2.0
---

# Resonate Saga Pattern — Rust

> **v0.1.0 caveat.** The Rust SDK is in active development. The saga pattern below uses only documented v0.1.0 surface (`ctx.run`, `Result<T>`, `?` propagation). Verify against the current SDK source before shipping.

## Overview

A saga is a long-running transaction split into smaller steps, each with a compensating action. Forward steps run top-to-bottom; on failure, compensations run bottom-to-top. Rust expresses this naturally via `Result<T>` + `?` for forward propagation and a tracked "completed" list for compensation dispatch.

For the language-agnostic mental model, see `resonate-saga-pattern-typescript`.

## When to use

- Multi-step workflow where intermediate state is visible to other systems
- Each step is idempotent or cheap to retry individually
- Compensation logic exists for every committed step
- You need "all or nothing" consistency without a distributed transaction

## Basic shape

```rust
use resonate::prelude::*;
use serde::{Serialize, Deserialize};

#[derive(Clone, Serialize, Deserialize)]
enum Step { Inventory, Payment, Shipment }

#[derive(Clone, Serialize, Deserialize)]
struct SagaResult {
    status: String,
    order_id: String,
    compensated: Vec<String>,
}

#[resonate::function]
async fn place_order(ctx: &Context, order_id: String) -> Result<SagaResult> {
    let mut completed: Vec<Step> = Vec::new();

    // attempt the forward path
    let result = try_forward(ctx, &order_id, &mut completed).await;

    match result {
        Ok(()) => Ok(SagaResult {
            status: "success".into(),
            order_id,
            compensated: vec![],
        }),
        Err(_err) => {
            // compensate in reverse order
            let mut comp_names = Vec::new();
            for step in completed.iter().rev() {
                ctx.run(compensate, (step.clone(), order_id.clone()))
                    .await?;
                comp_names.push(format!("{:?}", step));
            }
            Ok(SagaResult {
                status: "failed".into(),
                order_id,
                compensated: comp_names,
            })
        }
    }
}

async fn try_forward(
    ctx: &Context,
    order_id: &str,
    completed: &mut Vec<Step>,
) -> Result<()> {
    ctx.run(reserve_inventory, order_id.to_string()).await?;
    completed.push(Step::Inventory);

    ctx.run(charge_payment, order_id.to_string()).await?;
    completed.push(Step::Payment);

    ctx.run(create_shipment, order_id.to_string()).await?;
    completed.push(Step::Shipment);

    Ok(())
}

#[resonate::function]
async fn compensate((step, order_id): (Step, String)) -> Result<()> {
    match step {
        Step::Shipment => { /* cancel_shipment(&order_id) */ Ok(()) }
        Step::Payment => { /* refund_payment(&order_id) */ Ok(()) }
        Step::Inventory => { /* release_inventory(&order_id) */ Ok(()) }
    }
}

#[resonate::function]
async fn reserve_inventory(order_id: String) -> Result<()> { Ok(()) }

#[resonate::function]
async fn charge_payment(order_id: String) -> Result<()> { Ok(()) }

#[resonate::function]
async fn create_shipment(order_id: String) -> Result<()> { Ok(()) }
```

Each forward step is a durable checkpoint. If the worker crashes mid-saga, Resonate resumes from the last successful `yield` (in Rust, the last successful `.await?`).

## Compensation must be retryable

A compensation that fails stalls the saga in an inconsistent state:

```rust
use std::time::Duration;

for step in completed.iter().rev() {
    ctx.run(compensate, (step.clone(), order_id.clone()))
        .timeout(Duration::from_secs(600)) // 10 min per compensation
        .await?;
}
```

If a compensation exhausts its timeout/retry, you have a real inconsistency — log and alert.

## Explicit step objects via enum dispatch

Rust's enum + match is idiomatic for a small, closed set of saga steps:

```rust
#[derive(Clone, Serialize, Deserialize)]
enum ForwardStep { Inventory, Payment, Shipment }

impl ForwardStep {
    async fn run(&self, ctx: &Context, order_id: &str) -> Result<()> {
        match self {
            ForwardStep::Inventory => ctx.run(reserve_inventory, order_id.to_string()).await?,
            ForwardStep::Payment => ctx.run(charge_payment, order_id.to_string()).await?,
            ForwardStep::Shipment => ctx.run(create_shipment, order_id.to_string()).await?,
        };
        Ok(())
    }
}
```

For large sets of dynamic steps, prefer `Vec<Box<dyn StepTrait>>` — but that adds dyn-trait complexity beyond the basic pattern.

## Distinct Rust idioms

- **`Result<T>` + `?`** handles forward-path propagation naturally — no `try/except` needed for "raise on first error"
- **Explicit `let result = ... .await` + `match`** for the catch-and-compensate branch, since `?` would bail before compensation runs
- **`completed.iter().rev()`** for reverse iteration — idiomatic Rust, no `reversed()` builtin needed
- **`enum` + `match`** for compensation dispatch — cleaner than string-based switches
- **`(Step, String)` tuple parameter** to bundle args into a single serializable input for `ctx.run` — the SDK serializes the whole argument, so tuples work naturally
- **`Clone` derive** on step enums — needed because `iter().rev()` borrows and compensation needs owned values

## Avoid

- Panicking inside a durable function (`panic!`, `unreachable!`, `assert!`) — panics in v0.1.0 don't compensate; use `Result` explicitly
- Non-idempotent compensations — each compensation may retry
- Compensations that read the SoR without a fresh `ctx.run` — the read happens inside a leaf for checkpointing

## Related skills

- `resonate-basic-durable-world-usage-rust` — `ctx.run`, builder options, function kinds
- `resonate-recursive-fan-out-pattern-rust` — parallel fan-out, can be combined with sagas
- `durable-execution` — foundational replay semantics
- `resonate-saga-pattern-typescript` / `-python` — sibling SDKs for comparison
