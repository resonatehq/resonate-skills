---
name: resonate-basic-ephemeral-world-usage-go
description: Core patterns for using the Resonate Go SDK's Client APIs from the ephemeral world — initializing a Resonate instance, registering durable functions with the package-level generic resonate.Register, invoking them top-level via RegisteredFunc.Run, dispatching remotely with Resonate.RPC, reconnecting to an existing execution with Resonate.Get, reading typed and untyped handle results, RunOptions, and Stop semantics. Pre-release caveat: the Go SDK has no semver-tagged release yet; APIs may change before the first tag is cut.
license: Apache-2.0
---

# Resonate Basic Ephemeral World Usage — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx` and the `resonatehq-examples/*-go` repos at SDK commit `22076134651f`.

## Overview

The ephemeral world is wherever your Go program starts: `main()`, an HTTP handler, a CLI entry point, a background goroutine. You use a `*resonate.Resonate` instance to register durable functions and invoke them top-level. Once an invocation starts, it crosses into the durable world and Resonate guarantees its completion — retries on failure, resumes after crashes, continues across process restarts.

This skill covers the Client API surface only. The Durable World (Context APIs inside workflow functions) is covered in `resonate-basic-durable-world-usage-go`.

## Install

```shell
go get github.com/resonatehq/resonate-sdk-go@latest
```

Pin a specific commit for reproducible builds until a semver tag is published:

```shell
go get github.com/resonatehq/resonate-sdk-go@22076134651f
```

Core imports:

```go
import (
    resonate "github.com/resonatehq/resonate-sdk-go"
    "github.com/resonatehq/resonate-sdk-go/localnet"  // zero-dependency dev
    "github.com/resonatehq/resonate-sdk-go/httpnet"   // named worker groups
)
```

## Basic Shape

A complete one-file program: register, run, read, stop.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    resonate "github.com/resonatehq/resonate-sdk-go"
)

type GreetArgs struct {
    Name string `json:"name"`
}

func greet(_ *resonate.Context, args GreetArgs) (string, error) {
    return fmt.Sprintf("hello, %s!", args.Name), nil
}

func main() {
    r, err := resonate.New(resonate.Config{URL: "http://localhost:8001"})
    if err != nil {
        log.Fatalf("resonate.New: %v", err)
    }
    defer func() { _ = r.Stop() }()

    greetFn, err := resonate.Register(r, "greet", greet)
    if err != nil {
        log.Fatalf("Register: %v", err)
    }

    ctx := context.Background()
    id := fmt.Sprintf("hello-%d", time.Now().UnixNano())

    h, err := greetFn.Run(ctx, id, GreetArgs{Name: "world"})
    if err != nil {
        log.Fatalf("Run: %v", err)
    }

    out, err := h.Result(ctx)
    if err != nil {
        log.Fatalf("Result: %v", err)
    }
    fmt.Println(out) // hello, world!
}
```

## Initialization

### Remote server

```go
r, err := resonate.New(resonate.Config{URL: "http://localhost:8001"})
```

`Config` fields:

| Field | Type | Default | Purpose |
|---|---|---|---|
| `URL` | `string` | unset | Shorthand HTTP transport at this address (default group `"default"`). |
| `Network` | `Network` | nil | Explicit transport — use when `URL` is empty. |
| `Heartbeat` | `Heartbeat` | `AsyncHeartbeat` at `TTL/2` | Keeps acquired task leases alive. |
| `Encryptor` | `Encryptor` | `NoopEncryptor` | Codec encryption at the durability boundary. |
| `TTL` | `time.Duration` | `60s` | Per-task lease duration. |
| `Prefix` | `string` | empty | Prepended (with `:`) to every promise/task ID. |
| `Token` | `string` | unset | Sent as Bearer auth on every protocol request. |

Network precedence: `Config.URL` → `Config.Network` → `RESONATE_URL` env var → `resonate.ErrNetworkRequired`. The only env var the constructor reads is `RESONATE_URL`.

### Zero-dependency development (localnet)

```go
pid := "dev-worker"
r, err := resonate.New(resonate.Config{
    Network:   localnet.NewLocal("default", &pid),
    Heartbeat: resonate.NoopHeartbeat{}, // required — localnet has no lease endpoint
})
```

### Named worker group

There is no `Group` field on `Config`. Pass the transport explicitly:

```go
r, err := resonate.New(resonate.Config{
    Network: httpnet.NewHTTP("http://localhost:8001", httpnet.HTTPOptions{
        Group: "worker-group-a",
    }),
})
```

### Authentication

```go
r, err := resonate.New(resonate.Config{
    URL:   "https://resonate.example.com",
    Token: os.Getenv("RESONATE_TOKEN"),
})
```

## `resonate.Register`

`Register` is a package-level generic function (Go has no method-level generics). It takes the instance, a name, and the function; returns a typed `*RegisteredFunc[A, R]` and an error.

```go
greetFn, err := resonate.Register(r, "greet", greet)
if err != nil {
    log.Fatalf("Register: %v", err)
}
```

Registered functions must have the canonical signature `func(*resonate.Context, A) (R, error)`. Use `struct{}` for `A` or `R` when the function takes no input or produces no meaningful result.

## `RegisteredFunc.Run`

Invokes a registered function in the **same process** and returns a typed `*TypedHandle[R]`. The returned handle is durable — the function will complete even if the process crashes and another worker picks it up.

```go
h, err := greetFn.Run(ctx, "greet-1", GreetArgs{Name: "world"})
if err != nil {
    log.Fatalf("Run: %v", err)
}

result, err := h.Result(ctx) // returns (string, error) — typed
if err != nil {
    log.Fatalf("Result: %v", err)
}
```

### RunOptions

Pass as a trailing argument to `Run`:

```go
h, err := greetFn.Run(ctx, "greet-1", GreetArgs{Name: "world"}, resonate.RunOptions{
    Timeout:     60 * time.Second,
    Target:      "poll://any@workers",
    Tags:        map[string]string{"team": "checkout"},
    RetryPolicy: resonate.ConstantRetry{MaxAttempts: 3, Delay: time.Second},
})
```

| Field | Purpose |
|---|---|
| `Timeout` | Caps the root promise deadline. Zero uses `DefaultTopLevelTimeout` (24h). |
| `Target` | Logical routing address (`resonate:target` tag). Empty falls back to the configured group. |
| `Tags` | Merged into the root promise's tag set. |
| `RetryPolicy` | Re-execution policy on error. Nil applies `DefaultRetryPolicy` (exponential, 3 attempts). Only takes effect when this worker wins the create-and-acquire race and runs the function locally; a task picked up by another worker via server push uses `DefaultRetryPolicy`. |
| `Version` | Reserved — declared but not yet consumed ([issue #5](https://github.com/resonatehq/resonate-sdk-go/issues/5)). |

## `Resonate.RPC`

Invokes a registered function in a **remote process** by name. Returns an untyped `*Handle`; the target function does not need to be registered locally.

```go
h, err := r.RPC(ctx, "greet-1", "greet", GreetArgs{Name: "world"})
if err != nil {
    log.Fatalf("RPC: %v", err)
}

var result string
if err := h.Result(ctx, &result); err != nil {
    log.Fatalf("Result: %v", err)
}
```

`RPC` accepts an optional `resonate.RPCOptions{Timeout, Target, Tags, Version}` as a trailing argument.

## `Resonate.Get`

Gets a handle to an existing execution by promise ID. Returns `*resonate.ServerError` with `Code: 404` when the promise does not exist.

```go
h, err := r.Get(ctx, "greet-1")
if err != nil {
    log.Fatalf("Get: %v", err)
}

var result string
if err := h.Result(ctx, &result); err != nil {
    log.Fatalf("Result: %v", err)
}
```

## Reading Handle Results

- **Typed handle** — `RegisteredFunc.Run` returns `*TypedHandle[R]`; call `h.Result(ctx)` to get `(R, error)` directly.
- **Untyped handle** — `RPC` and `Get` return `*Handle`; call `h.Result(ctx, &out)` and pass a pointer to the target variable.
- **Generic helper for untyped handles:**

```go
result, err := resonate.ResultOf[string](ctx, h)
```

A rejected promise surfaces as `*resonate.ApplicationError` (or the deserialized concrete error where available).

## Stop Semantics

```go
defer func() { _ = r.Stop() }()
```

`Stop` closes the network connection, stops the heartbeat loop, and cancels the background subscription-refresh goroutine. It is idempotent.

Call `Stop` from processes that should exit after their work finishes: demo binaries, one-shot jobs, CI tasks, examples. Without it, background goroutines keep the process alive after `main` would otherwise return.

**Do not call `Stop` on a long-running worker.** Calling it tears down the channels a worker uses to receive and hold work:

- The connection to the Resonate Server closes — the worker stops receiving dispatched tasks.
- The heartbeat loop stops — the server-side TTL on in-flight tasks expires, and the server reassigns them.
- The subscription-refresh goroutine is cancelled — listeners on awaited promises are no longer re-registered.

The worker process keeps running but silently stops processing. Let `SIGINT` / `SIGTERM` end a worker's lifecycle instead.

## Distinct Go Idioms

- **`if err != nil` everywhere** — both `resonate.New`, `resonate.Register`, `Run`, `RPC`, `Get`, and `Result` return errors. Use explicit checks; no method chaining.
- **Option structs, not builders** — `RunOptions`, `RPCOptions`, `RunOpts`, `RPCOpts` are plain structs with exported fields. Pass them as trailing arguments; zero values mean "use the default."
- **Package-level generic `resonate.Register`** — Go has no method-level generics, so registration is a free function, not a method on the instance. This is why `greetFn` carries the type parameters, not `r`.
- **Exported struct fields with `json` tags for args** — args are JSON-serialized into the durable promise so any worker that resumes the function can decode them. Unexported fields are silently dropped.
- **`context.Context` is `ctx` passed to Run/RPC/Get/Result** — this is the standard `context.Context` (cancellation, deadlines), not the `*resonate.Context` inside a workflow.
- **`defer func() { _ = r.Stop() }()`** — the closure-wrapped defer is idiomatic Go for deferred calls where you want to discard the error without a naked `_` assignment at the call site.
- **`time.Duration` for timeouts** — `60 * time.Second`, not raw integers. Durations are native `time.Duration` throughout the SDK.

## Avoid

- Calling `resonate.New` with neither `URL` nor `Network` and without `RESONATE_URL` set — returns `ErrNetworkRequired` at startup.
- Using `localnet` without `Heartbeat: resonate.NoopHeartbeat{}` — the default `AsyncHeartbeat` issues HTTP keep-alive requests that localnet cannot serve; the heartbeat loop errors.
- Passing a `Group` on `Config` — there is no `Group` field. Use `httpnet.NewHTTP(..., httpnet.HTTPOptions{Group: "..."})` via `Config.Network`.
- Using Client APIs (`r.RPC`, `greetFn.Run`) inside a durable function — those belong to the Ephemeral World. Use `ctx.RPC` and `ctx.Run` inside workflows.
- Calling `Stop` on a long-running worker (see Stop Semantics above).
- Relying on `Config.Version` in `RunOptions` / `RPCOptions` — the field is declared but not yet consumed.

## Related Skills

- `resonate-basic-durable-world-usage-go` — Context APIs: `ctx.Run`, `ctx.RPC`, `ctx.Sleep`, `ctx.Promise`, `ctx.Detached`, fan-out patterns, retry policies, context accessors
- `durable-execution` — foundational concepts; read this first if new to Resonate
- `resonate-defaults` — SDK-wide defaults reference (TTL, retry policy, top-level timeout, child timeout)
- `resonate-basic-ephemeral-world-usage-typescript` — TypeScript sibling for comparison
- `resonate-basic-ephemeral-world-usage-rust` — Rust sibling for comparison
