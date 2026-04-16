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
- **Dependencies via closures or module-level statics** — no `ctx.get_dependency`; capture `Arc<T>` in a closure registered via `resonate.register`, or pass dependencies as arguments

## What is NOT documented in v0.1.0

Honest documentation of gaps vs TS + Python (per iter-14 CHARTER rule: don't invent SDK surface):

- **`ctx.promise()`** — not documented for Rust. HITL patterns that work cleanly in TS + Python don't have a documented Rust equivalent at v0.1.0. Use ephemeral-world `resonate.promises.*` from a sibling process and coordinate via `ctx.rpc` until Context-level promises land
- **`ctx.detached`** — no fire-and-forget documented; use a separate `ctx.rpc` or spawn-and-ignore
- **`ctx.get_dependency` / `ctx.set_dependency`** — no dependency-injection API documented
- **`ctx.random.random()` / `ctx.time.time()`** — no deterministic-randomness / deterministic-time helpers documented; standard Rust `std::time::SystemTime::now()` and `rand` will cause replay inconsistency
- **`ctx.panic()` / `ctx.assert()`** — not documented; use standard Rust `panic!` / `assert!` (which produce a non-recoverable error that the retry policy sees)

Each of these may land in a later v0.x; write your Rust workflows around the documented set for now.

## Related skills

- `resonate-basic-ephemeral-world-usage-rust` — Client APIs, registration, top-level invocation, promises
- `resonate-basic-debugging-rust` — v0.1.0-specific failure modes, git-dep install, serde errors
- `durable-execution` + `resonate-philosophy` — foundational concepts
- `resonate-basic-durable-world-usage-typescript` + `-python` — sibling SDKs for comparison
