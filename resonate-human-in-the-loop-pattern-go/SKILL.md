---
name: resonate-human-in-the-loop-pattern-go
description: Implement human-in-the-loop workflows in Go with the Resonate SDK — durable functions that park on ctx.Promise() until an external actor settles the latent promise, preferably via the Promises().Resolve/Reject/Cancel sub-client (shipped in 0.1.0); the CLI, the server HTTP API, and the low-level Sender().PromiseSettle remain as alternates for cross-process or non-Go settlement.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern — Go

> **Version note.** The Go SDK's first tagged release is [`0.1.0`](https://github.com/resonatehq/resonate-sdk-go/releases/tag/0.1.0) (`go get github.com/resonatehq/resonate-sdk-go@0.1.0` — the tag has no `v` prefix, so `@latest` does not resolve to it). `0.1.0` shipped a top-level `promises` sub-client (`r.Promises()`), matching TypeScript's `resonate.promises` and Rust's `resonate.promises`. Every code block here is verified against the `0.1.0` tag source and `example-human-in-the-loop-go`.

## Overview

For the language-agnostic mental model, start with `resonate-human-in-the-loop-pattern-typescript`. The idea is identical: create a latent durable promise, hand its ID to the external actor who will settle it, and await — the workflow goroutine parks until settlement arrives, surviving any number of crashes or restarts.

**Resolution path.** TypeScript has `resonate.promises.settle(id, ...)`. Rust has `resonate.promises.resolve(id, ...)`. Go's `0.1.0` tag has `r.Promises().Resolve(id, v)` / `.Reject(id, v)` / `.Cancel(id, v)` — the same shape, and the preferred way to settle a promise from Go code in another process. Three more mechanisms remain useful for settling from *outside* Go entirely (an ops runbook, curl, a non-Go service) — listed below in order of preference.

## When to use

- Approval gates (budget, deploy, content moderation)
- Third-party webhook callbacks (Stripe, DocuSign, Twilio)
- Operator unblock steps in runbooks
- Any step where the decision or data originates outside the Resonate worker set

## Basic shape

### Workflow side — `ctx.Promise` → `f.ID()` → publish → `Await`

```go
import (
    "context"
    "fmt"
    "time"

    resonate "github.com/resonatehq/resonate-sdk-go"
)

type ReviewRequest struct {
    Item      string `json:"item"`
    Requester string `json:"requester"`
}

// approvalWorkflow parks until an external actor settles the latent promise.
// promiseIDs is a buffered channel (capacity 1) that hands the promise ID to
// whoever resolves it — swap this for a DB write, a notification queue, etc.
func approvalWorkflow(ctx *resonate.Context, req ReviewRequest) (string, error) {
    // Create a latent durable promise. No registered function is behind it;
    // it only settles when an external caller issues a promise-settle.
    f, err := ctx.Promise(resonate.PromiseOpts{Timeout: 24 * time.Hour})
    if err != nil {
        return "", fmt.Errorf("ctx.Promise: %w", err)
    }

    promiseID := f.ID() // hand this to the external resolver

    // Publish the promise ID inside a ctx.Run so the write is checkpointed.
    // On replay, ctx.Run re-issues with the same child promise ID and
    // short-circuits — the side-effect does not run twice.
    // ctx.Run takes a single args value: ctx.Run(fn, args, opts...). Capture the
    // values the leaf needs via the closure and pass struct{}{} as the (unused) arg.
    _, err = ctx.Run(func(_ struct{}) (struct{}, error) {
        // In production: write to DB, push to a notification queue, etc.
        fmt.Printf("  [workflow] awaiting approval for %q — promise ID: %s\n", req.Item, promiseID)
        promiseIDs <- promiseID // example: buffered channel to a local resolver
        return struct{}{}, nil
    }, struct{}{})
    if err != nil {
        return "", fmt.Errorf("publish promise ID: %w", err)
    }

    // Await parks the workflow until the promise settles. The decision value
    // encoded by the settler is decoded here.
    var decision string
    if err := f.Await(&decision); err != nil {
        return "", fmt.Errorf("await approval: %w", err)
    }

    return fmt.Sprintf("item %q approved: %s", req.Item, decision), nil
}

// promiseIDs is a buffered channel for single-process demos. Replace with a
// DB write or notification in production.
var promiseIDs = make(chan string, 1)
```

Key points:
- `ctx.Promise()` with no args is valid; opts are optional.
- `f.ID()` is available immediately after the call — the promise record is created synchronously.
- Publish the ID **inside a `ctx.Run`** so the publication is itself durable. A bare side-effect above `f.Await` re-runs on every replay pass.
- On replay after a crash, `ctx.Promise` re-issues with the same SDK-generated ID (idempotent). `f.Await` short-circuits when the promise has already settled.
- `f.Await(nil)` is valid when you need to know the promise settled but don't need the payload.

## Resolving from outside

### 1. `r.Promises()` sub-client (preferred — resolving from Go code)

Use this when resolving from Go code in another process (an HTTP webhook handler, an admin CLI). `Resolve`/`Reject`/`Cancel` route the value through the instance's `Codec` — the same encode path the workflow machinery uses — so you never hand-roll the base64 layer.

```go
import (
    "context"
    "log"

    resonate "github.com/resonatehq/resonate-sdk-go"
)

func resolveApproval(r *resonate.Resonate, promiseID string, decision string) error {
    rec, err := r.Promises().Resolve(context.Background(), promiseID, decision)
    // or: r.Promises().Reject(ctx, promiseID, err) / r.Promises().Cancel(ctx, promiseID, nil)
    if err != nil {
        return err
    }
    log.Printf("settled — state=%s", rec.State)
    return nil
}
```

`Resolve`/`Reject`/`Cancel` return `*resonate.ServerError{Code: 404}` when the promise does not exist. Pass a Go `error` to `Reject`/`Cancel` and it encodes as the standard error payload, so `Future.Await` on the workflow side deserializes it into an `*ApplicationError` — the same round-trip `DeserializeError` does elsewhere in the SDK.

### 2. CLI (ops gates, manual approval)

```shell
# approve
resonate promise resolve <promiseID> --data '"approved"'

# reject
resonate promise reject <promiseID> --data '"rejected: budget exceeded"'
```

The `--data` value is the JSON payload that will be decoded into the type you passed to `f.Await`. For a `string` awaiter, a quoted JSON string like `'"approved"'` is correct. See the `resonate-cli` skill for full flag reference.

### 3. Server HTTP API

POST directly to the Resonate Server's promise-settle endpoint with the promise ID and a JSON body. Use this from any language or tool that can make HTTP calls (curl, an admin script, a serverless function) that isn't going through an SDK's codec:

```shell
curl -s -X POST "http://localhost:8001/promises/${PROMISE_ID}/resolve" \
     -H "Content-Type: application/json" \
     -d '{"value":{"data":"ImFwcHJvdmVkIg=="}}'
```

The `data` field must be the base64 encoding of the JSON-serialized payload — see the encoding note in mechanism 4 for why.

### 4. Low-level Go SDK — `Sender().PromiseSettle`

This is the transport-level primitive `r.Promises().Resolve` is itself built on. Reach for it directly only when you need to bypass the codec (e.g. a value that's already wire-encoded) — otherwise prefer mechanism 1.

```go
import (
    "context"
    "encoding/base64"
    "encoding/json"
    "log"

    resonate "github.com/resonatehq/resonate-sdk-go"
)

func resolveApprovalLowLevel(r *resonate.Resonate, promiseID string, decision string) error {
    // The Codec stores values as: JSON bytes → base64 → JSON-quoted string
    // stored in Value.Data. Replicate it manually when going around the Codec.
    rawJSON, err := json.Marshal(decision)       // "approved" → `"approved"`
    if err != nil {
        return err
    }
    b64 := base64.StdEncoding.EncodeToString(rawJSON)  // → base64 string
    quoted, err := json.Marshal(b64)                    // → JSON-quoted string
    if err != nil {
        return err
    }
    val := resonate.Value{Data: json.RawMessage(quoted)}

    rec, err := r.Sender().PromiseSettle(context.Background(), resonate.PromiseSettleReq{
        ID:    promiseID,
        State: resonate.SettleStateResolved, // or SettleStateRejected
        Value: val,
    })
    if err != nil {
        return err
    }
    log.Printf("settled — state=%s", rec.State)
    return nil
}
```

**Known friction on this low-level path only:** `resonate.NewValue(decision)` stores raw JSON bytes in `Value.Data` WITHOUT the base64 wrapper the Codec expects, so `Future.Await` on the workflow side fails with a decode error and no warning at the settle call site. `r.Promises().Resolve` (mechanism 1) does not have this trap — it goes through the Codec automatically. The manual `JSON → base64 → quoted` encoding above is only needed when you deliberately bypass the sub-client.

## Cross-process example

The [`resonatehq-examples/example-node-drain-orchestrator-go`](https://github.com/resonatehq-examples/example-node-drain-orchestrator-go) shows a `net/http` gateway process resolving promises owned by a separate worker process — a realistic two-process boundary. Prefer `r.Promises().Resolve(...)` (mechanism 1) for new code; the example may predate the `0.1.0` sub-client and use `Sender().PromiseSettle` directly (mechanism 4) — check which path it takes before copying it verbatim. Refer to it for the full plumbing (shared server URL, token config, error handling on 409 Already Settled). Do not reproduce the drain-orchestrator example inline; it is a large multi-file app.

## Localnet setup (development only)

For single-process demos and tests, use localnet. No external server is needed.

```go
import (
    resonate "github.com/resonatehq/resonate-sdk-go"
    "github.com/resonatehq/resonate-sdk-go/localnet"
)

pid := "hitl-worker"
r, err := resonate.New(resonate.Config{
    Network:   localnet.NewLocal("default", &pid),
    Heartbeat: resonate.NoopHeartbeat{}, // required — localnet has no heartbeat endpoint
    TTL:       5 * time.Minute,
})
```

`r.Promises().Resolve` and `Sender().PromiseSettle` both work against localnet exactly as they do against a real server. The `-url` flag in `example-human-in-the-loop-go` switches between the two modes at runtime.

## Known gaps

- **No hand-chosen promise IDs.** Go's `ctx.Promise` generates the ID internally. Unlike TypeScript's `ctx.promise({ id: "approval/order-42" })`, you cannot pass a deterministic application-layer ID. Fetch the ID via `f.ID()` after creation and publish it explicitly.
- **No first-responder race helper.** There is no `select`-style mechanism to await whichever of several latent promises settles first. Workaround: create one promise per participant, await each in sequence, and have a coordinator cancel the others once a winner is known; or use a helper workflow that calls `r.Promises().Resolve` on a shared promise from the first responder's branch.

## Distinct Go idioms

- **Publish inside `ctx.Run`.** The Go SDK does not have generator-based checkpoint semantics (no `yield*`). Wrap every observable side effect — DB writes, notifications, the channel send that hands off the promise ID — inside `ctx.Run` or `ctx.RPC` so the result is checkpointed and the action is not repeated on replay.
- **Options struct, not method chain.** All context calls take an optional trailing struct: `ctx.Promise(resonate.PromiseOpts{Timeout: 24*time.Hour})`. The struct is optional; `ctx.Promise()` is valid when defaults are acceptable.
- **`f.Await(nil)` for fire-and-observe.** When you only need to know a promise settled (not its payload), pass `nil` to `Await`. Saves a decode round-trip.
- **`defer r.Stop()`.** Required in demo binaries and one-shot jobs; keeps background goroutines from blocking process exit. Do not call `Stop` on a long-lived worker process — it tears down the task-receive channel and the heartbeat loop silently.

## Avoid

- **Polling via `ctx.Sleep` + a status check.** Defeats the park-and-resume semantics; burns checkpoints and wall-clock time. Use `ctx.Promise` + `f.Await` instead.
- **`resonate.NewValue(v)` passed straight to `Sender().PromiseSettle`.** It omits the base64 layer the codec expects, causing a silent decode failure on the workflow side. Use `r.Promises().Resolve(...)` (mechanism 1) instead — it encodes correctly — or the manual `JSON → base64 → quoted` encoding shown in mechanism 4 if you must stay on the low-level path.
- **Settling before publishing the ID.** A narrow race: if the external resolver runs before the `ctx.Run` that publishes the ID has been checkpointed, a crash between those two lines can lose the ID. Always checkpoint the publication first (via `ctx.Run`), then await.
- **Using `ctx.Run` for long-blocking work.** `ctx.Run` goroutines must return promptly; a blocking call holds the task lease open until TTL expires. Long waits belong in `ctx.Promise` (latent, externally settled) or `ctx.RPC` (remote dispatch).

## Related skills

- `resonate-basic-ephemeral-world-usage-go` — `r.Promises()` / `r.Schedules()` sub-client reference
- `resonate-basic-durable-world-usage-go` — `ctx.Promise` builder, opts, and `Future.Await` details
- `resonate-cli` — `resonate promise resolve / reject / cancel` flag reference (mechanism 2)
- `durable-execution` — foundational replay semantics; why `f.Await` survives crashes
- `resonate-human-in-the-loop-pattern-typescript` — mental model and `resonate.promises.settle` reference
- `resonate-human-in-the-loop-pattern-rust` — `ctx.promise::<T>()` builder and `resonate.promises.resolve` reference
