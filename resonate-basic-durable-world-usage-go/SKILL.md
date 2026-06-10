---
name: resonate-basic-durable-world-usage-go
description: Core patterns for writing Resonate durable functions in Go — function kinds (registered vs leaf), Context APIs (ctx.Run, ctx.RPC, ctx.Sleep, ctx.Promise, ctx.Detached, Future.Await), option structs, context accessors, retries (bounded 3-attempt default), fan-out/fan-in, and the replay model. Pre-release: no semver tag yet; APIs verified against develop/go.mdx and resonatehq-examples/*-go repos at SDK commit 22076134651f.
license: Apache-2.0
---

# Resonate Basic Durable World Usage — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx` and the `resonatehq-examples/*-go` repos at SDK commit `22076134651f`.

## Overview

Durable functions in Go are ordinary Go functions — no attribute macro, no wrapper type. Every Context method returns a `*resonate.Future` immediately; `f.Await(&out)` reads the result and, if the promise is still pending, suspends the workflow until it settles. Suspension is durable: the runtime persists the continuation on the Resonate server, so any worker (including this one after restart) can resume it when the promise resolves.

This skill covers the Context API surface used **inside** durable functions. The ephemeral-world counterpart (registration, top-level invocation, `resonate.New`) lives in `resonate-basic-ephemeral-world-usage-go`.

## When to use

Use this skill whenever you are writing the body of a Go function that will be registered with `resonate.Register` or passed as a leaf to `ctx.Run`. The moment you touch `ctx.Run`, `ctx.RPC`, `ctx.Sleep`, `ctx.Promise`, or `ctx.Detached`, this skill applies.

## Function kinds

### Registered functions

Any function passed to `resonate.Register` must carry the canonical two-parameter signature. `Register` is generic over `A` (args) and `R` (result), so both are required. Use `struct{}` for no-input or no-result:

```go
type OrderArgs struct {
    OrderID string `json:"order_id"`
}

func processOrder(ctx *resonate.Context, args OrderArgs) (string, error) {
    // orchestration logic here
    return "done", nil
}

// No meaningful result:
func cleanup(ctx *resonate.Context, _ struct{}) (struct{}, error) {
    return struct{}{}, nil
}
```

### Leaf functions (ctx.Run targets)

A function you only ever hand to `ctx.Run` — never registered by name — may use a leaner shape. `ctx.Run` resolves the signature by reflection and accepts any of:

| Signature | When to use |
|---|---|
| `func(*resonate.Context, A) (R, error)` | Needs the context and args |
| `func(*resonate.Context) (R, error)` | Needs the context, no args |
| `func(A) (R, error)` | Stateless leaf with args |
| `func() (R, error)` | Stateless leaf, no args |

```go
// Stateless leaf — no Context needed.
func formatGreeting(name string) (string, error) {
    return fmt.Sprintf("hello, %s!", name), nil
}
```

**Footgun:** `ctx.Run` takes `fn any` — leaf signatures are NOT compiler-checked. A bad shape only fails at execution time. If you see a runtime reflection panic on a `ctx.Run` call, check the function signature first.

## `ctx.Run` — same-process invocation

`ctx.Run` dispatches a function in the current process and returns a `*Future` immediately. Awaiting right after is sequential; dispatching multiple futures before awaiting any is parallel (see Fan-out below).

```go
func greetWorkflow(ctx *resonate.Context, args GreetArgs) (string, error) {
    f, err := ctx.Run(formatGreeting, args.Name)
    if err != nil {
        return "", err
    }
    var msg string
    if err := f.Await(&msg); err != nil {
        return "", err
    }
    return msg, nil
}
```

**`ctx.Run` functions must return promptly.** The runtime joins every `ctx.Run`-spawned goroutine before the workflow can suspend. A function that blocks indefinitely holds the task lease open until it expires. Long-running or externally-blocking work belongs in `ctx.RPC` or `ctx.Promise`, not `ctx.Run`.

## Fan-out / fan-in

Dispatch ALL children first (collect futures), then Await each. Awaiting inside the dispatch loop serializes them — that is never what you want for a fan-out.

```go
// From example-fan-out-fan-in-go: dispatch one RPC per channel, then await all.
func fanout(ctx *resonate.Context, args FanoutArgs) (FanoutResult, error) {
    futures := make([]*resonate.Future, 0, len(args.Channels))
    for _, ch := range args.Channels {
        f, err := ctx.RPC("send", SendArgs{Channel: ch, Message: args.Message})
        if err != nil {
            return FanoutResult{}, err
        }
        futures = append(futures, f)
    }

    out := FanoutResult{Delivered: make([]Delivery, 0, len(futures))}
    for i, f := range futures {
        var d Delivery
        if err := f.Await(&d); err != nil {
            out.Delivered = append(out.Delivered, Delivery{
                Channel: args.Channels[i], OK: false, Reason: err.Error(),
            })
            continue
        }
        out.Delivered = append(out.Delivered, d)
    }
    return out, nil
}
```

The same pattern applies to `ctx.Run` fan-outs: collect all futures, then range over them and Await.

## `ctx.RPC` — remote-process invocation

`ctx.RPC` dispatches a registered function by name to a remote process. `Await` suspends the workflow until the remote function returns. All arguments must be JSON-serializable (they cross a process boundary).

```go
func orchestrator(ctx *resonate.Context, batchID string) (string, error) {
    f, err := ctx.RPC("worker-fn", batchID, resonate.RPCOpts{
        Target: "poll://any@processing-workers",
    })
    if err != nil {
        return "", err
    }
    var result string
    if err := f.Await(&result); err != nil {
        return "", err
    }
    return result, nil
}
```

The fan-out pattern applies equally to `ctx.RPC` — dispatch all calls in a loop, collect futures, then Await each.

## `ctx.Sleep` — durable sleep

`ctx.Sleep` takes a `time.Duration` (never raw ms/sec integers). There is no upper bound on the duration; sleeps survive process restarts. Pass `nil` to `Await` — there is no value to decode.

```go
func reminder(ctx *resonate.Context, userID string) (struct{}, error) {
    f, err := ctx.Sleep(24 * time.Hour)
    if err != nil {
        return struct{}{}, err
    }
    if err := f.Await(nil); err != nil {
        return struct{}{}, err
    }
    // send the reminder
    return struct{}{}, nil
}
```

## `ctx.Promise` — latent durable promise

`ctx.Promise` creates a durable promise that no registered function backs — it only settles when an external actor (webhook, human, CLI) calls promise-settle with its ID. Options are optional: `ctx.Promise()` with no arguments is valid.

```go
func approval(ctx *resonate.Context, _ struct{}) (string, error) {
    f, err := ctx.Promise(resonate.PromiseOpts{Timeout: 24 * time.Hour})
    if err != nil {
        return "", err
    }
    // f.ID() returns the promise ID — hand it to whoever resolves it.
    // e.g. resonate promise resolve <id> --data '"approved"'

    var decision string
    if err := f.Await(&decision); err != nil {
        return "", err
    }
    return decision, nil
}
```

For full external-resolution mechanics (encoding, `r.Sender().PromiseSettle`, webhook handlers, the base64 encoding requirement, issue #28), see `resonate-human-in-the-loop-pattern-go`.

## `ctx.Detached` — fire-and-forget remote dispatch

`ctx.Detached` dispatches a remote function without suspending the parent. It returns only the new promise's `id` (a string); the parent workflow never awaits it. Use this for side-channel work (receipts, analytics events, audit logs) that must not block the critical path.

```go
func placeOrder(ctx *resonate.Context, orderID string) (struct{}, error) {
    id, err := ctx.Detached("send-receipt", ReceiptArgs{Order: orderID})
    if err != nil {
        return struct{}{}, err
    }
    _ = id // the detached execution's promise ID; parent does not await it
    return struct{}{}, nil
}
```

**Detached IDs are not cross-SDK-portable.** The Go SDK derives them with FNV-1a; the Rust SDK uses a different scheme. Do not rely on the same workflow body producing the same Detached IDs across SDKs.

## Option structs

Go uses trailing option structs (not chained builders). All option arguments are optional — omit the struct entirely when defaults suffice.

| Method | Options struct | Fields |
|---|---|---|
| `ctx.Run` | `resonate.RunOpts` | `Timeout time.Duration`, `RetryPolicy resonate.RetryPolicy` |
| `ctx.RPC` | `resonate.RPCOpts` | `Timeout time.Duration`, `Target string` |
| `ctx.Promise` | `resonate.PromiseOpts` | `Timeout time.Duration`, `Data any` |
| `ctx.Detached` | `resonate.DetachedOpts` | `Timeout time.Duration`, `Target string` |

```go
f, err := ctx.Run(chargeCard, args, resonate.RunOpts{
    Timeout:     30 * time.Second,
    RetryPolicy: resonate.LinearRetry{MaxAttempts: 5, Base: time.Second},
})
```

## Context accessors

Inside a workflow, the Context provides read-only metadata about the current execution:

```go
func myWorkflow(ctx *resonate.Context, _ struct{}) (struct{}, error) {
    _ = ctx.ID()        // this execution's promise ID
    _ = ctx.ParentID()  // parent promise ID (empty for root)
    _ = ctx.OriginID()  // root workflow ID — stable across the whole call tree
    _ = ctx.FuncName()  // registered function name
    _ = ctx.TimeoutAt() // promise deadline, epoch milliseconds (int64)
    return struct{}{}, nil
}
```

`ctx.OriginID()` is particularly useful for distributed tracing: every node in the call graph shares the same origin ID.

## Retries

When a function returns an error, Resonate re-executes it according to a retry policy. Nil applies `DefaultRetryPolicy`.

| Policy | Behavior |
|---|---|
| `resonate.ConstantRetry{MaxAttempts, Delay}` | Fixed delay between attempts |
| `resonate.LinearRetry{MaxAttempts, Base}` | Attempt N waits `Base * N` |
| `resonate.ExponentialRetry{MaxAttempts, Base, Max, Jitter}` | Doubling delay capped at Max, with optional jitter |
| `resonate.NoRetry` | Single attempt, no retries |

**`DefaultRetryPolicy` is `ExponentialRetry{MaxAttempts: 3, Base: 100ms, Max: 30s, Jitter: true}`** — bounded to 3 attempts. This is asymmetric with the TypeScript and Python defaults, which are effectively unbounded. If your workflow depends on many retries, set the policy explicitly.

```go
f, err := ctx.Run(chargeCard, args, resonate.RunOpts{
    RetryPolicy: resonate.ExponentialRetry{
        MaxAttempts: 5,
        Base:        100 * time.Millisecond,
        Max:         30 * time.Second,
        Jitter:      true,
    },
})
```

**Non-retryable errors.** To mark a specific error terminal so the retry loop stops immediately regardless of policy, wrap it with `resonate.NewNonRetryable`. The wrapped error remains reachable via `errors.Is` / `errors.As` / `errors.Unwrap`:

```go
func validate(input string) (string, error) {
    if input == "" {
        return "", resonate.NewNonRetryable(errors.New("input is required"))
    }
    return input, nil
}
```

## The replay model

This is the most important section. Read it before you write your first workflow.

Whenever a workflow suspends and resumes — after `ctx.Sleep`, an `ctx.RPC` await, or a pending `ctx.Promise` await — **the entire workflow body re-runs from the top.** Resonate short-circuits already-settled child promises (their stored results are replayed without re-executing the function), but any code that runs *before* reaching a settled durable boundary executes again on every resume.

**Consequence: side effects before a durable boundary run more than once.**

```go
// insertOrder is a leaf; db is whatever client you close over or inject.
// WRONG — the fmt.Println and the DB write re-run on every resume.
func badWorkflow(ctx *resonate.Context, id string) (string, error) {
    fmt.Println("starting order") // runs on every replay
    db.Insert(id)                 // inserts a duplicate row on replay!

    f, err := ctx.Sleep(1 * time.Hour)
    if err != nil {
        return "", err
    }
    if err := f.Await(nil); err != nil {
        return "", err
    }
    return "done", nil
}

// CORRECT — the side effect is wrapped in ctx.Run, so it is checkpointed:
// on resume the settled promise short-circuits and insertOrder does not re-run.
func goodWorkflow(ctx *resonate.Context, id string) (string, error) {
    f, err := ctx.Run(insertOrder, id)
    if err != nil {
        return "", err
    }
    if err := f.Await(nil); err != nil { // wait for the checkpointed write
        return "", err
    }

    s, err := ctx.Sleep(1 * time.Hour)
    if err != nil {
        return "", err
    }
    if err := s.Await(nil); err != nil {
        return "", err
    }
    return "done", nil
}
```

Rules of thumb:
- DB writes, payment calls, emails, and any observable I/O must live inside a `ctx.Run` or `ctx.RPC` call so the result is checkpointed.
- Pure computation (parsing, formatting, struct construction) before a durable boundary is fine — it is idempotent.
- `ctx.Run` functions themselves replay-safe by definition: Resonate returns the stored result on replay rather than re-executing the function body.
- The Resonate instance (`r`) cannot be used inside durable functions. The Context (`ctx`) cannot be used in the Ephemeral World.

## Distinct Go idioms

- **Option structs, not chained builders.** The Rust SDK chains `.timeout(d).target(s)` etc. Go passes a trailing struct. The TypeScript SDK uses `ctx.options({...})`. Go uses `resonate.RunOpts{...}` as the last positional argument.
- **`time.Duration` everywhere.** Never pass raw millisecond or second integers for time values. `30 * time.Second`, not `30000`.
- **`struct{}` for no-input / no-result.** `Register` is generic; both type params are required. `struct{}` is the idiomatic empty stand-in.
- **`f.Await(nil)` after Sleep.** Sleep returns a future with no value. Pass `nil` to Await — passing a pointer to a non-nil variable will cause a decode error.
- **`ctx.Promise()` with no args is valid.** `PromiseOpts` is optional; omit it entirely for a no-timeout latent promise.
- **`localnet` requires `NoopHeartbeat{}`** — the in-process transport has no HTTP endpoint for the default `AsyncHeartbeat` to reach. Always pair `localnet.NewLocal(...)` with `Heartbeat: resonate.NoopHeartbeat{}`.

## Avoid

- Calling `r.Stop()` on a long-running worker — it tears down the heartbeat and subscription loops, silently stopping task processing.
- Using `time.Sleep` or `time.After` for waits that should survive restarts — use `ctx.Sleep`.
- Placing observable side effects outside `ctx.Run` / `ctx.RPC` — they will re-execute on every replay.
- Calling the Resonate instance (`r`) from inside a durable function — use Context APIs only.
- Passing non-JSON-serializable values to `ctx.RPC` or `ctx.Detached` — they cross a process boundary and must encode.
- Awaiting inside a fan-out dispatch loop — this serializes what should be concurrent.

## Related skills

- `resonate-basic-ephemeral-world-usage-go` — `resonate.New`, `resonate.Register`, `RegisteredFunc.Run`, `Resonate.RPC`, `Resonate.Get`, `localnet` setup
- `resonate-recursive-fan-out-pattern-go` — dynamic-depth tree fan-outs, child-workflow dispatch
- `resonate-human-in-the-loop-pattern-go` — full `ctx.Promise` resolution mechanics, `r.Sender().PromiseSettle`, base64 encoding, webhook handlers
- `resonate-durable-sleep-scheduled-work-go` — recurring work loops, `ctx.Sleep` with server-backed crash recovery
- `durable-execution` — foundational concepts: checkpointing, replay, the Ephemeral/Durable World split
- `resonate-defaults` — full defaults table across all SDKs with source citations
- `resonate-basic-durable-world-usage-typescript` — TypeScript counterpart (generator functions, `yield*`, `beginRun` for fan-out, unbounded default retries)
- `resonate-basic-durable-world-usage-rust` — Rust counterpart (`#[resonate::function]`, `.spawn()` for parallelism, `ctx.get_dependency`)
