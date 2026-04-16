---
name: resonate-basic-durable-world-usage-rust
description: Core patterns for writing Resonate durable functions in Rust using the #[resonate::function] attribute — function kinds (Workflow, Leaf with Info, Pure leaf), Context APIs (ctx.run, ctx.rpc, ctx.sleep with Duration, .spawn() for parallelism, builder options), and the Result<T> return convention. Use when writing any Rust function decorated with #[resonate::function]. v0.1.0 caveat: APIs may change between releases.
license: Apache-2.0
---

# Resonate Basic Durable World Usage — Rust

> **v0.1.0 caveat.** The Resonate Rust SDK is in active development (not yet on crates.io). API surface described here matches `docs/develop/rust.mdx` as of April 2026. Verify against the current SDK source before relying on any specific shape.

## Overview

Durable functions in Rust are async functions decorated with `#[resonate::function]`. The macro wraps your function in Resonate's durable-execution machinery — every successful `ctx.run` / `ctx.rpc` / `ctx.sleep` is a checkpoint, and the function resumes from the last checkpoint on process restart.

This skill covers the Context API surface used inside those functions. The ephemeral-world counterpart (registration, top-level invocation, promises) lives in `resonate-basic-ephemeral-world-usage-rust`.

## The contract

- **Attribute macro:** `#[resonate::function]` (or `#[resonate::function(name = "alias")]`)
- **Async function:** `async fn` — tokio is the default runtime
- **Return type:** `Result<T>` (aliased from `resonate::error::Result`) — use `?` for propagation
- **First parameter inferred kind:**

| First param | Kind | Capabilities |
|---|---|---|
| `&Context` | Workflow | `ctx.run`, `ctx.rpc`, `ctx.sleep`, `.spawn()` parallelism, context accessors |
| `&Info` | Leaf with metadata | Read-only access to `info.id()`, `info.parent_id()`, etc. |
| value types (`String`, `MyStruct`) | Pure leaf | No context; stateless computation |

Minimal workflow shape:

```rust
use resonate::prelude::*;

#[resonate::function]
async fn process_order(ctx: &Context, order_id: String) -> Result<String> {
    let order = ctx.run(load_order, order_id.clone()).await?;
    let charge = ctx.rpc::<String>("charge_card", order.clone()).await?;
    Ok(format!("order={} charge={}", order, charge))
}

#[resonate::function]
async fn load_order(order_id: String) -> Result<String> {
    Ok(format!("order-{}", order_id))
}
```

If this workflow crashes after `load_order` but before `rpc("charge_card")`, it resumes at the `rpc` call on restart — `load_order` is NOT re-executed; its stored result is returned.

## `ctx.run` — same-process invocation

### Sequential (await directly)

```rust
#[resonate::function]
async fn foo(ctx: &Context, input: String) -> Result<String> {
    let a = ctx.run(step_a, input.clone()).await?;
    let b = ctx.run(step_b, a).await?;
    Ok(b)
}
```

### Parallel (`.spawn()` returns a DurableFuture)

```rust
#[resonate::function]
async fn foo(ctx: &Context, input: String) -> Result<(String, String)> {
    let fut_a = ctx.run(step_a, input.clone()).spawn().await?;
    let fut_b = ctx.run(step_b, input).spawn().await?;

    let a = fut_a.await?;
    let b = fut_b.await?;

    Ok((a, b))
}
```

`.spawn()` starts the sub-task without blocking; the returned `DurableFuture` is awaited later. This is the Rust analog of TS's `begin_run` / Python's `begin_run` — different shape, same semantics.

## `ctx.rpc` — remote-process invocation

The registered name is a string; the result type needs a turbofish:

```rust
#[resonate::function]
async fn orchestrator(ctx: &Context, batch_id: String) -> Result<()> {
    let _result = ctx.rpc::<String>("worker_fn", batch_id).await?;
    Ok(())
}
```

With target group + parallelism:

```rust
#[resonate::function]
async fn parallel_remote(ctx: &Context) -> Result<()> {
    let f1 = ctx.rpc::<String>("worker-a", "data".into())
        .target("poll://any@group-a")
        .spawn()
        .await?;

    let f2 = ctx.rpc::<String>("worker-b", "data".into())
        .target("poll://any@group-b")
        .spawn()
        .await?;

    f1.await?;
    f2.await?;
    Ok(())
}
```

## `ctx.sleep` — durable sleep

`ctx.sleep` takes a `std::time::Duration`. There is no upper limit; sleeps survive process restarts because the continuation lives in the Resonate server:

```rust
use std::time::Duration;

#[resonate::function]
async fn daily_digest(ctx: &Context, user_id: String) -> Result<()> {
    loop {
        ctx.sleep(Duration::from_secs(24 * 60 * 60)).await?;
        ctx.rpc::<()>("send_digest", user_id.clone()).await?;
    }
}
```

## Builder options

All Context execution methods return a builder that accepts options before `.await` or `.spawn()`:

```rust
use std::time::Duration;

#[resonate::function]
async fn foo(ctx: &Context) -> Result<String> {
    let result: String = ctx.run(expensive_leaf, "input".into())
        .timeout(Duration::from_secs(30))
        .await?;
    Ok(result)
}
```

Documented Context builder options:

| Method | Applies to | Purpose |
|---|---|---|
| `.timeout(Duration)` | `ctx.run`, `ctx.rpc` | Execution timeout |
| `.target(&str)` | `ctx.rpc` only | Worker group routing (`poll://any@group-name`) |

Note: `.version(u32)` and `.tags(HashMap<String, String>)` are **ephemeral-world only** (on `resonate.run()` / `resonate.rpc()`) — they are NOT available on Context builders.

## Context accessors

Inside a workflow, `&Context` provides read-only metadata:

```rust
#[resonate::function]
async fn foo(ctx: &Context) -> Result<()> {
    println!("id={}", ctx.id());
    println!("parent={}", ctx.parent_id());
    println!("origin={}", ctx.origin_id());
    println!("func={}", ctx.func_name());
    println!("timeout_at={}", ctx.timeout_at());
    Ok(())
}
```

`origin_id` is the top-level invocation that kicked off this call graph; useful for tracing across RPC hops.

`ctx.info()` returns a snapshot `Info` struct with a superset of these accessors, including `branch_id()` and `tags()` (a `&HashMap<String, String>` of tags set at invocation time). Useful when you want to hand execution metadata to a helper function without passing the whole `Context`.

## Dependency access: `ctx.get_dependency::<T>()`

Dependencies set on the `Resonate` instance with `.with_dependency<T>(value)` (see `resonate-basic-ephemeral-world-usage-rust`) are retrieved inside a durable function by type:

```rust
use std::sync::Arc;
use sqlx::PgPool;

#[resonate::function]
async fn write_order(ctx: &Context, order_id: String) -> Result<()> {
    let pool: Arc<PgPool> = ctx.get_dependency::<PgPool>();
    // wrap I/O in a leaf so the effect is checkpointed
    ctx.run(insert_order_row, (pool.clone(), order_id)).await?;
    Ok(())
}

async fn insert_order_row((pool, order_id): (Arc<PgPool>, String)) -> Result<()> {
    sqlx::query("INSERT INTO orders (id) VALUES ($1) ON CONFLICT DO NOTHING")
        .bind(&order_id)
        .execute(&*pool)
        .await?;
    Ok(())
}
```

Type-dispatched: there is one dependency per type per `Resonate` instance. For multiple values of the same logical type (e.g. two Postgres pools pointing at different databases), wrap them in newtypes and register each separately. `ctx.get_dependency::<T>()` panics if no dependency of that type was registered — wire your DI at startup and keep it static, not conditional.

`&Info` also exposes `get_dependency::<T>()`, so a leaf that takes `info: &Info` as its first parameter can access dependencies without needing the full Context.

> **Note:** `ctx.get_dependency` + `Info::get_dependency` are in the v0.1.0 SDK source (`resonate-sdk-rs:resonate/src/context.rs:115`, `info.rs:42`) but not yet covered in `docs/develop/rust.mdx` as of April 2026. Rust-skill review discovered the docs lag; the API is real.

## Human-in-the-loop: `ctx.promise::<T>()`

Context-side promises let a durable function block until an external actor (webhook, UI, CLI, operator) resolves or rejects. The returned `PromiseTask<T>` is a lazy builder: you can attach `.timeout(Duration)` and `.data(&impl Serialize)`, fetch the generated ID via `.id().await?`, eagerly `create()` a handle for later awaiting, or `.await` it directly to block.

```rust
use serde::{Serialize, Deserialize};
use std::time::Duration;

#[derive(Serialize, Deserialize)]
struct Decision {
    approved: bool,
    reviewer: String,
}

#[resonate::function]
async fn expense_approval(ctx: &Context, expense_id: String) -> Result<String> {
    // create a promise the reviewer will resolve from outside
    let decision: Decision = ctx
        .promise::<Decision>()
        .timeout(Duration::from_secs(24 * 60 * 60)) // 24-hour SLA
        .data(&serde_json::json!({ "expense_id": expense_id }))?
        .await?;                                     // blocks until resolved

    if decision.approved {
        ctx.run(process_reimbursement, expense_id.clone()).await?;
        Ok(format!("approved by {}", decision.reviewer))
    } else {
        Ok(format!("rejected by {}", decision.reviewer))
    }
}
```

From outside the worker, resolve it with the ephemeral-world `resonate.promises.resolve(...)` API using the same ID. Since the ID is SDK-generated per-invocation, the typical pattern is: fetch `task.id().await?` before awaiting, stash it somewhere the reviewer can read (DB, webhook payload, UI), then `await` the promise.

For the deep HITL pattern (multi-approver, webhooks, SLAs), see `resonate-human-in-the-loop-pattern-rust`.

> **Note:** `ctx.promise::<T>()` is in the v0.1.0 SDK source (`resonate-sdk-rs:resonate/src/context.rs:352`) with a full `PromiseTask<T>` builder (`.timeout`, `.data`, `.id`, `.create`, `.await`) but not yet covered in `docs/develop/rust.mdx` as of April 2026.

## Leaf with `&Info`

A pure leaf that still needs execution metadata uses `&Info` as its first parameter:

```rust
#[resonate::function]
async fn stamped_log(info: &Info, message: String) -> Result<()> {
    println!("[{}] {}", info.id(), message);
    Ok(())
}
```

`&Info` is read-only — no sub-task invocation, no sleep — which lets the SDK optimize its execution differently from workflows.

## Common shapes

**Sequential pipeline with typed `?` propagation:**

```rust
#[resonate::function]
async fn ingest(ctx: &Context, url: String) -> Result<usize> {
    let raw = ctx.run(fetch, url).await?;
    let parsed = ctx.run(parse, raw).await?;
    let count = ctx.run(persist, parsed).await?;
    Ok(count)
}
```

**Parallel fan-out over a known set:**

```rust
#[resonate::function]
async fn enrich_batch(ctx: &Context, ids: Vec<String>) -> Result<Vec<String>> {
    let mut futures = Vec::with_capacity(ids.len());
    for id in ids {
        futures.push(ctx.run(enrich_one, id).spawn().await?);
    }
    let mut results = Vec::with_capacity(futures.len());
    for f in futures {
        results.push(f.await?);
    }
    Ok(results)
}
```

**Error mapping at a step boundary:**

```rust
#[resonate::function]
async fn resilient(ctx: &Context, input: String) -> Result<String> {
    match ctx.rpc::<String>("flaky_worker", input.clone()).await {
        Ok(v) => Ok(v),
        Err(_) => ctx.run(fallback, input).await,
    }
}
```

## Distinct Rust idioms

- **`#[resonate::function]` attribute macro** — not a higher-order function call like TS's `resonate.register`. Registration still happens ephemeral-side; the attribute just transforms the function's signature for the SDK to pick up
- **Turbofish on `ctx.rpc::<T>`** — Rust needs the result type when it can't be inferred from the call site
- **`.spawn().await?`** pattern for parallelism — the double-await is real: `.spawn()` returns a `DurableFuture` (which itself is a future that resolves to the DurableFuture), and then you await the `DurableFuture` later to get the `T`
- **`Duration::from_secs` / `from_millis` / `from_millis` / `from_micros`** — never raw numeric literals for time
- **`Result<T>` + `?` everywhere** — idiomatic Rust propagation; matches the SDK's uniform return shape
- **`tokio::main` runtime** — async without it requires boilerplate; every Resonate Rust app uses `#[tokio::main]` on its entry point
- **Dependencies via type-dispatched `ctx.get_dependency::<T>()`** — attach at `Resonate::new(...).with_dependency(value)` time; retrieve in any durable function by type. Newtype if you need multiple values of the same logical type

## Rust SDK API coverage status

Honest reading of v0.1.0 as of April 2026, verified against `resonate-sdk-rs` source (not just docs):

### In the SDK source + documented in `rust.mdx`
- `ctx.run` / `ctx.rpc` / `ctx.sleep` with builders + `.spawn()` parallelism
- Context accessors `ctx.id/parent_id/origin_id/func_name/timeout_at`
- `Info` accessors on leaf-with-info functions

### In the SDK source BUT not in `rust.mdx` (safe to use; docs lag)
- **`ctx.promise::<T>()`** — Context-side HITL primitive with `PromiseTask<T>` builder (`.timeout`, `.data`, `.id`, `.create`, `.await`). Fully featured; discovered in post-session audit against `resonate/src/context.rs:352`
- **`ctx.get_dependency::<T>()`** — type-dispatched DI. Panics if not registered. `Info::get_dependency` is the same shape on leaves
- **`ctx.info()`** — returns an `Info` snapshot with extra accessors (`branch_id()`, `tags()`)

Each has been verified as `pub fn` with source-level docstrings; they appear intended for users. Treat them as "supported but not yet in rust.mdx; verify signatures against `resonate-sdk-rs` at the commit you depend on."

### NOT in the SDK source at v0.1.0
- **`ctx.detached`** — no fire-and-forget; use `ctx.rpc(...).spawn().await?` and simply don't await the returned future, or start a separate RPC without `.await`
- **`ctx.random.random()` / `ctx.time.time()`** — no deterministic randomness/time helpers. Standard Rust `SystemTime::now()` and `rand` will cause replay inconsistency. Workaround: do non-deterministic work inside a leaf so the value is checkpointed
- **`ctx.panic()` / `ctx.assert()`** — use standard `panic!` / `assert!` (non-recoverable; retry policy sees them). For recoverable invariant checks, propagate via `Result`

Each of these may land in a later v0.x; write your Rust workflows around the available set for now.

## Related skills

- `resonate-basic-ephemeral-world-usage-rust` — Client APIs, registration, top-level invocation, promises
- `resonate-basic-debugging-rust` — v0.1.0-specific failure modes, git-dep install, serde errors
- `durable-execution` + `resonate-philosophy` — foundational concepts
- `resonate-basic-durable-world-usage-typescript` + `-python` — sibling SDKs for comparison
