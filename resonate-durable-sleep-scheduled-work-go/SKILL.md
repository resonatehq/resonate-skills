---
name: resonate-durable-sleep-scheduled-work-go
description: Implement durable sleep and recurring-work patterns in Go with Resonate — ctx.Sleep(time.Duration) inside workflows for timers, countdowns, reminders, and long-horizon delays that survive process restarts; Resonate.Schedules() (shipped in 0.1.0) for direct cron-fired promises, plus in-workflow ctx.Sleep loops or external cron → RPC for recurring registered-function dispatch. Use when a workflow must wait for hours or days, or when a function should run on a fixed schedule. Verified against the resonate-sdk-go 0.1.0 tag.
license: Apache-2.0
---

# Resonate Durable Sleep + Scheduled Work — Go

> **Version note.** The Go SDK's first tagged release is [`0.1.0`](https://github.com/resonatehq/resonate-sdk-go/releases/tag/0.1.0) (`go get github.com/resonatehq/resonate-sdk-go@0.1.0` — the tag has no `v` prefix, so `@latest` does not resolve to it). `0.1.0` shipped a top-level `Schedules()` sub-client for direct cron-fired promises. This skill covers durable sleep (`ctx.Sleep`), the `Schedules()` sub-client, and the patterns for recurring *registered-function* dispatch (in-workflow `ctx.Sleep` loops; external cron → `RPC`). Every code block is verified against the `0.1.0` tag source, `example-durable-sleep-go`, and `example-countdown-go`.

## Overview

Three related capabilities in the Go SDK:

1. **Durable sleep inside a workflow** — `ctx.Sleep(d time.Duration)` pauses execution; the worker process can exit and resume later without losing its place. The server holds the timer promise; cost is one promise record, not process uptime.
2. **Direct cron schedules** — `r.Schedules().Create(...)` (shipped in `0.1.0`) creates a promise from a cron expression, on a recurring basis, without any workflow involved. This is the Go equivalent of Python's `resonate.schedules.create(...)` sub-client.
3. **Recurring registered-function dispatch** — Go's `0.1.0` tag does not yet have the top-level `resonate.schedule(id, cron, fn, args)` convenience wrapper that Python/TypeScript/Rust expose, which dispatches a *registered function* on the cron. Until it lands, use an in-workflow `ctx.Sleep` loop (bounded or long-running periodic task owned by one workflow), an external cron that fires `r.RPC(...)` on each tick, or `Schedules().Create` with the dispatch tags set by hand (shown below).

All three are durable: Resonate holds the continuation (or the schedule) in its store, not in a long-running goroutine.

## When to use

- Delays spanning minutes, hours, days, or weeks that must survive crashes
- Reminder sequences (7-day trial expiry, multi-stage onboarding drips)
- Countdown workflows that post a notification per tick
- Periodic jobs where an in-workflow loop is acceptable, or where an external scheduler already exists
- Any place you would reach for `time.Sleep` but need the work to survive a process restart

## `ctx.Sleep` basics

`ctx.Sleep` takes a `time.Duration` and returns a `*resonate.Future`. Call `f.Await(nil)` to suspend until the timer fires — there is no value to decode.

```go
// From example-durable-sleep-go/main.go
func sleepingWorkflow(ctx *resonate.Context, args SleepArgs) (string, error) {
    d := time.Duration(args.Secs) * time.Second

    f, err := ctx.Sleep(d)
    if err != nil {
        return "", fmt.Errorf("ctx.Sleep: %w", err)
    }
    // Await(nil) — no value to decode; suspends until the timer promise resolves.
    if err := f.Await(nil); err != nil {
        return "", fmt.Errorf("sleep await: %w", err)
    }

    return fmt.Sprintf("slept for %d second(s)", args.Secs), nil
}
```

**Crash recovery.** With a real Resonate server (`-url=http://localhost:8001`), killing the worker mid-sleep and restarting with the same promise ID resumes from the outstanding timer rather than restarting the workflow. The `localnet` transport runs state in process memory, so crash recovery requires a real server.

**Duration encoding tip.** `time.Duration` is `int64` nanoseconds and round-trips through JSON as a bare number, which is opaque in promise payloads. Store durations as explicit seconds fields (as `SleepArgs.Secs` does) to keep stored promise data readable.

## Reminder / multi-stage sleeps

Sequential `ctx.Sleep` calls are independent durable checkpoints. A crash mid-sleep resumes from that exact sleep on restart — earlier sleeps that already settled are skipped.

```go
// Three-phase renewal reminder: 7 days out, 1 day out, renewal day.
func renewalReminder(ctx *resonate.Context, subID string) (struct{}, error) {
    // 7 days before renewal
    if f, err := ctx.Sleep(7 * 24 * time.Hour); err != nil {
        return struct{}{}, err
    } else if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }
    if f, err := ctx.RPC("send-renewal-warning", subID); err != nil {
        return struct{}{}, err
    } else if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }

    // 6 more days (1 day before renewal)
    if f, err := ctx.Sleep(6 * 24 * time.Hour); err != nil {
        return struct{}{}, err
    } else if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }
    if f, err := ctx.RPC("send-final-warning", subID); err != nil {
        return struct{}{}, err
    } else if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }

    // 1 more day — renewal day
    if f, err := ctx.Sleep(24 * time.Hour); err != nil {
        return struct{}{}, err
    } else if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }
    f, err := ctx.RPC("charge-renewal", subID)
    if err != nil {
        return struct{}{}, err
    }
    return struct{}{}, f.Await(nil)
}
```

## Countdown loop (in-workflow recurring pattern)

The `example-countdown-go` canonical example shows the real in-workflow loop pattern: dispatch the side effect via `ctx.RPC` (durable, checkpointed), then `ctx.Sleep` between ticks.

```go
// From example-countdown-go/main.go — adapted for clarity.
func countdown(ctx *resonate.Context, args CountdownArgs) (CountdownResult, error) {
    sent := 0
    for i := args.Start; i > 0; i-- {
        // Side effect lives inside ctx.RPC so it's checkpointed — won't double-fire on resume.
        f, err := ctx.RPC("notify", NotifyArgs{Count: i, URL: args.NotifyURL})
        if err != nil {
            return CountdownResult{}, err
        }
        var r NotifyResult
        if err := f.Await(&r); err != nil {
            return CountdownResult{}, fmt.Errorf("notify %d: %w", i, err)
        }
        sent++

        if i > 1 {
            s, err := ctx.Sleep(time.Duration(args.StepSeconds) * time.Second)
            if err != nil {
                return CountdownResult{}, err
            }
            if err := s.Await(nil); err != nil {
                return CountdownResult{}, fmt.Errorf("sleep before %d: %w", i-1, err)
            }
        }
    }
    return CountdownResult{Sent: sent}, nil
}
```

A crash during the `ctx.Sleep` between ticks resumes mid-loop — completed `ctx.RPC` ticks short-circuit on replay; the pending sleep re-suspends until its timer fires.

## Long-horizon sleeps are cheap

Sleep duration is unbounded. Cost is roughly one promise record; the process does not need to stay alive.

```go
// Sleep for months — the process can exit and the timer holds in the server.
func birthdayGreeting(ctx *resonate.Context, args BirthdayArgs) (struct{}, error) {
    f, err := ctx.Sleep(args.UntilBirthday) // weeks or months ahead
    if err != nil {
        return struct{}{}, err
    }
    if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }
    gf, err := ctx.RPC("send-birthday-email", args.UserID)
    if err != nil {
        return struct{}{}, err
    }
    return struct{}{}, gf.Await(nil)
}
```

## `Resonate.Schedules()` — direct cron-fired promises

`r.Schedules()` (a top-level Client API, called from the ephemeral world — not a Context method) creates a schedule that fires a fresh promise on every cron tick, from a promise ID template:

```go
ctx := context.Background()

// Fires a fresh promise every night at 00:00, from the report-{{.timestamp}} template.
s, err := r.Schedules().Create(ctx, "nightly-report", "0 0 * * *", "report-{{.timestamp}}", time.Hour,
    resonate.ScheduleCreateOptions{PromiseParam: ReportArgs{Region: "us"}})
if err != nil {
    log.Fatalf("Schedules().Create: %v", err)
}

s, err = r.Schedules().Get(ctx, "nightly-report")
err = r.Schedules().Delete(ctx, "nightly-report")
```

This creates the promise directly — nobody is dispatched to run a registered function unless you arrange it. `Schedules().Create` does not exist to wrap Rust or TypeScript's `resonate.schedule(id, cron, fn, args)` examples 1:1 — translating those verbatim (expecting a function to run automatically) will compile but the promise it creates will just sit there for a worker to notice, not auto-dispatch.

**To dispatch a registered function on the cron** (matching Rust/TypeScript's `resonate.schedule(...)` behavior), set the dispatch tags and param shape by hand — the same shape `RegisteredFunc.Run` and `Resonate.RPC` build internally:

```go
s, err := r.Schedules().Create(ctx, "nightly-reconciliation", "0 2 * * *",
    "recon-{{.timestamp}}", 24*time.Hour,
    resonate.ScheduleCreateOptions{
        PromiseParam: map[string]any{
            "func": "reconcile", // must match the name passed to resonate.Register
            "args": ReconArgs{Date: "{{.timestamp}}"},
        },
        PromiseTags: map[string]string{
            "resonate:target": "poll://any@default", // routes to a worker group, like RunOptions.Target
        },
    })
```

A worker polling that target picks up each fired promise as an ordinary task, same as an `RPC` dispatch. This is a manual assembly of what a future top-level `resonate.Schedule(...)` wrapper would do for you — verify the exact param/tag shape against `buildRootPromiseCreateReq` in `resonate.go` before shipping, since it is not a documented public contract.

## Recurring work without the function-dispatch convenience wrapper

Two further substitutes for recurring *registered-function* dispatch, useful when you'd rather not hand-assemble the dispatch tags above:

### Pattern 1 — In-workflow `ctx.Sleep` loop

A workflow that loops indefinitely (or for a bounded count) and sleeps between iterations is a self-contained recurring job. The loop is fully durable — a crash mid-sleep resumes at the current iteration.

```go
// Periodic cleanup job: runs every intervalDays days, indefinitely.
func periodicCleanup(ctx *resonate.Context, args CleanupArgs) (struct{}, error) {
    for {
        // Side effect checkpointed in ctx.Run — won't double-fire on replay.
        f, err := ctx.Run(runCleanup, args)
        if err != nil {
            return struct{}{}, err
        }
        if err := f.Await(nil); err != nil {
            return struct{}{}, err
        }

        // Durable sleep until next run.
        s, err := ctx.Sleep(time.Duration(args.IntervalDays) * 24 * time.Hour)
        if err != nil {
            return struct{}{}, err
        }
        if err := s.Await(nil); err != nil {
            return struct{}{}, err
        }
    }
}
```

Start once with a stable promise ID:

```go
// Invoke from the ephemeral world — deduplicated on the ID, so safe to re-run on deploy.
h, err := cleanupFn.Run(ctx, "periodic-cleanup-prod", CleanupArgs{IntervalDays: 7})
```

**When to use:** bounded or long-running periodic task owned by exactly one workflow; interval driven by business logic inside the workflow.

### Pattern 2 — External cron → `r.RPC`

Keep the schedule outside Resonate (OS cron, Cloud Scheduler, GitHub Actions, Kubernetes CronJob). Each tick calls `r.RPC` (or the `resonate invoke` CLI) to create a durable invocation.

```go
// cron-trigger/main.go — runs on every cron tick; idempotent on stable ID.
func main() {
    r, err := resonate.New(resonate.Config{URL: os.Getenv("RESONATE_URL")})
    if err != nil {
        log.Fatalf("resonate.New: %v", err)
    }
    defer func() { _ = r.Stop() }()

    // Stable ID for today's run — deduplicates if the cron fires twice.
    today := time.Now().UTC().Format("2006-01-02")
    id := fmt.Sprintf("nightly-recon/%s", today)

    ctx := context.Background()
    h, err := r.RPC(ctx, id, "nightly-reconciliation", ReconArgs{Date: today})
    if err != nil {
        log.Fatalf("RPC: %v", err)
    }
    var result ReconResult
    if err := h.Result(ctx, &result); err != nil {
        log.Fatalf("Result: %v", err)
    }
    log.Printf("reconciliation done: %+v", result)
}
```

**When to use:** the schedule already lives in an external system; per-firing invocations are independent (no loop state carried across ticks); or the interval must be changed without redeploying a long-running workflow.

**CLI equivalent (no code trigger needed):**

```shell
resonate invoke nightly-reconciliation --id "nightly-recon/$(date +%F)" --data '{"date":"2026-06-10"}'
```

## Distinct Go idioms

- **`f.Await(nil)`** — sleep futures carry no value; pass `nil` to `Await` (unlike `ctx.Run`/`ctx.RPC` futures where you decode into a pointer).
- **`time.Duration`** — all sleep durations are native Go durations (`24*time.Hour`, `time.Minute`, etc.); no raw millisecond integers, no cron strings.
- **`ctx.Sleep` vs `time.Sleep`** — `time.Sleep` inside a durable function is not durable (lost on crash, blocks the goroutine for its full duration). Always use `ctx.Sleep` for anything you need to survive a restart.
- **Options struct last** — `ctx.Sleep` takes only a `time.Duration`; no options struct. `ctx.Run`/`ctx.RPC` accept an optional trailing `RunOpts`/`RPCOpts` struct.
- **Localnet requires `NoopHeartbeat{}`** — `localnet.NewLocal(...)` has no HTTP endpoint; the default `AsyncHeartbeat` will error. Always pair localnet with `Heartbeat: resonate.NoopHeartbeat{}`.

## Avoid

- **`time.Sleep` inside a durable function** — ephemeral; lost on crash; holds the goroutine for the full duration. Use `ctx.Sleep`.
- **Un-checkpointed side effects before a sleep** — any code that runs before a `ctx.Sleep` (or any other durable boundary) re-executes on resume. Wrap observable side effects (DB writes, emails, webhooks) in `ctx.Run`/`ctx.RPC` so the durable promise records the result and short-circuits replay.
- **Assuming `Schedules().Create` auto-dispatches a registered function** — it creates the cron-fired promise only; translating Rust or TypeScript `resonate.schedule(...)` code verbatim compiles but nothing consumes the fired promise unless you set the `resonate:target` tag and `{func, args}` param shape yourself, or fall back to one of the patterns above.
- **Clock precision assumptions** — `ctx.Sleep(24*time.Hour)` firing in 23–25h is within spec (server/worker drift). Don't treat variance of ±1h as a bug for long-horizon sleeps.
- **Raw `time.Duration` nanoseconds in promise payloads** — store durations as explicit integer fields (seconds, minutes) to keep promise data human-readable across crashes and inspections.

## Related skills

- `resonate-basic-durable-world-usage-go` — `ctx.Run`, `ctx.RPC`, `ctx.Promise` fundamentals; the same Context the `ctx.Sleep` API lives on
- `resonate-basic-ephemeral-world-usage-go` — `Resonate.Promises()` / `Resonate.Schedules()` sub-client reference (Client APIs, not Context APIs)
- `durable-execution` — foundational replay semantics; sleep is a durability checkpoint by design
- `resonate-durable-sleep-scheduled-work-typescript` — TypeScript sibling; has the top-level `resonate.schedule()` function-dispatch wrapper (cron strings, ms durations)
- `resonate-durable-sleep-scheduled-work-rust` — Rust sibling; has the top-level `resonate.schedule()` function-dispatch wrapper (cron strings, `std::time::Duration`)
- **SDK gap note:** Go's `0.1.0` `Schedules()` sub-client covers direct cron-fired promises (like Python's `schedules.create(...)` sub-client), but Go alone still lacks the higher-level top-level `schedule(id, cron, fn, args)` convenience wrapper that dispatches a registered function — Python (0.7.4), TypeScript, and Rust all have one. If porting a scheduled workflow from Python/Rust/TypeScript to Go, either hand-assemble the dispatch tags (see above) or use one of the two patterns in the "Recurring work" section.
