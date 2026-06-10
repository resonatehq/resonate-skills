---
name: resonate-external-system-of-record-pattern-go
description: Maintain consistency between a Resonate Go workflow and an external system that owns the truth — a database, ledger, or third-party API — without distributed transactions, by wrapping every interaction with that system in its own idempotent ctx.Run step so the durable promise records the result and replay never re-fires the external call. Pre-release caveat: the Go SDK has no semver-tagged release; pin a commit for stability.
license: Apache-2.0
---

# Resonate External System of Record Pattern — Go

> **Pre-release caveat.** The Go SDK has no semver-tagged release yet — `go get …@latest` resolves to a pseudo-version; pin a commit for stability. APIs may change before the first tag is cut. Every code block here is verified against `develop/go.mdx`; there is no Go system-of-record example repo yet, so all snippets use only documented SDK surface at commit `22076134651f`.

## Overview

For the language-agnostic framing and the "Write Last, Read First" mental model, see `resonate-external-system-of-record-pattern-typescript`. This skill translates that model to idiomatic Go: explicit `(T, error)` returns, option structs, and `ctx.Run`-based checkpointing without a separate dependency-injection mechanism.

The core contract is simple: one system owns the truth. Resonate coordinates the workflow and guarantees at-least-once execution; the external system enforces consistency through its own primitives — idempotency keys, upserts, conditional writes, ledger deduplication.

## When to use

- A Resonate workflow must create, update, or read from a database, payment processor, ledger, or any external store that has its own durability.
- You need multi-step operations (reserve → charge → record) to be safe to retry without double-writes.
- You cannot use distributed transactions across Resonate's promise store and the external system.

Do not use this pattern when all writes live inside a single ACID transaction scope — just use the transaction.

## The replay principle — wrap every SoR call in `ctx.Run`

**This is the entire reason the pattern exists.**

Whenever a workflow suspends and resumes (after a `ctx.Sleep`, a remote `ctx.RPC`, or a pending `ctx.Promise`), the entire workflow body re-executes from the top. Durable child promises short-circuit steps that already settled — the external call is not re-fired, and the stored result is returned directly. But any external call that is NOT wrapped in a `ctx.Run` will execute again on every replay, producing double-writes.

Rule: **every read from or write to the external SoR must be inside its own `ctx.Run` leaf.** A bare HTTP call or DB write in workflow code will fire once per replay.

```go
// BAD — fires on every replay, may charge the card twice
func placeOrder(ctx *resonate.Context, args OrderArgs) (string, error) {
    chargeID, err := stripe.Charge(args.AmountCents, args.CardToken) // replay fires this again
    if err != nil {
        return "", err
    }
    // ...
}

// GOOD — checkpointed; on replay the stored chargeID is returned, Stripe is not called again
func placeOrder(ctx *resonate.Context, args OrderArgs) (string, error) {
    f, err := ctx.Run(chargeCard, args)
    if err != nil {
        return "", err
    }
    var chargeID string
    if err := f.Await(&chargeID); err != nil {
        return "", err
    }
    // ...
}
```

## Idempotency keys from `ctx.ID()`

A `ctx.Run` step can re-execute on retry before it settles (Resonate delivers at-least-once). The SoR must be addressed idempotently so a duplicate call is harmless.

Derive stable keys from `ctx.ID()`, `ctx.OriginID()`, or stable workflow args — never from `time.Now()` or `rand`, which produce different values across replays and produce different keys, defeating deduplication.

```go
type ChargeArgs struct {
    OrderID     string `json:"order_id"`
    AmountCents int64  `json:"amount_cents"`
    CardToken   string `json:"card_token"`
}

// chargeCard is called via ctx.Run — it may execute more than once before settling.
// The idempotency key is derived from the stable order ID, not wall-clock time.
func chargeCard(args ChargeArgs) (string, error) {
    idempotencyKey := "charge:" + args.OrderID // stable across retries

    req, err := http.NewRequest("POST", "https://api.stripe.com/v1/charges", encodeForm(map[string]string{
        "amount":   fmt.Sprintf("%d", args.AmountCents),
        "currency": "usd",
        "source":   args.CardToken,
    }))
    if err != nil {
        return "", err
    }
    req.Header.Set("Idempotency-Key", idempotencyKey)
    // ... execute request, parse response
    return chargeID, nil
}
```

## Full order workflow: reserve → charge → record

Three sequential SoR interactions, each in its own `ctx.Run`. Any step that has already settled short-circuits on replay; no external system is hit twice.

```go
type OrderArgs struct {
    OrderID     string `json:"order_id"`
    SKU         string `json:"sku"`
    Quantity    int    `json:"quantity"`
    AmountCents int64  `json:"amount_cents"`
    CardToken   string `json:"card_token"`
}

type ReserveArgs struct {
    OrderID  string `json:"order_id"`
    SKU      string `json:"sku"`
    Quantity int    `json:"quantity"`
}

type ChargeArgs struct {
    OrderID     string `json:"order_id"`
    AmountCents int64  `json:"amount_cents"`
    CardToken   string `json:"card_token"`
}

// createOrder orchestrates three idempotent SoR steps.
// Each ctx.Run is checkpointed: on replay, settled steps return their stored
// result without touching the external system.
func createOrder(ctx *resonate.Context, args OrderArgs) (string, error) {
    // Step 1 — reserve inventory; idempotent via ON CONFLICT in the DB
    fReserve, err := ctx.Run(reserveInventory, ReserveArgs{
        OrderID:  args.OrderID,
        SKU:      args.SKU,
        Quantity: args.Quantity,
    })
    if err != nil {
        return "", err
    }
    var reservationID string
    if err := fReserve.Await(&reservationID); err != nil {
        return "", err
    }

    // Step 2 — charge payment; idempotent via Stripe idempotency key
    fCharge, err := ctx.Run(chargeCard, ChargeArgs{
        OrderID:     args.OrderID,
        AmountCents: args.AmountCents,
        CardToken:   args.CardToken,
    }, resonate.RunOpts{
        RetryPolicy: resonate.ExponentialRetry{
            MaxAttempts: 5,
            Base:        100 * time.Millisecond,
            Max:         30 * time.Second,
            Jitter:      true,
        },
    })
    if err != nil {
        return "", err
    }
    var chargeID string
    if err := fCharge.Await(&chargeID); err != nil {
        return "", err
    }

    // Step 3 — record the order in the SoR DB; idempotent via ON CONFLICT DO NOTHING
    fRecord, err := ctx.Run(recordOrder, args.OrderID, chargeID)
    if err != nil {
        return "", err
    }
    if err := fRecord.Await(nil); err != nil {
        return "", err
    }

    return args.OrderID, nil
}
```

## Check-then-act made replay-safe

A common SoR pattern is: read current state, branch on it, then apply a mutation. Both the read and the write must be in separate `ctx.Run` calls. Reading the SoR in bare workflow code is unsafe — on replay the bare read fires again and may observe a different value (e.g., another process updated the account between the first run and the replay).

```go
type AccountArgs struct {
    AccountID string `json:"account_id"`
    Delta     int64  `json:"delta"`
}

type AccountState struct {
    Balance int64  `json:"balance"`
    Status  string `json:"status"`
}

// adjustBalance reads account state as a checkpointed step, branches in plain
// Go code (no SoR access here), then applies the mutation in another step.
func adjustBalance(ctx *resonate.Context, args AccountArgs) (int64, error) {
    // Checkpointed read — on replay returns the stored snapshot, not a fresh DB hit
    fState, err := ctx.Run(getAccountState, args.AccountID)
    if err != nil {
        return 0, err
    }
    var state AccountState
    if err := fState.Await(&state); err != nil {
        return 0, err
    }

    // Branch in plain workflow code — no external calls here
    if state.Status == "frozen" {
        return 0, resonate.NewNonRetryable(fmt.Errorf("account %s is frozen", args.AccountID))
    }
    if state.Balance+args.Delta < 0 {
        return 0, resonate.NewNonRetryable(fmt.Errorf("insufficient funds"))
    }

    // Checkpointed write — idempotent conditional update keyed on account ID + delta
    idempotencyKey := fmt.Sprintf("%s:%s:adjust", args.AccountID, ctx.ID())
    fApply, err := ctx.Run(applyBalanceDelta, ApplyArgs{
        AccountID:      args.AccountID,
        Delta:          args.Delta,
        IdempotencyKey: idempotencyKey,
    })
    if err != nil {
        return 0, err
    }
    var newBalance int64
    if err := fApply.Await(&newBalance); err != nil {
        return 0, err
    }
    return newBalance, nil
}
```

`ctx.ID()` is stable across replays, so the derived `idempotencyKey` is identical on every retry — the SoR deduplicates.

## Retries and non-retryable SoR errors

Use `resonate.RunOpts{RetryPolicy: ...}` to control per-step retry behaviour. The default (`nil`) is `ExponentialRetry{MaxAttempts: 3, Base: 100ms, Max: 30s, Jitter: true}`.

Wrap terminal SoR errors (validation failures, 4xx responses that will never succeed) with `resonate.NewNonRetryable` so the retry loop stops immediately:

```go
func reserveInventory(args ReserveArgs) (string, error) {
    // ... call inventory service
    if statusCode == 400 {
        // Bad request — retrying won't help
        return "", resonate.NewNonRetryable(fmt.Errorf("invalid SKU %q", args.SKU))
    }
    if statusCode == 409 {
        // Already reserved under this order ID — idempotent success
        return existingReservationID, nil
    }
    // 5xx — transient, let the retry policy handle it
    return "", fmt.Errorf("inventory service error: %d", statusCode)
}
```

## Distinct Go idioms

- **Explicit `(T, error)` at every layer.** Every leaf and workflow returns two values; there is no `Result<T>` enum or `?` operator. Error paths are explicit `if err != nil` chains.
- **Checkpointed reads via `ctx.Run`.** Go has no dependency-injection mechanism like Rust's `ctx.get_dependency::<T>()`. Pass clients/pools as closed-over values or construct them outside the workflow and reference them in the leaf closure. The checkpoint is the same — the leaf's return value is stored in the durable promise and returned on replay without re-executing.
- **Stable idempotency-key derivation.** Use `ctx.ID()` or a deterministic hash of stable args. `time.Now()` and `math/rand` produce different values across replays — never use them to derive keys passed into a SoR.
- **`resonate.NewNonRetryable` for 4xx SoR errors.** Stops the step retry loop; the caller's `Await` returns the wrapped error immediately.
- **Option structs are the last positional arg.** `ctx.Run(fn, args, resonate.RunOpts{...})` — the options struct must be last; misplacing it passes it as the function argument.

## Avoid

- **Bare external calls in workflow code.** Any DB query, HTTP call, or file write outside a `ctx.Run` re-executes on every replay. Always wrap SoR interactions in a leaf.
- **`time.Now()` or `rand` as idempotency key inputs.** These differ across replays; the SoR sees a different key each time and cannot deduplicate. Derive keys from `ctx.ID()` or stable args.
- **Reading the SoR in bare workflow code.** The read fires again on replay and may observe a changed value. Checkpoint SoR reads in their own `ctx.Run`.
- **Writing to two systems without a clear SoR.** If both fail partially, neither wins. Designate one SoR; make the other a compensatable side-effect. For multi-system compensation, see `resonate-saga-pattern-go`.

## Related skills

- `resonate-basic-durable-world-usage-go` — `ctx.Run`, `ctx.RPC`, `ctx.Sleep`, option structs, `RegisteredFunc.Run`
- `resonate-saga-pattern-go` — when the SoR doesn't cover all steps and you need compensation across multiple services
- `durable-execution` — foundational replay semantics; this pattern is checkpoint-centric
- `resonate-external-system-of-record-pattern-typescript` — same pattern, TS SDK; good for mental model
- `resonate-external-system-of-record-pattern-rust` — same pattern, Rust SDK; type-dispatched DI contrast
