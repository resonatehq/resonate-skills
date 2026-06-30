---
name: resonate-external-system-of-record-pattern-java
description: Maintain consistency between a Resonate Java workflow and an external system that owns the truth — a database, ledger, or third-party API — without distributed transactions, by wrapping every interaction in its own idempotent ctx.run step so the durable promise records the result and replay never re-fires the external call. Uses type-keyed dependency injection (r.withDependency / ctx.getDependency) to reach the SoR client, and idempotency keys derived from ctx.info().id(). Verified against develop/java.mdx (docs PR #230) and the resonate-sdk-java source at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate External System of Record Pattern — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** (virtual threads, a feature generally available in Java 21). There is no Java system-of-record example repo yet, so every code block here uses only documented SDK surface, compile-verified against `0.1.1` and cross-checked against `develop/java.mdx` (docs PR #230) and the SDK source.

## Overview

For the language-agnostic framing and the "Write Last, Read First" mental model, see `resonate-external-system-of-record-pattern-typescript`. This skill translates that model to idiomatic Java: `ctx.run`-based checkpointing, type-keyed dependency injection for the external client, and idempotency keys derived from the durable execution's stable ID.

The core contract is simple: one system owns the truth. Resonate coordinates the workflow and guarantees at-least-once execution; the external system enforces consistency through its own primitives — idempotency keys, upserts, conditional writes, ledger deduplication.

## When to use

- A Resonate workflow must create, update, or read from a database, payment processor, ledger, or any external store that has its own durability.
- You need multi-step operations (reserve → charge → record) to be safe to retry without double-writes.
- You cannot use distributed transactions across Resonate's promise store and the external system.

Do not use this pattern when all writes live inside a single ACID transaction scope — just use the transaction.

## The replay principle — wrap every SoR call in `ctx.run`

**This is the entire reason the pattern exists.**

Whenever a workflow suspends and resumes (after a `ctx.sleep`, a remote `ctx.rpc`, or a pending `ctx.promise`), the entire workflow body re-executes from the top. Durable child promises short-circuit steps that already settled — the external call is not re-fired, and the stored result is returned directly. But any external call NOT wrapped in a `ctx.run` will execute again on every replay, producing double-writes.

Rule: **every read from or write to the external SoR must be inside its own `ctx.run` leaf.**

```java
// BAD — fires on every replay, may charge the card twice
public static String placeOrder(Context ctx, OrderArgs args) {
    String chargeId = stripe.charge(args.amountCents(), args.cardToken()); // replay fires this again
    return chargeId;
}

// GOOD — checkpointed; on replay the stored chargeId is returned, Stripe is not called again
public static String placeOrder(Context ctx, OrderArgs args) {
    return ctx.run(Orders::chargeCard, args).await();
}
```

## Reaching the SoR client via dependency injection

Unlike Go (which closes the client over the leaf), the Java SDK has real type-keyed DI. Register the client once on the instance with `r.withDependency`, and fetch it inside a leaf by type with `ctx.getDependency`:

```java
import io.resonatehq.resonate.Context;

// In the ephemeral world, before processing starts:
//   r.withDependency(new PaymentGateway(System.getenv("STRIPE_KEY")));

public static String chargeCard(Context ctx, ChargeArgs args) {
    PaymentGateway gateway = ctx.getDependency(PaymentGateway.class);
    // Idempotency key derived from a STABLE value — never time/random.
    String idempotencyKey = "charge:" + args.orderId();
    return gateway.submit(args.amountCents(), args.cardToken(), idempotencyKey);
}
```

Dependencies are keyed by their concrete class — register at most one instance per type. This keeps the SoR client out of the serialized args while still being reachable from any worker that runs the leaf.

## Idempotency keys from `ctx.info().id()`

A `ctx.run` step can re-execute on retry before it settles (Resonate delivers at-least-once). The SoR must be addressed idempotently so a duplicate call is harmless. Derive stable keys from `ctx.info().id()`, `ctx.info().originId()`, or stable workflow args — never from `System.currentTimeMillis()` or `Math.random()`, which differ across replays and defeat deduplication.

```java
public static long applyBalanceDelta(Context ctx, AccountArgs args) {
    Ledger ledger = ctx.getDependency(Ledger.class);
    String idempotencyKey = args.accountId() + ":" + ctx.info().id() + ":adjust";
    return ledger.apply(args.accountId(), args.delta(), idempotencyKey);
}
```

`ctx.info().id()` is stable across replays, so the derived key is identical on every retry — the SoR deduplicates.

## Full order workflow: reserve → charge → record

Three sequential SoR interactions, each in its own `ctx.run`. Any step that already settled short-circuits on replay; no external system is hit twice.

```java
import io.resonatehq.resonate.Context;

public final class Orders {
    private Orders() {}

    public record OrderArgs(String orderId, String sku, int quantity, long amountCents, String cardToken) {}
    public record ReserveArgs(String orderId, String sku, int quantity) {}
    public record ChargeArgs(String orderId, long amountCents, String cardToken) {}
    public record RecordArgs(String orderId, String chargeId) {}

    /** Orchestrates three idempotent SoR steps. Each ctx.run is checkpointed. */
    public static String createOrder(Context ctx, OrderArgs args) {
        // Step 1 — reserve inventory; idempotent via ON CONFLICT in the DB
        String reservationId = ctx.run(Orders::reserveInventory,
                new ReserveArgs(args.orderId(), args.sku(), args.quantity())).await();

        // Step 2 — charge payment; idempotent via the gateway's idempotency key
        String chargeId = ctx.run(Orders::chargeCard,
                new ChargeArgs(args.orderId(), args.amountCents(), args.cardToken())).await();

        // Step 3 — record the order; idempotent via ON CONFLICT (order_id) DO NOTHING
        ctx.run(Orders::recordOrder, new RecordArgs(args.orderId(), chargeId)).await();

        return args.orderId();
    }

    public static String reserveInventory(Context ctx, ReserveArgs args) {
        Inventory inv = ctx.getDependency(Inventory.class);
        return inv.reserve(args.orderId(), args.sku(), args.quantity()); // ON CONFLICT → existing id
    }

    public static String chargeCard(Context ctx, ChargeArgs args) {
        PaymentGateway gateway = ctx.getDependency(PaymentGateway.class);
        return gateway.submit(args.amountCents(), args.cardToken(), "charge:" + args.orderId());
    }

    public static String recordOrder(Context ctx, RecordArgs args) {
        OrderStore store = ctx.getDependency(OrderStore.class);
        store.insertIfAbsent(args.orderId(), args.chargeId()); // ON CONFLICT (order_id) DO NOTHING
        return "recorded";
    }
}
```

## Check-then-act made replay-safe

A common SoR pattern is: read current state, branch on it, then apply a mutation. Both the read and the write must be in separate `ctx.run` calls. Reading the SoR in bare workflow code is unsafe — on replay the bare read fires again and may observe a different value.

```java
public static long adjustBalance(Context ctx, AccountArgs args) {
    // Checkpointed read — on replay returns the stored snapshot, not a fresh DB hit.
    AccountState state = ctx.run(Orders::getAccountState, args.accountId()).await();

    // Branch in plain workflow code — no external calls here.
    if (state.status().equals("frozen")) {
        throw new IllegalStateException("account " + args.accountId() + " is frozen");
    }
    if (state.balance() + args.delta() < 0) {
        throw new IllegalStateException("insufficient funds");
    }

    // Checkpointed write — idempotent conditional update keyed on a stable value.
    return ctx.run(Orders::applyBalanceDelta, args).await();
}
```

Because the read is checkpointed, a replay sees the snapshot the workflow already acted on, not a value another process changed in the meantime.

## Retries and terminal SoR errors

Per-step retry behavior is set with `ctx.options(new Opts().withRetryPolicy(...))`. The Java SDK has **no non-retryable error wrapper** (unlike Go's `NewNonRetryable`) — control retries with the policy. For a 4xx that will never succeed, run the leaf under `new Retry.Never()` so a thrown error propagates immediately instead of being retried:

```java
import io.resonatehq.resonate.Context.Opts;
import io.resonatehq.resonate.Retry.Exponential;
import io.resonatehq.resonate.Retry.Never;

// Transient-tolerant step — retry with backoff:
ctx.options(new Opts().withRetryPolicy(new Exponential(1, 5, 2, 30)))
        .run(Orders::chargeCard, charge).await();

// Validation step that must not be retried on a bad request:
ctx.options(new Opts().withRetryPolicy(new Never()))
        .run(Orders::validateSku, sku).await();
```

Inside the leaf, treat an idempotent-success response (e.g. a `409 Already Reserved` under the same key) as success and return the existing ID rather than throwing.

## Distinct Java idioms

- **Type-keyed DI via `ctx.getDependency(Foo.class)`** — register once with `r.withDependency(new Foo())`. This is the idiomatic way to reach a DB pool / HTTP client / gateway inside a leaf; you do not close it over the workflow (the Go approach) or pass it as a serialized arg.
- **`record` args bundle the leaf's inputs** — `ChargeArgs`, `ReserveArgs`, etc. keep each `ctx.run` to a single serializable value and document the boundary.
- **Stable idempotency keys from `ctx.info().id()`** — stable across replays. `System.currentTimeMillis()` / `Math.random()` / `UUID.randomUUID()` differ across replays and defeat deduplication.
- **Retry policy controls terminal-vs-retried**, since there is no `NonRetryable` — use `new Retry.Never()` for 4xx-style errors that should propagate immediately.
- **Branch in plain Java between checkpointed steps** — the read and write are each a `ctx.run`; the `if` logic between them is pure and replay-safe.

## Avoid

- **Bare external calls in workflow code.** Any DB query, HTTP call, or file write outside a `ctx.run` re-executes on every replay. Always wrap SoR interactions in a leaf.
- **`System.currentTimeMillis()` / `Math.random()` / `UUID.randomUUID()` as idempotency-key inputs.** They differ across replays; the SoR sees a different key each time and cannot deduplicate. Derive keys from `ctx.info().id()` or stable args.
- **Reading the SoR in bare workflow code.** The read fires again on replay and may observe a changed value. Checkpoint SoR reads in their own `ctx.run`.
- **Passing the SoR client as a workflow arg.** It is not JSON-serializable. Inject it with `r.withDependency` / `ctx.getDependency` instead.
- **Writing to two systems without a clear SoR.** Designate one SoR; make the other a compensatable side effect. For multi-system compensation, see `resonate-saga-pattern-java`.

## Related skills

- `resonate-basic-durable-world-usage-java` — `ctx.run`, `Opts`, `ctx.getDependency`, `ctx.info`, the replay model
- `resonate-saga-pattern-java` — when the SoR doesn't cover all steps and you need compensation across services
- `durable-execution` — foundational replay semantics; this pattern is checkpoint-centric
- `resonate-external-system-of-record-pattern-typescript` — language-agnostic mental model
- `resonate-external-system-of-record-pattern-python` — the closest sibling; the Java DI API mirrors Python
