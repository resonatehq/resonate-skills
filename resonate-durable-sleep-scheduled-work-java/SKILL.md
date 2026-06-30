---
name: resonate-durable-sleep-scheduled-work-java
description: Implement durable sleep and scheduled/recurring work in Java with Resonate — ctx.sleep(Duration) inside workflows for timers, countdowns, reminders, and long-horizon delays that survive process restarts, plus the top-level r.schedule(...) cron API (and the r.schedules sub-client) for periodic invocation of a registered function. The Java SDK ships a Schedule API (the Go SDK does not). Use when a workflow must wait for hours or days, or when a function should run on a fixed cron schedule. Verified against example-countdown-java / example-quickstart-java and develop/java.mdx (docs PR #230) at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Durable Sleep + Scheduled Work — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** (virtual threads, a feature generally available in Java 21). Every code block here is compile-verified against `0.1.1` and drawn from `example-countdown-java` / `example-quickstart-java` and `develop/java.mdx` (docs PR #230).

## Overview

Two related capabilities in the Java SDK:

1. **Durable sleep inside a workflow** — `ctx.sleep(Duration)` pauses execution; the worker process can exit and resume later without losing its place. The server holds the timer promise; the cost is one promise record, not process uptime.
2. **Scheduled / recurring work** — `r.schedule(...)` registers a cron schedule that periodically invokes a registered function. The Java SDK includes a top-level Schedule API (plus the lower-level `r.schedules` sub-client); the Go SDK does not expose one yet. This removes the need for an in-workflow sleep loop or an external cron trigger.

Both patterns are durable: Resonate holds the continuation (or the schedule) in its store, not in a long-running thread.

## When to use

- Delays spanning minutes, hours, days, or weeks that must survive crashes (`ctx.sleep`)
- Reminder sequences (7-day trial expiry, multi-stage onboarding drips)
- Countdown workflows that act on each tick
- Periodic jobs on a fixed cron (`r.schedule`) — daily reports, nightly reconciliation
- Anywhere you would reach for `Thread.sleep` but need the work to survive a process restart

## `ctx.sleep` basics

`ctx.sleep` takes a `java.time.Duration` and returns a future with no value to decode — `await()` to suspend until the timer fires.

```java
import io.resonatehq.resonate.Context;
import java.time.Duration;

public static void reminder(Context ctx) {
    ctx.sleep(Duration.ofHours(1)).await(); // no value to decode; suspends until the timer fires
    // ... send the reminder
}
```

**Crash recovery.** With a real Resonate server, killing the worker mid-sleep and restarting with the same promise ID resumes from the outstanding timer rather than restarting the workflow. Local mode (`new Resonate()`) runs state in process memory, so crash recovery requires a real server. **Long-horizon sleeps are cheap** — duration is unbounded; cost is roughly one promise record, and the process need not stay alive.

## Countdown loop (in-workflow recurring pattern)

`example-countdown-java` shows the canonical in-workflow loop: run a side effect via `ctx.run` (durable, checkpointed), then `ctx.sleep` between ticks.

```java
import io.resonatehq.resonate.Context;
import java.time.Duration;

public static int countdown(Context ctx, int start, int stepSeconds) {
    int sent = 0;
    for (int i = start; i > 0; i--) {
        // The side effect lives in a durable step — its result is recorded, so a resume after a
        // crash short-circuits already-sent ticks instead of re-sending them.
        ctx.run(Countdown::tick, i).await();
        sent++;
        // Sleep between ticks (but not after the last one). A durable sleep can suspend for
        // seconds, hours, or days and still resume exactly where it left off.
        if (i > 1) {
            ctx.sleep(Duration.ofSeconds(stepSeconds)).await();
        }
    }
    return sent;
}

public static String tick(Context ctx, int count) {
    System.out.println("  [tick] " + count);
    return "ok";
}
```

A crash during the `ctx.sleep` between ticks resumes mid-loop — completed `ctx.run` ticks short-circuit on replay; the pending sleep re-suspends until its timer fires.

## Multi-stage sleeps

Sequential `ctx.sleep` calls are independent durable checkpoints. A crash mid-sleep resumes from that exact sleep on restart — earlier sleeps that already settled are skipped.

```java
import java.time.Duration;

// Three-phase renewal reminder: 7 days out, 1 day out, renewal day.
public static void renewalReminder(Context ctx, String subId) {
    ctx.sleep(Duration.ofDays(7)).await();
    ctx.run(Reminders::sendRenewalWarning, subId).await();

    ctx.sleep(Duration.ofDays(6)).await(); // 1 day before renewal
    ctx.run(Reminders::sendFinalWarning, subId).await();

    ctx.sleep(Duration.ofDays(1)).await(); // renewal day
    ctx.run(Reminders::chargeRenewal, subId).await();
}
```

## Scheduled work with `r.schedule(...)`

The Java SDK has a top-level cron Schedule API. `r.schedule(...)` creates a schedule that periodically invokes a registered function:

```java
import io.resonatehq.resonate.Resonate;
import java.util.List;
import java.util.Map;

Resonate.ResonateSchedule daily = r.schedule(
        "daily-report",   // schedule id
        "0 0 * * *",      // cron expression (midnight daily)
        "generateReport", // registered function name
        List.of(),        // positional args passed to the function
        Map.of(),         // keyword args
        null,             // promise timeout (null = default)
        1);               // function version

daily.delete(); // remove the schedule when no longer needed
```

The function named in the schedule (`"generateReport"`) must be registered on a worker so the server has somewhere to dispatch each firing. Each tick creates a durable invocation, so the scheduled function gets the same crash-resilience as any other Resonate workflow.

The lower-level `r.schedules` sub-client is available for direct manipulation:

```java
r.schedules.get("daily-report").join();                 // fetch the schedule record
r.schedules.search(Map.of(), 100, null).join();         // list schedules (tags, limit, cursor)
r.schedules.delete("daily-report").join();              // remove it
```

Each `r.schedules` method returns a `CompletableFuture`, so `.join()` (or compose with `thenApply`) to wait.

**`r.schedule` vs an in-workflow `ctx.sleep` loop.** Reach for `r.schedule` when the cadence is a fixed cron and each firing is independent (no state carried across ticks). Reach for an in-workflow `ctx.sleep` loop when the interval is driven by business logic inside the workflow or the run carries state from one tick to the next.

## Distinct Java idioms

- **`ctx.sleep(Duration).await()` returns nothing** — sleep futures carry no value; call `await()` to suspend. (Contrast `ctx.run` / `ctx.rpc`, whose futures decode a result.)
- **`java.time.Duration` for sleeps** — `Duration.ofHours(1)`, `Duration.ofDays(7)`, `Duration.ofSeconds(stepSeconds)`. No raw millisecond integers, no cron strings for `ctx.sleep`.
- **`r.schedule(...)` uses a cron string** — the schedule cadence is a 5-field cron expression; the `ctx.sleep` duration is a `Duration`. Don't confuse the two.
- **Top-level Schedule API** — Java ships `r.schedule(...)` and `r.schedules`; the Go SDK does not have this yet. When porting a scheduled workflow from TypeScript, Python, or Rust, the Java equivalent exists — no in-workflow-loop workaround needed.
- **`ctx.sleep` vs `Thread.sleep`** — `Thread.sleep` inside a durable function is not durable (lost on crash, holds the virtual thread for the full duration). Always use `ctx.sleep` for anything that must survive a restart.

## Avoid

- **`Thread.sleep` inside a durable function** — ephemeral; lost on crash; holds the thread for the full duration. Use `ctx.sleep`.
- **Un-checkpointed side effects before a sleep** — any code before a `ctx.sleep` (or any durable boundary) re-executes on resume. Wrap observable side effects (DB writes, emails, webhooks) in `ctx.run` / `ctx.rpc` so the durable promise records the result and short-circuits replay.
- **Clock-precision assumptions** — `ctx.sleep(Duration.ofHours(24))` firing in 23–25h is within spec (server/worker drift). Don't treat ±1h variance as a bug for long-horizon sleeps.
- **Confusing the cron string with the function name in `r.schedule`** — the argument order is `(id, cron, funcName, args, kwargs, timeout, version)`.
- **Putting function arguments in the `kwargs` (`Map.of()`) slot of `r.schedule`** — Java has no keyword arguments, so the SDK packs only the positional `args` list and leaves the kwargs slot empty (`Durable.java`). Pass all arguments in `List.of(arg1, arg2, ...)`; keep `kwargs` as `Map.of()`.
- **Assuming a scheduled function runs without a registered worker** — `r.schedule` only creates the schedule; a worker must register the named function to execute each firing.

## Related skills

- `resonate-basic-durable-world-usage-java` — `ctx.run`, `ctx.rpc`, `ctx.sleep` fundamentals; the Context the sleep API lives on
- `resonate-basic-ephemeral-world-usage-java` — the `r.schedule(...)` / `r.schedules` sub-client surface
- `resonate-recursive-fan-out-pattern-java` — the worker/client builder split and `CountDownLatch` keep-alive pattern; the registered function behind `r.schedule` runs in such a worker
- `durable-execution` — foundational replay semantics; sleep is a durability checkpoint by design
- `resonate-durable-sleep-scheduled-work-typescript` — sibling with `resonate.schedule()` (cron strings, ms durations)
- `resonate-durable-sleep-scheduled-work-rust` — sibling with `resonate.schedule()`
- **SDK parity note:** TypeScript, Python, Rust, and **Java** expose a top-level `schedule()`; the Go SDK does not yet (it uses in-workflow `ctx.sleep` loops or external cron). When porting *to* Java, use `r.schedule(...)` directly.
