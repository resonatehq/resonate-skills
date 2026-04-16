---
name: resonate-basic-ephemeral-world-usage-rust
description: Core patterns for using the Resonate Rust SDK's Client APIs from the ephemeral world — initializing, registering durable functions with the #[resonate::function] attribute, invoking them top-level (run / rpc / schedule), getting handles, and managing external promises. Use when writing any Rust binary or process-level code that needs to launch or coordinate Resonate workflows. v0.1.0 caveat: APIs may change between releases.
license: Apache-2.0
---

# Resonate Basic Ephemeral World Usage — Rust

> **v0.1.0 caveat.** The Resonate Rust SDK is in active development. It is NOT yet published on crates.io; install as a git dependency. APIs may change between releases. Treat every code example as a moving target until the SDK reaches 1.0.

## Overview

The ephemeral world is anywhere your Rust program starts: `main()`, an Axum or Actix handler, a CLI entry point, a background service. You use the `Resonate` client to register durable functions and invoke them top-level. Once an invocation starts, it crosses into the durable world and Resonate guarantees its completion — retries on failure, resumes after crashes, continues across process restarts.

This skill covers the Client API surface. The Durable World (Context APIs inside `#[resonate::function]`) lives in `resonate-basic-durable-world-usage-rust`.

## Install

The Rust SDK is a git dependency (v0.1.0; not on crates.io yet):

```toml title="Cargo.toml"
[dependencies]
resonate = { git = "https://github.com/resonatehq/resonate-sdk-rs", branch = "master" }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

Most programs also want `serde_json` for the `json!(...)` macro used in the promises API.

```rust
use resonate::prelude::*;
```

## Initialize

Local mode (zero-dependency; in-memory promise store):

```rust
use resonate::prelude::*;

let resonate = Resonate::local();
```

Remote mode (connects to a Resonate server):

```rust
use resonate::prelude::*;

let resonate = Resonate::new(ResonateConfig {
    url: Some("http://localhost:8001".into()),
    ..Default::default()
});
```

With explicit worker group + auth token:

```rust
let resonate = Resonate::new(ResonateConfig {
    url: Some("https://resonate.example.com".into()),
    group: Some("workers".into()),
    token: std::env::var("RESONATE_TOKEN").ok(),
    ..Default::default()
});
```

The SDK reads environment variables when config fields are not set:

- `RESONATE_URL` — full base URL
- `RESONATE_HOST` + `RESONATE_PORT` — alternate construction
- `RESONATE_TOKEN` — JWT for authenticated servers
- `RESONATE_SCHEME` — `http` by default
- `RESONATE_PREFIX` — prepended to all promise + task IDs for multi-tenant namespacing

## Define a durable function

The `#[resonate::function]` attribute macro transforms an async function into a durable function. The SDK infers the function kind from the first parameter:

| First parameter | Kind | Semantics |
|---|---|---|
| `&Context` | Workflow | Can orchestrate sub-tasks via `ctx.run`, `ctx.rpc`, `ctx.sleep` |
| `&Info` | Leaf with metadata | Read-only access to execution metadata (id, parent, tags) |
| anything else (e.g., `String`, `OrderInput`) | Pure leaf | Stateless computation; no context access |

All durable functions MUST return `resonate::error::Result<T>` (aliased to `Result<T>` when you `use resonate::prelude::*`):

```rust
use resonate::prelude::*;

#[resonate::function]
async fn process_order(ctx: &Context, order_id: String) -> Result<String> {
    let order = ctx.run(load_order, order_id.clone()).await?;
    Ok(format!("processed {}", order))
}

#[resonate::function]
async fn load_order(order_id: String) -> Result<String> {
    // pure leaf — no context, no Info
    Ok(format!("order-{}", order_id))
}
```

Override the registered name for RPC cross-process identification:

```rust
#[resonate::function(name = "order-processor")]
async fn process_order(ctx: &Context, order_id: String) -> Result<String> {
    // ...
}
```

## Share resources via dependencies

Dependencies (DB pools, HTTP clients, config) are attached to the `Resonate` instance at construction time with the `.with_dependency<T>(value)` builder. The SDK uses Rust's type system (not a string key) — you retrieve a dependency by its type inside a durable function via `ctx.get_dependency::<T>()`.

```rust
use std::sync::Arc;
use sqlx::PgPool;

struct AppConfig {
    api_key: String,
}

#[tokio::main]
async fn main() -> Result<()> {
    let pool = PgPool::connect(&std::env::var("DATABASE_URL")?).await?;
    let config = AppConfig { api_key: std::env::var("API_KEY")? };

    let resonate = Resonate::new(ResonateConfig::default())
        .with_dependency(pool)
        .with_dependency(config);

    resonate.register(charge_card).unwrap();
    // ...
}

#[resonate::function]
async fn charge_card(ctx: &Context, order_id: String) -> Result<()> {
    let pool = ctx.get_dependency::<PgPool>();       // Arc<PgPool>
    let config = ctx.get_dependency::<AppConfig>();  // Arc<AppConfig>
    // use pool + config in a leaf (see resonate-basic-durable-world-usage-rust)
    Ok(())
}
```

Type-dispatched DI means you can only have one dependency per type per `Resonate` instance; for multiple values of the same logical type, wrap them in distinct newtypes (`struct PrimaryDb(PgPool)` vs `struct AnalyticsDb(PgPool)`).

> **Note:** `with_dependency` is a pub fn on `Resonate` in the v0.1.0 SDK source (`resonate-sdk-rs:resonate/src/resonate.rs:271`) but not mentioned in `docs/develop/rust.mdx` as of April 2026. Use with the understanding that docs lag source here.

## Register

Registration exposes a function to the Resonate system. Use `.unwrap()` or handle the `Result` (registration fails if the name is duplicated):

```rust
let resonate = Resonate::local();
resonate.register(process_order).unwrap();
resonate.register(load_order).unwrap();
```

One `.register` call per function per process. Register everything your process should handle before invoking anything.

## Invoke a durable function

### Same process, synchronous (awaits result)

```rust
let result: String = resonate
    .run("invocation-id", process_order, "order-123".into())
    .await?;
```

The builder accepts options before `.await`:

```rust
use std::time::Duration;

let result: String = resonate
    .run("order:123", process_order, "order-123".into())
    .timeout(Duration::from_secs(60))
    .version(2)
    .tags([("tenant".to_string(), "acme".to_string())].into())
    .await?;
```

`resonate.run` is the synchronous ephemeral-world entry. For async lifecycles where you want a handle, use `.get(id)` later.

### Remote process, synchronous

```rust
let result: String = resonate
    .rpc("invocation-id", "process_order", "order-123".into())
    .target("poll://any@workers")
    .await?;
```

The `target` option routes to a worker group; the called function must be registered in a process that polls that target.

### Scheduled (cron) invocation

The Rust SDK has first-class cron scheduling:

```rust
let schedule = resonate
    .schedule("daily-reconciliation", "0 2 * * *", "reconcile", "2026-04-16".into())
    .await?;

// ...later, to delete:
schedule.delete().await?;
```

(Python SDK does not expose a top-level `.schedule(...)` at v0.6.7; Rust does at v0.1.0. Cross-SDK parity is not yet achieved — see Related notes below.)

### Subscribe to an existing invocation

```rust
let mut handle = resonate.get::<String>("invocation-id").await?;
let result = handle.result().await?;
```

The type parameter on `.get::<T>` deserializes the result; you need to know or agree on T with the invocation site.

## External promises

External promises let code outside the durable function resolve or reject it — the primitive for human-in-the-loop workflows and webhook-driven resumption:

```rust
use serde_json::json;

// create a promise with a timeout (ms since Unix epoch), initial param, and tags
let timeout_at_ms: i64 = (std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH)?
    .as_millis() as i64) + 30 * 60 * 1000; // 30 minutes from now

resonate.promises.create("approval:order-123", timeout_at_ms, json!({}), json!({})).await?;

// elsewhere (webhook, CLI, UI):
resonate.promises.resolve("approval:order-123", json!({"approved": true})).await?;

// or reject:
resonate.promises.reject("approval:order-123", json!({"reason": "over-budget"})).await?;

// or cancel (settles as rejected_canceled):
resonate.promises.cancel("approval:order-123", json!(null)).await?;

// query state:
let promise = resonate.promises.get("approval:order-123").await?;
```

The `data` arguments are `serde_json::Value`; use the `json!` macro for ergonomic literals.

## Graceful shutdown

```rust
resonate.stop().await?;
```

Stops background tasks (heartbeat, subscription loops). Good citizen for long-running daemons or CLI tools.

## Complete example: a worker binary

```rust
use resonate::prelude::*;
use tokio::signal;

#[tokio::main]
async fn main() -> Result<()> {
    let resonate = Resonate::new(ResonateConfig {
        url: std::env::var("RESONATE_URL").ok(),
        group: Some("order-workers".into()),
        ..Default::default()
    });

    resonate.register(process_order).unwrap();
    resonate.register(load_order).unwrap();

    // wait for Ctrl+C; the SDK polls in the background
    signal::ctrl_c().await?;

    resonate.stop().await?;
    Ok(())
}

#[resonate::function]
async fn process_order(ctx: &Context, order_id: String) -> Result<String> {
    let order = ctx.run(load_order, order_id).await?;
    Ok(order)
}

#[resonate::function]
async fn load_order(order_id: String) -> Result<String> {
    Ok(format!("order-{}", order_id))
}
```

## Distinct Rust idioms

- **`Result<T>` everywhere** — both Client API methods and durable functions return `Result`. Use `?` for concise propagation.
- **`ResonateConfig { ... , ..Default::default() }`** — Rust's struct-update syntax; always include `..Default::default()` when partially constructing.
- **`Some(...)` / `.ok()` / `std::env::var(...).ok()`** for optional config — the SDK accepts `Option<String>` throughout.
- **`tokio::main`** — durable functions are async; your `main` must be async. `#[tokio::main]` sets up the runtime.
- **`serde_json::json!` macro** for promise data — explicit serialization; matches Rust's strong-typing discipline.
- **`Duration::from_secs(N)` / `from_millis(N)`** for timeouts — not raw numbers. Use `std::time::Duration`.
- **Turbofish syntax on `.rpc::<T>(...)` and `.get::<T>(...)`** — Rust's type inference needs help when the result type isn't constrained by the call site.

## Rust SDK API coverage status

Honest reading of v0.1.0 as of April 2026, verified against `resonate-sdk-rs` source (not just docs):

### In the SDK source + documented in `rust.mdx`
- `Resonate::new/local`, `.register/run/rpc/schedule/get/stop/promises.*`
- `#[resonate::function]` macro + 3 function kinds (Workflow / Leaf with Info / Pure leaf)
- `ctx.run/rpc/sleep` + `.spawn()` double-await pattern
- Context accessors `ctx.id/parent_id/origin_id/func_name/timeout_at`
- Builder options `.timeout(Duration)`, `.target(&str)` (context), `.version/.tags` (ephemeral)

### In the SDK source BUT not in `rust.mdx` (use with eyes open)
- `resonate.with_dependency<T>(value)` — ephemeral-side DI (covered above)
- `ctx.get_dependency::<T>()` — durable-side DI (see `resonate-basic-durable-world-usage-rust`)
- `ctx.promise::<T>()` — Context-side HITL primitive (see `resonate-basic-durable-world-usage-rust` + `resonate-human-in-the-loop-pattern-rust`)
- `ctx.info()` — returns an `Info` struct with extra accessors `branch_id` and `tags`

These are `pub fn` with docstring-level examples in the source; they appear intended for users, just not yet covered by the docs. Until `rust.mdx` catches up, cite the source path (e.g., `resonate/src/context.rs:115`) when reviewers question whether the API exists.

### NOT in the SDK source at v0.1.0
- `ctx.detached` — fire-and-forget; parallelism is via `.spawn()` instead
- `ctx.random.random()` / `ctx.time.time()` — no deterministic time/random helpers
- `ctx.panic()` / `ctx.assert()` — use Rust's own `panic!`/`assert!` + `Result` propagation

Treat the last three as "may land in future versions; verify when v0.2.0+ ships."

## Related skills

- `resonate-basic-durable-world-usage-rust` — Context APIs, `#[resonate::function]` mechanics, `.spawn()` for parallelism, builder options
- `resonate-basic-debugging-rust` — Rust-specific failure modes: v0.1.0 caveat, git-dep install issues, serde errors, `Result<T>` handling
- `durable-execution` + `resonate-philosophy` — foundational concepts; read these first if new to Resonate
- `resonate-basic-ephemeral-world-usage-typescript` + `-python` — sibling SDKs' ephemeral-world surfaces for comparison
