---
name: resonate-recursive-fan-out-pattern-go
description: Implement recursive fan-out / parallel workflow execution in Go with Resonate — dispatch all children via ctx.RPC or ctx.Run, collect futures in a slice, then await each. Use when processing batches, trees, or graphs where each child is independently durable and optionally recurses. Pre-release caveat: the Go SDK has no semver tag yet; pin a commit for stability.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx` and the `example-fan-out-fan-in-go` / `example-recursive-factorial-go` repos at SDK commit `22076134651f`.

## Overview

Recursive fan-out dispatches multiple child invocations in parallel, awaits them individually, and optionally recurses deeper. The Go expression is a two-loop pattern: a dispatch loop builds a `[]*resonate.Future` slice, then a separate await loop reads each result. Mixing dispatch and await serializes the children — that is the single most common mistake.

For the language-agnostic mental model (promise deduplication, load-balancing, detached vs. result-gathering), see `resonate-recursive-fan-out-pattern-typescript`.

## When to use

- Batch processing where items are independent and each needs durability
- Map-reduce shaped workflows (fan out N workers, aggregate results)
- Recursive tree / graph traversal with dynamic depth
- Any case where a crash mid-fan-out must resume, not restart from zero

## Dispatch-then-await shape

The canonical fan-out from `example-fan-out-fan-in-go` — dispatch all children first, await all second:

```go
type FanoutArgs struct {
    Channels []string `json:"channels"`
    Message  string   `json:"message"`
}

type FanoutResult struct {
    Delivered []Delivery `json:"delivered"`
}

type SendArgs struct {
    Channel string `json:"channel"`
    Message string `json:"message"`
}

type Delivery struct {
    Channel string `json:"channel"`
    OK      bool   `json:"ok"`
    Reason  string `json:"reason,omitempty"`
}

func fanout(ctx *resonate.Context, args FanoutArgs) (FanoutResult, error) {
    // Dispatch loop — create all durable promises before awaiting any.
    futures := make([]*resonate.Future, 0, len(args.Channels))
    for _, ch := range args.Channels {
        f, err := ctx.RPC("send", SendArgs{Channel: ch, Message: args.Message})
        if err != nil {
            return FanoutResult{}, err
        }
        futures = append(futures, f)
    }

    // Await loop — read results in order; record per-child failures.
    out := FanoutResult{Delivered: make([]Delivery, 0, len(futures))}
    for i, f := range futures {
        var d Delivery
        if err := f.Await(&d); err != nil {
            out.Delivered = append(out.Delivered, Delivery{
                Channel: args.Channels[i],
                OK:      false,
                Reason:  err.Error(),
            })
            continue
        }
        out.Delivered = append(out.Delivered, d)
    }
    return out, nil
}
```

The dispatch loop runs serially in Go code, but each `ctx.RPC` creates an independent durable promise on the server — all children execute concurrently. The await loop only reads results; it does not gate execution of siblings.

## Recursive fan-out

A workflow can call `ctx.RPC(SameName, smallerArgs)` to recurse. Each recursive call is its own durable promise; with multiple workers running the same registered name, the promise tree fans out across them automatically. From `example-recursive-factorial-go`:

```go
// factorial/factorial.go

const Name = "Factorial"
const WorkerGroup = "factorial-workers"

type Args struct {
    N int `json:"n"`
}

func Workflow(ctx *resonate.Context, args Args) (int, error) {
    // Base case — must always be present to terminate the recursion.
    if args.N <= 1 {
        return 1, nil
    }
    // Recurse via ctx.RPC so the sub-call is dispatched as a separate
    // durable promise; with multiple workers it may run on a different process.
    f, err := ctx.RPC(Name, Args{N: args.N - 1})
    if err != nil {
        return 0, err
    }
    var sub int
    if err := f.Await(&sub); err != nil {
        return 0, err
    }
    return args.N * sub, nil
}
```

Worker binary — joins the named group so only registered workers receive dispatches:

```go
// cmd/worker/main.go

import "github.com/resonatehq/resonate-sdk-go/httpnet"

r, err := resonate.New(resonate.Config{
    Network: httpnet.NewHTTP(url, httpnet.HTTPOptions{
        PID:   pid,
        Group: factorial.WorkerGroup, // "factorial-workers"
    }),
})
if _, err := resonate.Register(r, factorial.Name, factorial.Workflow); err != nil {
    log.Fatalf("Register: %v", err)
}
// block on SIGINT / SIGTERM; do NOT call r.Stop() on long-running workers
```

Client binary — uses a distinct group so it cannot be mistakenly assigned worker tasks:

```go
// cmd/client/main.go

r, err := resonate.New(resonate.Config{
    Network: httpnet.NewHTTP(url, httpnet.HTTPOptions{
        PID:   pid,
        Group: "factorial-client", // distinct — not in WorkerGroup
    }),
})
target := fmt.Sprintf("poll://any@%s", factorial.WorkerGroup)
h, err := r.RPC(ctx, "factorial-10", factorial.Name, factorial.Args{N: 10},
    resonate.RPCOptions{Target: target})
var result int
if err := h.Result(ctx, &result); err != nil { ... }
```

## Collecting results and partial-failure handling

The `example-fan-out-fan-in-go` pattern records per-child failures rather than aborting the parent at the first error. Use this when you want a summary of all deliveries rather than a short-circuit:

```go
for i, f := range futures {
    var d Delivery
    if err := f.Await(&d); err != nil {
        // Record the failure and continue to the next child.
        out.Delivered = append(out.Delivered, Delivery{
            Channel: channels[i],
            OK:      false,
            Reason:  err.Error(),
        })
        continue
    }
    out.Delivered = append(out.Delivered, d)
}
```

To fail-fast on the first child error, replace the `continue` block with `return ..., err`.

## `ctx.Run` vs `ctx.RPC` for fan-out

| | `ctx.Run` | `ctx.RPC` |
|---|---|---|
| Execution target | Same process | Remote process (by registered name) |
| Fan-out across workers | No | Yes — server distributes to the named group |
| Recursion across workers | No | Yes — each `ctx.RPC(SameName, ...)` is re-dispatched |
| Best for | In-process leaf tasks | Distributed branches, cross-worker recursion |

Both methods return `*resonate.Future` immediately. Both support the two-loop fan-out pattern. `ctx.Run` functions must return promptly — long-running or blocking work belongs in `ctx.RPC`.

## Distinct Go idioms

- **`[]*resonate.Future` slice with `make(..., 0, len(items))`** — pre-allocate to the known capacity; the two-loop pattern iterates this slice twice (dispatch, then await).
- **Two-loop pattern is mandatory** — a single loop that dispatches and awaits in the same iteration serializes children, defeating the purpose of fan-out.
- **Struct args with `json` tags** — args are JSON-encoded into the durable promise payload; use exported fields with `json` tags on every type that crosses `ctx.RPC` or `ctx.Run`.
- **`WorkerGroup` constant** — define group names as shared constants in a package imported by both worker and client binaries (see `factorial.WorkerGroup`). This prevents string drift between the `httpnet.HTTPOptions{Group: ...}` in the worker and the `poll://any@...` target in the client.
- **Distinct client group** — processes that only create promises (CLI tools, HTTP gateways, test harnesses) must use a different group name from the worker group. Without this, the server dispatches tasks to the client process, which hasn't registered the function, and the task is dropped as "function not registered".
- **`r.Stop()` only on short-lived processes** — call it on demo binaries and one-shot clients. Do not call it on long-running workers; let SIGINT/SIGTERM end the lifecycle.
- **Replay safety** — the whole workflow body re-runs on resume. The dispatch loop deterministically re-issues the same `ctx.RPC` / `ctx.Run` calls, and already-settled children short-circuit. No replay guard code is needed.

## Avoid

- **Awaiting inside the dispatch loop** — `ctx.RPC` followed immediately by `f.Await` in the same loop body runs children sequentially. Collect all futures first, then await.
- **Unbounded recursion without a base case** — the server-side promise graph is not free; always guard `ctx.RPC(SameName, ...)` with a termination condition (`if args.N <= 1`, `if depth == 0`, etc.).
- **Sharing the default group between client and worker** — `Config.URL` shorthand builds a transport in the `"default"` group. If both the worker and the client use `Config.URL`, the client joins the worker pool and steals dispatches it cannot fulfill.

## Related skills

- `resonate-basic-durable-world-usage-go` — `ctx.Run`, `ctx.RPC`, `Future.Await`, options structs
- `resonate-saga-pattern-go` — fan-out alongside compensating transactions
- `durable-execution` — foundational replay semantics
- `resonate-recursive-fan-out-pattern-typescript` — canonical mental model, deduplication strategy
- `resonate-recursive-fan-out-pattern-rust` — `.spawn()` / `Vec<DurableFuture>` sibling
