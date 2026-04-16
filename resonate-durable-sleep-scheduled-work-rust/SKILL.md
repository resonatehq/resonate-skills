---
name: resonate-durable-sleep-scheduled-work-rust
description: Implement durable sleep and cron-scheduled work in Rust with Resonate — ctx.sleep(Duration) inside workflows for timers/countdowns/reminders, resonate.schedule() from the ephemeral world for cron-style recurring invocations. Use when a workflow must wait for hours or days, or when a function should run on a fixed schedule. v0.1.0 caveat: API surface may change between Rust SDK releases.
license: Apache-2.0
---

# Resonate Durable Sleep + Scheduled Work — Rust

> **v0.1.0 caveat.** The Rust SDK is in active development. This skill covers two features documented in v0.1.0: `ctx.sleep(Duration)` for in-workflow durable sleep, and `resonate.schedule(name, cron, fn, input)` for cron-registered ephemeral-world scheduling. (Note: the Python SDK does not yet expose a top-level `schedule()`; Rust does.)

## Overview

Two related but distinct capabilities:

1. **Durable sleep inside a workflow** — `ctx.sleep(Duration)` pauses execution; the worker process is free to exit and resume days later without the sleep "losing its place."
2. **Cron-scheduled invocation from the ephemeral world** — `resonate.schedule(...)` registers a function to fire on a cron schedule until explicitly deleted.

Both are Rust-idiomatic — `Duration` for in-workflow time, cron strings for cron schedules. Both are durable; Resonate holds the continuation in its store, not in a long-running process.

## Durable sleep: `ctx.sleep(Duration)`

```rust
use resonate::prelude::*;
use std::time::Duration;

#[resonate::function]
async fn daily_reminder(ctx: &Context, user_id: String) -> Result<()> {
    loop {
        ctx.sleep(Duration::from_secs(24 * 60 * 60)).await?; // 24 hours
        ctx.rpc::<()>("send_reminder", user_id.clone()).await?;
    }
}
```

No upper bound on sleep duration. The worker process can exit after the `.await?` — Resonate's server holds the continuation and re-dispatches when the sleep expires.

### Reminder workflows

```rust
#[resonate::function]
async fn seven_day_renewal_reminder(ctx: &Context, subscription_id: String) -> Result<()> {
    // 7 days before renewal
    ctx.sleep(Duration::from_secs(7 * 24 * 60 * 60)).await?;
    ctx.rpc::<()>("send_renewal_warning", subscription_id.clone()).await?;

    // 1 day before renewal
    ctx.sleep(Duration::from_secs(6 * 24 * 60 * 60)).await?;
    ctx.rpc::<()>("send_final_warning", subscription_id.clone()).await?;

    // renewal day
    ctx.sleep(Duration::from_secs(24 * 60 * 60)).await?;
    ctx.rpc::<()>("charge_renewal", subscription_id).await?;

    Ok(())
}
```

Each `ctx.sleep` is a checkpoint; if the worker crashes during the 7-day sleep, the function resumes from the exact sleep on restart.

### Countdown patterns

```rust
#[resonate::function]
async fn countdown_workflow(ctx: &Context, (name, minutes): (String, u32)) -> Result<String> {
    for remaining in (1..=minutes).rev() {
        ctx.run(send_tick, (name.clone(), remaining)).await?;
        ctx.sleep(Duration::from_secs(60)).await?;
    }
    ctx.run(send_complete, name.clone()).await?;
    Ok(format!("completed {}", name))
}

#[resonate::function]
async fn send_tick((name, remaining): (String, u32)) -> Result<()> {
    // emit a webhook or log; side effect lives inside the leaf so it's checkpointed
    Ok(())
}

#[resonate::function]
async fn send_complete(name: String) -> Result<()> { Ok(()) }
```

The `ctx.run(send_tick, ...)` is durable; the subsequent `ctx.sleep(...)` is durable. A crash during the minute-long pause resumes mid-loop on restart.

### Long-horizon sleeps are cheap

```rust
#[resonate::function]
async fn birthday_greeting(ctx: &Context, (user_id, until): (String, Duration)) -> Result<()> {
    ctx.sleep(until).await?;              // sleep months/years
    ctx.rpc::<()>("send_birthday_email", user_id).await?;
    Ok(())
}
```

Cost is roughly one promise-record for the duration; not tied to process uptime.

## Cron-scheduled invocation: `resonate.schedule(...)`

Scheduling is an ephemeral-world API — you register schedules at process start (or dynamically in response to admin actions), and Resonate dispatches invocations at the configured times.

```rust
use resonate::prelude::*;

#[tokio::main]
async fn main() -> Result<()> {
    let resonate = Resonate::local();

    resonate.register(nightly_reconciliation).unwrap();

    // cron: at 02:00 every day
    let _schedule = resonate
        .schedule(
            "nightly-recon",
            "0 2 * * *",
            "nightly_reconciliation",
            serde_json::json!({ "date": "auto" }),
        )
        .await?;

    // keep the process alive; SDK dispatches invocations on schedule
    tokio::signal::ctrl_c().await?;
    resonate.stop().await?;
    Ok(())
}

#[resonate::function]
async fn nightly_reconciliation(ctx: &Context, input: serde_json::Value) -> Result<()> {
    // called daily at 02:00
    Ok(())
}
```

### Deleting a schedule

```rust
let schedule = resonate.schedule("temp-job", "*/15 * * * *", "cleanup", serde_json::json!({})).await?;

// ... later ...
schedule.delete().await?;
```

### Cron format

Standard 5-field POSIX cron (`minute hour day month dayofweek`). The Resonate server evaluates the schedule and dispatches invocations to the registered function by name.

### Idempotency via stable invocation IDs

The SDK generates an invocation ID per schedule firing that's stable given the schedule + time. If the worker crashes during a scheduled run, the same ID is reused — the partial execution resumes; a duplicate schedule does not create a duplicate invocation.

## Combining sleep and schedule

Common pattern: a schedule fires a "daily digest" workflow; the workflow uses `ctx.sleep` to delay sub-steps:

```rust
// schedule fires daily at 09:00
let _digest_schedule = resonate
    .schedule("morning-digest", "0 9 * * *", "daily_digest", serde_json::json!({}))
    .await?;

#[resonate::function]
async fn daily_digest(ctx: &Context, _input: serde_json::Value) -> Result<()> {
    // aggregate overnight
    ctx.run(aggregate_overnight, ()).await?;

    // small delay before sending to batch up late arrivals
    ctx.sleep(Duration::from_secs(300)).await?;

    ctx.rpc::<()>("send_digests_to_all_users", ()).await?;
    Ok(())
}
```

## Distinct Rust idioms

- **`Duration::from_secs` / `from_millis` / `from_hours`** — always a `std::time::Duration`, never a raw number
- **`tokio::signal::ctrl_c().await`** for graceful-shutdown waits in the ephemeral-world binary
- **`serde_json::json!({ ... })`** for structured schedule input
- **`Schedule::delete().await?`** — the handle from `resonate.schedule(...)` is the delete API
- **Cross-SDK asymmetry:** Python's v0.6.7 SDK has no equivalent `resonate.schedule(...)`; Rust has it at v0.1.0. TypeScript has it. If you're porting a scheduled workflow across SDKs, this asymmetry is load-bearing

## Avoid

- Reaching for `tokio::time::sleep(...)` inside a durable function — that's NOT durable; use `ctx.sleep(Duration::...)`
- Treating `resonate.schedule()` as "fire-and-forget register once, never delete" — leaked schedules invoke forever; keep handles or at least names you can revisit
- Assuming the clock is deterministic — `ctx.sleep` is durable but time drift between worker + server is a real failure mode; if a 24h sleep fires in 23h or 25h, that's within spec

## Related skills

- `resonate-basic-durable-world-usage-rust` — `ctx.sleep` API
- `resonate-basic-ephemeral-world-usage-rust` — `resonate.schedule` API
- `durable-execution` — foundational replay semantics; sleep + schedule are durability-centered by design
- `resonate-durable-sleep-scheduled-work-typescript` — sibling SDK for comparison (Python equivalent does not exist at v0.6.7; see README)
