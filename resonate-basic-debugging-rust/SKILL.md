---
name: resonate-basic-debugging-rust
description: Debug and troubleshoot Resonate applications using the Rust SDK. Use when investigating registration errors, serde serialization failures, tokio runtime mismatches, git-dependency install issues, or the v0.1.0-specific caveats of the early-development Rust SDK.
license: Apache-2.0
---

# Resonate Basic Debugging — Rust

> **v0.1.0 caveat.** This SDK is in active development, not on crates.io, and documented behaviors may shift between point releases. Failure modes listed here are what the documented surface produces; new ones will appear as the SDK grows.

## Overview

Rust's failure modes differ from TS's and Python's. Compile-time type errors catch many bugs the other SDKs only discover at runtime; the remaining runtime failures are usually serde, tokio, or registration-name related. This skill covers the shapes you'll actually see.

For the language-agnostic replay + recovery mental model, read `durable-execution` first.

## Triage flow

1. **Does it compile?** If `cargo build` fails, you're in type-error territory (likely: missing `Result<T>` return, wrong first-parameter type, forgotten `&` on Context/Info, missing `?` on an await)
2. **Does it register?** `resonate.register(fn)` returns a `Result`; unwrap it and look at the error (duplicate name, bad signature)
3. **Does the function kind match expectations?** The SDK infers kind from the first parameter — `&Context` vs `&Info` vs a value type
4. **Is serde happy?** Input/output types need `Serialize` + `Deserialize` derives; stack traces mentioning `serde_json::Error` mean a type doesn't derive correctly
5. **Is the tokio runtime correct?** `#[tokio::main]` or `#[tokio::test(flavor = "multi_thread")]` needed; single-threaded runtime can deadlock on SDK internals
6. **Check server + SDK compatibility.** v0.1.0 git-dep-from-master — your checkout may be behind or ahead of whichever server version you're testing against

## Install / dependency issues

**Symptom:** `error: failed to resolve patches for manifest` or `cannot find crate 'resonate'`.

**Cause:** SDK is not on crates.io; must be a git dependency. Check your `Cargo.toml`:

```toml
[dependencies]
resonate = { git = "https://github.com/resonatehq/resonate-sdk-rs", branch = "master" }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

If you want a pinned revision for reproducibility:

```toml
resonate = { git = "https://github.com/resonatehq/resonate-sdk-rs", rev = "abc1234" }
```

Run `cargo update -p resonate` to pull the latest master commit when upstream changes.

## Registration errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `register` returns `Err(AlreadyRegistered)` | Same function registered twice | Register once per process; check for accidentally-looped registration in tests |
| `register` returns `Err(BadSignature)` | Function doesn't match one of the 3 valid shapes | First param must be `&Context`, `&Info`, or a value type deriving `Deserialize`. Return must be `Result<T>` where T derives `Serialize` |
| Runtime `FunctionNotRegistered` on an RPC call | Caller's name doesn't match registered name | If you used `#[resonate::function(name = "custom")]`, callers must use `"custom"`. Default registered name is the function's Rust identifier |

## Serde errors

**Symptom:** `serde_json::Error { ... }` at runtime when a function result crosses a checkpoint or RPC boundary.

**Cause:** Input or output type doesn't derive `Serialize` / `Deserialize`. Every argument and return value gets serialized by Resonate.

**Fix:**

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Order {
    id: String,
    amount: f64,
}

#[resonate::function]
async fn process(ctx: &Context, order: Order) -> Result<Order> {
    // ...
    Ok(order)
}
```

Common sub-issues:
- `#[serde(rename_all = "snake_case")]` if your JSON payload conventions differ from Rust field names
- `#[serde(skip_serializing_if = "Option::is_none")]` for optional fields
- Enums need `#[serde(tag = "type")]` for internally-tagged representations; pick a representation and stick with it across all workers

## `ctx` vs `info` confusion

**Symptom:** "Cannot find method `run` on `&Info`" at compile time.

**Cause:** Only `&Context` has `run`, `rpc`, `sleep`. `&Info` gives metadata but not execution capabilities.

**Fix:** use `&Context` as the first parameter when the function needs to orchestrate sub-tasks.

| You want | Use |
|---|---|
| Sub-task invocation | `ctx: &Context` |
| Metadata only (read execution ID, parent ID) | `info: &Info` |
| Stateless pure computation | no Context/Info; just value types |

## Missing `?` on `.await`

**Symptom:** compile error about `Result<String>` doesn't implement something, or `future cannot be awaited`.

**Cause:** Every async method in the SDK returns a `Future<Output = Result<T>>`. Awaiting gives you `Result<T>`; you still need `?` to extract `T`:

```rust
// Bad — `result` is Result<String>, not String
let result = ctx.run(leaf, "input".into()).await;

// Good
let result: String = ctx.run(leaf, "input".into()).await?;
```

## `.spawn()` double-await

**Symptom:** "Cannot await on `DurableFuture` directly" or "expected String, found DurableFuture".

**Cause:** `.spawn()` returns a `DurableFuture` after its own `.await` — you need a second `.await?` on the returned future later.

```rust
// the spawn itself awaits; you get a DurableFuture back
let fut = ctx.run(leaf, "input".into()).spawn().await?;

// ... do other work ...

// later: await the DurableFuture to get the actual T
let result: String = fut.await?;
```

This is different from TS/Python's `begin_run` which returns a promise-like handle directly without the double await.

## tokio runtime mismatches

**Symptom:** deadlock at startup, or "Cannot drop runtime in a runtime" panic.

**Cause:** wrong tokio runtime flavor, or nested `Runtime::block_on` inside an already-running runtime.

**Fix:** use `#[tokio::main]` on your entry point with the default multi-threaded runtime. Do NOT wrap Resonate calls in `Handle::block_on` inside a durable function.

```rust
#[tokio::main]
async fn main() -> Result<()> {
    let resonate = Resonate::local();
    resonate.register(my_fn).unwrap();

    let result: String = resonate.run("id", my_fn, "input".into()).await?;
    println!("{}", result);

    resonate.stop().await?;
    Ok(())
}
```

For tests, prefer `#[tokio::test(flavor = "multi_thread")]` over the default single-thread flavor when tests involve multiple workers.

## Non-determinism regressions

Durable functions replay from the last checkpoint. Any non-deterministic code above a checkpoint can cause divergence.

Common Rust-specific footguns:

```rust
// BAD — system time changes between runs
#[resonate::function]
async fn bad(ctx: &Context) -> Result<()> {
    let now = std::time::SystemTime::now();
    if now > SOME_THRESHOLD {
        ctx.run(branch_a, "".into()).await?;
    } else {
        ctx.run(branch_b, "".into()).await?;
    }
    Ok(())
}

// BAD — random values change between runs
use rand::Rng;

#[resonate::function]
async fn bad2(ctx: &Context) -> Result<()> {
    let roll = rand::thread_rng().gen_range(0..100);
    // branch on roll — different on each replay
    Ok(())
}
```

v0.1.0 does NOT expose `ctx.time.time()` / `ctx.random.random()` helpers. Until those land, the safe pattern is:
- Do non-deterministic work inside a leaf (so the value is checkpointed)
- Or derive branches from the invocation's stable ID / input args, not runtime randomness

```rust
#[resonate::function]
async fn good(ctx: &Context, input: String) -> Result<()> {
    // random work inside a checkpointed leaf
    let roll = ctx.run(roll_dice, ()).await?;
    if roll > 50 {
        ctx.run(branch_a, input).await?;
    } else {
        ctx.run(branch_b, input).await?;
    }
    Ok(())
}

#[resonate::function]
async fn roll_dice(_: ()) -> Result<u32> {
    Ok(rand::thread_rng().gen_range(0..100))
}
```

## Minimal repro

```rust
// src/main.rs
use resonate::prelude::*;

#[tokio::main]
async fn main() -> Result<()> {
    let resonate = Resonate::local();
    resonate.register(ping).unwrap();

    let result: String = resonate.run("ping:alice", ping, "Alice".into()).await?;
    println!("{}", result);

    resonate.stop().await?;
    Ok(())
}

#[resonate::function]
async fn ping(name: String) -> Result<String> {
    Ok(format!("pong {}", name))
}
```

If this fails, the problem is infrastructure (Cargo deps, tokio runtime, SDK version). If it succeeds but your real code fails, diff your function signatures against this template.

## Server compatibility

v0.1.0 of the Rust SDK's server-protocol compatibility is in flux. Before reporting a bug, verify:

1. Your SDK commit — `cargo metadata | grep resonate` for the git commit hash
2. Your server version — `resonate --version` on the server binary
3. The SDK's compatibility notes — check the SDK's CHANGELOG or README on GitHub

Expect intermittent breaks between master-branch SDK commits and stable server releases until the SDK ships 1.0.

## CLI one-liners

```shell
resonate dev                                  # local dev server
resonate tree <invocation-id>                 # call-graph
resonate promises get <id>                    # single promise state
resonate promises search 'order:*'            # prefix search
resonate promises resolve <id> --data '{}'    # settle a pending promise
```

The CLI is SDK-agnostic; same commands work for TS, Python, Rust worker ecosystems.

## What is NOT yet in the Rust SDK

(Cross-reference with `resonate-basic-durable-world-usage-rust`'s "What is NOT documented" section — the same gaps.)

- `ctx.promise()` Context-level HITL API
- `ctx.detached` fire-and-forget
- `ctx.get_dependency` / `ctx.set_dependency`
- `ctx.random.random()` / `ctx.time.time()` deterministic helpers
- `ctx.panic()` / `ctx.assert()`

Don't chase these as "missing docs" — they're genuine SDK-surface gaps. Each may land in a future version; check `docs/develop/rust.mdx` when a new release ships.

## Related skills

- `resonate-basic-ephemeral-world-usage-rust` — Client APIs at the process-entry layer
- `resonate-basic-durable-world-usage-rust` — Context APIs inside durable functions
- `durable-execution` + `resonate-philosophy` — foundational; many debug sessions end up being about patterns warned against here
- `resonate-basic-debugging-typescript` + `-python` — sibling SDKs for comparison
