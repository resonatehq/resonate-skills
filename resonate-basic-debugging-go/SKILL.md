---
name: resonate-basic-debugging-go
description: Debug and troubleshoot Resonate applications using the Go SDK. Use when investigating stuck or never-resuming workflows, duplicated side effects after replay, promise decode errors, latent-promise settlement encoding traps, localnet heartbeat failures, or the pre-release caveats of the Go SDK. Pre-release: no semver tag yet; pin a commit for stability.
license: Apache-2.0
---

# Resonate Basic Debugging — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx` and the `resonatehq-examples/*-go` repos at SDK commit `22076134651f`.

## Overview

Go's type system catches some bugs at compile time, but several Go-specific traps only surface at workflow-execution time. The most dangerous ones are silent: wrong leaf signatures, bad promise encoding, and `r.Stop()` on a live worker all fail without a clear error at the point of mistake. This skill is a symptom-first guide to those failure modes.

For the language-agnostic replay and recovery mental model, read `durable-execution` first.

## Triage flow

1. Is the worker connected? Confirm `resonate.New` did not return an error and that the correct `URL`/`Network` is set.
2. Is the function registered? `resonate.Register` returns `(RegisteredFunc, error)` — unwrap and log.
3. Is the promise stuck? Run `resonate promise get <id>` to check state (`pending` / `resolved` / `rejected` / `timedout`).
4. Is the workflow replaying but producing duplicates? An un-checkpointed side effect is re-running above a durable boundary.
5. Does `Future.Await` return a decode error after external settlement? Likely a `resonate.NewValue` encoding mismatch.
6. Is the worker up but not picking up work? `r.Stop()` may have been called on a live worker.

---

## Stuck / never-resuming workflows

### Latent promise never settled

**Symptom:** `Future.Await` blocks indefinitely; `resonate promise get <id>` shows state `pending`.

**Causes:**
- The external actor never called `PromiseSettle` / `resonate promise resolve <id>`.
- The promise was settled but the value encoding is wrong — `Future.Await` fails with a decode error and the workflow re-suspends.

The external settlement path via `r.Sender().PromiseSettle` requires the value to be encoded as JSON → base64 → quoted string stored in `Value.Data`. Using `resonate.NewValue(x)` stores raw JSON without the base64 layer, so `Future.Await` fails silently with a decode error (sdk-go issue #28):

```go
// WRONG — NewValue stores raw JSON; Codec.Decode fails with a base64 error.
val := resonate.NewValue(decision)

// CORRECT — JSON → base64 → JSON-quoted string.
rawJSON, _ := json.Marshal(decision)
b64 := base64.StdEncoding.EncodeToString(rawJSON)
quotedB64, _ := json.Marshal(b64)
val := resonate.Value{Data: json.RawMessage(quotedB64)}

settleReq := resonate.PromiseSettleReq{
    ID:    promiseID,
    State: resonate.SettleStateResolved,
    Value: val,
}
_, err := r.Sender().PromiseSettle(ctx, settleReq)
```

There is no compile-time or runtime warning at the settle call site. The failure only surfaces inside `Future.Await` on the workflow side. Until sdk-go issue #28 lands a higher-level `Promises().Resolve` API, replicate the encoding above exactly.

### `ctx.Run` leaf blocks indefinitely

**Symptom:** the workflow task lease expires and the server reassigns the task; workflow appears to restart rather than resume; repeated attempts never complete.

**Cause:** the runtime joins every `ctx.Run`-spawned goroutine before it can suspend or fulfill the parent task. A function that does external I/O, waits on a channel, or sleeps for a long time inside `ctx.Run` holds the lease open until the TTL expires (default 60 s).

**Fix:** move long-running or external-blocking work into `ctx.RPC` (remote dispatch, workflow suspends cleanly) or `ctx.Promise` (latent promise settled by an external actor). Reserve `ctx.Run` for pure in-process computation that returns quickly.

---

## Duplicated side effects (replay)

**Symptom:** emails, charges, log entries, or DB writes happen more than once per logical invocation.

**Cause:** the entire workflow body re-runs from the top on every resume. Durable child promises short-circuit work that already settled, but code that runs *before* reaching a durable boundary (`ctx.Sleep`, `ctx.RPC`, `ctx.Promise`) executes again on each replay pass.

```go
// BAD — the log line re-executes on every replay pass.
func myWorkflow(ctx *resonate.Context, id string) (string, error) {
    log.Printf("charging card for order %s", id)   // runs on every replay
    f, err := ctx.Run(chargeCard, id)
    // ...
}

// GOOD — the side effect is inside a checkpointed ctx.Run; it runs once.
func myWorkflow(ctx *resonate.Context, id string) (string, error) {
    f, err := ctx.Run(chargeCard, id)   // result is checkpointed
    if err != nil {
        return "", err
    }
    var receipt string
    if err := f.Await(&receipt); err != nil {
        return "", err
    }
    return receipt, nil
}
```

**Rule:** any observable side effect (network call, write, notification) belongs inside its own `ctx.Run` or `ctx.RPC` so the durable promise records the result and short-circuits on replay.

---

## Decode and error handling

### Wrong leaf signature — silent runtime failure

**Symptom:** `ctx.Run(myLeaf, arg)` returns an error at execution time (not compile time): "unsupported function signature" or similar.

**Cause:** `ctx.Run` takes `fn any` and resolves the signature by reflection. The four valid shapes are:

| Signature | Notes |
|---|---|
| `func(*resonate.Context, A) (R, error)` | Full form |
| `func(*resonate.Context) (R, error)` | No args |
| `func(A) (R, error)` | Stateless leaf with args |
| `func() (R, error)` | Stateless leaf |

Any other shape (wrong return arity, missing `error`, pointer-receiver method) compiles fine and only fails at execution time.

**Mitigation:** add a compile-time type guard in a test or `init` block:

```go
// Fails at compile time if myLeaf's signature drifts.
var _ func(string) (string, error) = myLeaf
```

Also verify new leaf functions against a `localnet` run before shipping.

### `r.Get` on a missing promise

**Symptom:** call returns an error; caller does not know whether the promise does not exist yet or whether the transport failed.

**Fix:** type-assert with `errors.As` to distinguish a 404 from a transport error:

```go
h, err := r.Get(ctx, "order-123")
if err != nil {
    var se *resonate.ServerError
    if errors.As(err, &se) && se.Code == 404 {
        // Promise does not exist yet — normal during startup races.
        return
    }
    log.Fatalf("Get: %v", err) // unexpected transport or server error
}
```

### Rejected promise surfaces as `ApplicationError`

**Symptom:** `h.Result` or `f.Await` returns a non-nil error even though no Go panic occurred.

**Cause:** the promise was rejected (either by a returned error from the registered function, or by an external `resonate promise reject <id>` call). The error is deserialized as `*resonate.ApplicationError`.

```go
var result string
if err := f.Await(&result); err != nil {
    var ae *resonate.ApplicationError
    if errors.As(err, &ae) {
        log.Printf("workflow rejected: %s", ae.Message)
        return
    }
    log.Fatalf("unexpected await error: %v", err)
}
```

### Bounded `DefaultRetryPolicy` — not a bug

The Go SDK's `DefaultRetryPolicy` is `ExponentialRetry{MaxAttempts: 3, Base: 100ms, Max: 30s, Jitter: true}`. A workflow that "gives up too early" compared to TypeScript or Python expectations is hitting this 3-attempt ceiling, not a runtime defect. Override with a custom policy via `RunOpts.RetryPolicy`.

---

## Setup footguns (localnet, Stop)

### `localnet` without `NoopHeartbeat{}`

**Symptom:** heartbeat loop errors at startup; `resonate.New` or early task processing logs HTTP errors against an address that isn't serving.

**Cause:** the default `AsyncHeartbeat` issues HTTP keep-alive requests to refresh the task lease. `localnet` has no such endpoint.

**Fix:**

```go
pid := "dev-worker"
r, err := resonate.New(resonate.Config{
    Network:   localnet.NewLocal("default", &pid),
    Heartbeat: resonate.NoopHeartbeat{},
})
```

This is the only required deviation from the HTTP-server setup when using localnet.

### `r.Stop()` on a long-running worker

**Symptom:** the worker process is running and healthy-looking, but it stops picking up new tasks.

**Cause:** `r.Stop()` closes the server connection, stops the heartbeat loop, and cancels the subscription-refresh goroutine. Any in-flight leased tasks have their TTL expire; the server reassigns them. The process keeps running, but the dispatch pipeline is dead.

**Rule:** call `r.Stop()` only in one-shot binaries, demos, and CI tasks that exit after their work finishes. Long-running workers should stay up; end the process lifecycle with SIGINT / SIGTERM.

```go
// Correct for a one-shot job:
defer func() { _ = r.Stop() }()

// For a long-running worker — omit Stop and let the OS signal end the process.
```

---

## Inspection tools

```shell
resonate dev                                     # local dev server (in-process state)
resonate promise get <id>                        # single promise state + value
resonate promise search 'order:*'               # prefix search across promises
resonate promise resolve <id> --data '"approved"' # settle a pending latent promise
resonate tree <id>                               # call graph for an invocation
```

See the `resonate-cli` skill for the full command surface. The CLI is SDK-agnostic; the same commands work against any worker language.

**Durable sleep tolerance:** a 24 h `ctx.Sleep` firing in 23–25 h is within the server's timer tolerance window, not a bug.

---

## Avoid

- Branching on `time.Now()` or `rand.Float64()` directly inside a workflow body — non-deterministic values change between replay passes and cause divergent execution. Move them into a leaf so the result is checkpointed.
- Using `time.Duration` as a JSON-serializable arg type — it round-trips as a bare nanosecond `int64`, which is opaque in stored promise payloads. Prefer an explicit seconds or milliseconds field (e.g. `Secs int64`).
- Passing unexported struct fields or non-serializable types (channels, functions, `sync.Mutex`) as workflow args — `ctx.Run` and `ctx.RPC` encode args into the durable promise via JSON; non-serializable types produce a silent zero value or a marshal error.

---

## Related skills

- `resonate-basic-durable-world-usage-go` — Context APIs (`ctx.Run`, `ctx.RPC`, `ctx.Sleep`, `ctx.Promise`)
- `resonate-human-in-the-loop-pattern-go` — latent promise settlement, `PromiseSettle` encoding detail
- `resonate-cli` — full CLI command surface for promise inspection and settlement
- `resonate-defaults` — default TTL, retry policy, and timeout values across all SDKs
- `durable-execution` — foundational replay and recovery model
- `resonate-basic-debugging-typescript` — TypeScript sibling (`yield*`, group routing, determinism helpers)
- `resonate-basic-debugging-rust` — Rust sibling (serde, tokio runtime, `ctx` vs `info`)
