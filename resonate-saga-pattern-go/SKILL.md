---
name: resonate-saga-pattern-go
description: Implement saga patterns for distributed transactions in Go with Resonate — forward steps tracked in a slice with compensating actions that unwind in reverse on failure, using explicit (T, error) returns and a type Step string + switch for compensation dispatch. Use when coordinating multi-step Go workflows that need consistency across failures without a distributed transaction. Pre-release caveat: the Go SDK has no semver-tagged release yet; every code block here is verified against develop/go.mdx at commit 22076134651f.
license: Apache-2.0
---

# Resonate Saga Pattern — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx`; there is no Go saga example repo yet, so all snippets use only documented SDK surface at commit `22076134651f`.

## Overview

A saga is a long-running transaction split into discrete steps, each with a compensating action. Forward steps execute top-to-bottom; on any failure, completed compensations run bottom-to-top. Because the whole workflow body re-runs on resume, the `completed []Step` slice is reconstructed deterministically on replay — each `ctx.Run` call short-circuits its settled durable promise in the same order, so the slice is rebuilt identically before reaching the point of failure.

For the language-agnostic mental model — including choreography vs orchestration, idempotency requirements, and when not to use sagas — see `resonate-saga-pattern-typescript`.

## When to use

- Multi-step workflow where intermediate state is visible to other systems between steps
- Each step is individually compensable (inventory hold, payment charge, shipment create)
- You need "all or nothing" consistency without a distributed 2PC transaction
- Steps span services and may take seconds to minutes each
- Compensation logic exists and is idempotent for every committed step

## Basic shape

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "time"

    resonate "github.com/resonatehq/resonate-sdk-go"
)

// Step is a string enum for the forward steps that have been committed.
// Using type Step string + a switch gives idiomatic Go compensation dispatch.
type Step string

const (
    StepInventory Step = "inventory"
    StepPayment   Step = "payment"
    StepShipment  Step = "shipment"
)

type OrderArgs struct {
    OrderID string `json:"order_id"`
}

type SagaResult struct {
    Status      string   `json:"status"`
    OrderID     string   `json:"order_id"`
    Compensated []string `json:"compensated,omitempty"`
}

type CompensateArgs struct {
    Step    Step   `json:"step"`
    OrderID string `json:"order_id"`
}

// placeOrder is the saga orchestrator. It runs forward steps in sequence,
// tracking each committed step. On the first failure it runs compensations in
// reverse, each as its own durable ctx.Run so they are retryable.
func placeOrder(ctx *resonate.Context, args OrderArgs) (SagaResult, error) {
    var completed []Step

    // --- forward path ---
    fInv, err := ctx.Run(reserveInventory, args.OrderID)
    if err != nil {
        return SagaResult{}, err
    }
    if err := fInv.Await(nil); err != nil {
        return compensateAll(ctx, args.OrderID, completed, err)
    }
    completed = append(completed, StepInventory)

    fPay, err := ctx.Run(chargePayment, args.OrderID)
    if err != nil {
        return SagaResult{}, err
    }
    if err := fPay.Await(nil); err != nil {
        return compensateAll(ctx, args.OrderID, completed, err)
    }
    completed = append(completed, StepPayment)

    fShip, err := ctx.Run(createShipment, args.OrderID)
    if err != nil {
        return SagaResult{}, err
    }
    if err := fShip.Await(nil); err != nil {
        return compensateAll(ctx, args.OrderID, completed, err)
    }
    completed = append(completed, StepShipment)

    return SagaResult{Status: "success", OrderID: args.OrderID}, nil
}

// compensateAll iterates completed steps in reverse and runs each compensation
// as a durable ctx.Run. Returns a SagaResult with status "failed" on success,
// or propagates the compensation error if a compensation itself fails fatally.
func compensateAll(
    ctx *resonate.Context,
    orderID string,
    completed []Step,
    forwardErr error,
) (SagaResult, error) {
    _ = forwardErr // log or inspect as needed

    var names []string
    for i := len(completed) - 1; i >= 0; i-- {
        step := completed[i]
        f, err := ctx.Run(compensate, CompensateArgs{Step: step, OrderID: orderID},
            resonate.RunOpts{
                Timeout: 10 * time.Minute,
                RetryPolicy: resonate.ExponentialRetry{
                    MaxAttempts: 5,
                    Base:        200 * time.Millisecond,
                    Max:         30 * time.Second,
                    Jitter:      true,
                },
            },
        )
        if err != nil {
            return SagaResult{}, fmt.Errorf("dispatch compensation for %s: %w", step, err)
        }
        if err := f.Await(nil); err != nil {
            // A compensation that exhausts retries is a real inconsistency — log and alert.
            return SagaResult{}, fmt.Errorf("compensation for %s failed: %w", step, err)
        }
        names = append(names, string(step))
    }

    return SagaResult{
        Status:      "failed",
        OrderID:     orderID,
        Compensated: names,
    }, nil
}

// compensate dispatches to the correct undo action via a switch on Step.
// Each branch must be idempotent — compensations may be retried.
func compensate(_ *resonate.Context, args CompensateArgs) (struct{}, error) {
    switch args.Step {
    case StepShipment:
        // cancelShipment(args.OrderID)
        return struct{}{}, nil
    case StepPayment:
        // refundPayment(args.OrderID)
        return struct{}{}, nil
    case StepInventory:
        // releaseInventory(args.OrderID)
        return struct{}{}, nil
    default:
        return struct{}{}, resonate.NewNonRetryable(
            fmt.Errorf("unknown saga step %q — programming error", args.Step),
        )
    }
}

// Forward leaf stubs — replace with real service calls.
func reserveInventory(_ *resonate.Context, orderID string) (struct{}, error) {
    _ = orderID
    return struct{}{}, nil
}

func chargePayment(_ *resonate.Context, orderID string) (struct{}, error) {
    _ = orderID
    return struct{}{}, nil
}

func createShipment(_ *resonate.Context, orderID string) (struct{}, error) {
    _ = orderID
    // Simulate a transient failure to trigger compensation.
    return struct{}{}, errors.New("shipment service unavailable")
}

func main() {
    r, err := resonate.New(resonate.Config{URL: "http://localhost:8001"})
    if err != nil {
        log.Fatalf("resonate.New: %v", err)
    }
    defer func() { _ = r.Stop() }()

    // Only the workflow needs registration; leaves reached via ctx.Run do not.
    placeOrderFn, err := resonate.Register(r, "placeOrder", placeOrder)
    if err != nil {
        log.Fatalf("Register: %v", err)
    }

    h, err := placeOrderFn.Run(context.Background(), "order-1", OrderArgs{OrderID: "order-1"})
    if err != nil {
        log.Fatalf("Run: %v", err)
    }
    res, err := h.Result(context.Background())
    if err != nil {
        log.Fatalf("Result: %v", err)
    }
    fmt.Printf("saga: %+v\n", res)
}
```

Each `ctx.Run` + `Await` pair is a durable checkpoint. If the worker crashes mid-saga, Resonate resumes from the last settled step — completed forward steps do not re-execute.

## Compensation must be retryable

A compensation that exhausts retries leaves the system in a real inconsistency. Always give compensations a generous timeout and an explicit retry policy:

```go
f, err := ctx.Run(compensate, CompensateArgs{Step: step, OrderID: orderID},
    resonate.RunOpts{
        Timeout: 10 * time.Minute,
        RetryPolicy: resonate.ExponentialRetry{
            MaxAttempts: 5,
            Base:        200 * time.Millisecond,
            Max:         30 * time.Second,
            Jitter:      true,
        },
    },
)
```

If `f.Await(nil)` returns an error after retries are exhausted, you have a manual-intervention situation — log the step name, the order ID, and alert before returning the error.

To fail a forward step immediately without retrying (useful for non-idempotent external APIs):

```go
return struct{}{}, resonate.NewNonRetryable(errors.New("card declined — do not retry"))
```

## Step dispatch via `type Step string` + switch

Go has no enum keyword. A `type Step string` with named constants gives the closed-step-set semantics of Rust's enum while staying serializable without extra infrastructure:

```go
type Step string

const (
    StepInventory Step = "inventory"
    StepPayment   Step = "payment"
    StepShipment  Step = "shipment"
)

// In the compensation leaf:
switch args.Step {
case StepShipment:
    // ...
case StepPayment:
    // ...
case StepInventory:
    // ...
default:
    return struct{}{}, resonate.NewNonRetryable(
        fmt.Errorf("unknown saga step %q", args.Step),
    )
}
```

The `default` branch with `NewNonRetryable` turns a programming error (a new step added without a compensation) into a fast, observable failure rather than a silent infinite retry loop.

## Distinct Go idioms

- **Explicit `(T, error)` returns** handle forward-path propagation — there is no exception unwinding. You must check each `if err != nil` explicitly and call `compensateAll` yourself rather than relying on a `try/except` unwind.
- **`if err := f.Await(nil); err != nil` immediately after dispatch** keeps each step sequential and is idiomatic for the saga forward path. Fan-out (dispatch all, then await all) is used for parallel work — sagas are sequential by nature.
- **`completed []Step` slice rebuilt on replay** — on crash-resume the workflow body re-runs from the top; each `ctx.Run` call short-circuits its settled promise in order, so the slice is reconstructed identically before reaching the failed step.
- **`for i := len(completed) - 1; i >= 0; i--`** is the idiomatic Go reverse loop — no `reversed()` builtin or `slices.Reverse` needed.
- **`type CompensateArgs struct`** with exported fields and `json` tags bundles step + order ID into a single serializable arg, matching the SDK's encoding contract.
- **`struct{}`** as the result type for void leaf functions — idiomatic Go; avoids a named unit type.

## Avoid

- **Panicking inside a durable function** — `panic` in a `ctx.Run` leaf will crash the worker, not run compensations. Always return `error` explicitly.
- **Non-idempotent compensations** — each compensation `ctx.Run` may execute more than once on retry. A compensation that double-refunds or double-releases is worse than the original failure.
- **Reading the system-of-record without a fresh `ctx.Run`** — a raw DB/API call in the workflow body (outside a leaf) re-executes on every replay. Wrap any observable side effect in its own `ctx.Run` so the result is checkpointed.
- **Swallowing compensation errors** — if `f.Await(nil)` returns an error during the compensation loop, surface it. Silently continuing produces a partially compensated saga with no audit trail.

## Related skills

- `resonate-basic-durable-world-usage-go` — `ctx.Run`, `RunOpts`, function signature variants
- `resonate-recursive-fan-out-pattern-go` — parallel dispatch, can be composed with saga steps that fan out internally
- `resonate-external-system-of-record-pattern-go` — idempotency keys and SoR reads inside durable leaves
- `durable-execution` — foundational replay semantics that make the `completed` slice safe
- `resonate-saga-pattern-typescript` — language-agnostic mental model, choreography vs orchestration, idempotency patterns
- `resonate-saga-pattern-rust` — Rust sibling using `Result<T>` + `?` and enum match
