---
name: resonate-saga-pattern-java
description: Implement saga patterns for distributed transactions in Java with Resonate — forward steps wrapped in a try/catch, each tracked in a List, with compensating actions that unwind in reverse on failure. Uses an enum + switch for compensation dispatch and the retry policy (Retry.Never / bounded Exponential) to control which steps retry, since the Java SDK has no non-retryable error wrapper. Use when coordinating multi-step Java workflows that need consistency across failures without a distributed transaction. Verified against develop/java.mdx (docs PR #230) and the resonate-sdk-java source at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Saga Pattern — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** (virtual threads, a feature generally available in Java 21). There is no Java saga example repo yet, so every code block here uses only documented SDK surface, compile-verified against `0.1.1` and cross-checked against `develop/java.mdx` (docs PR #230) and the SDK source.

## Overview

A saga is a long-running transaction split into discrete steps, each with a compensating action. Forward steps execute top-to-bottom; on any failure, completed compensations run bottom-to-top. Because the whole workflow body re-runs on resume, the `completed` list is reconstructed deterministically on replay — each `ctx.run` short-circuits its settled durable promise in the same order, so the list is rebuilt identically before reaching the point of failure.

Java's exception model makes the saga read more like the TypeScript/Python idiom than Go's: the forward path lives in a `try` block, and a single `catch` triggers compensation — there is no per-step `(T, error)` check to thread by hand.

For the language-agnostic mental model — choreography vs orchestration, idempotency requirements, when not to use sagas — see `resonate-saga-pattern-typescript`.

## When to use

- Multi-step workflow where intermediate state is visible to other systems between steps
- Each step is individually compensable (inventory hold, payment charge, shipment create)
- You need "all or nothing" consistency without a distributed 2PC transaction
- Steps span services and may take seconds to minutes each
- Compensation logic exists and is idempotent for every committed step

## Basic shape

```java
package io.resonatehq.examples.saga;

import io.resonatehq.resonate.Context;
import io.resonatehq.resonate.Context.Opts;
import io.resonatehq.resonate.Handle.ResonateHandle;
import io.resonatehq.resonate.Resonate;
import io.resonatehq.resonate.Retry.Exponential;
import java.util.ArrayList;
import java.util.List;

public final class Saga {
    private Saga() {}

    /** The forward steps that can be committed; the enum + switch gives closed-set compensation dispatch. */
    public enum Step { INVENTORY, PAYMENT, SHIPMENT }

    public record SagaResult(String status, String orderId, List<String> compensated) {}

    /**
     * The saga orchestrator. Forward steps run in sequence inside a try; the first failure throws,
     * and the catch runs compensations in reverse. Each ctx.run is a durable checkpoint, so a
     * crash mid-saga resumes from the last settled step — committed steps do not re-execute.
     */
    public static SagaResult placeOrder(Context ctx, String orderId) {
        List<Step> completed = new ArrayList<>();
        try {
            ctx.run(Saga::reserveInventory, orderId).await();
            completed.add(Step.INVENTORY);

            ctx.run(Saga::chargePayment, orderId).await();
            completed.add(Step.PAYMENT);

            ctx.run(Saga::createShipment, orderId).await();
            completed.add(Step.SHIPMENT);

            return new SagaResult("success", orderId, List.of());
        } catch (Exception forwardErr) {
            // await() sneaky-throws the rejection, which may be unchecked OR a sneakily-rethrown
            // checked exception — catch Exception so no failure path slips past the compensator.
            return compensateAll(ctx, orderId, completed);
        }
    }

    /** Iterate completed steps in reverse, running each compensation as its own durable ctx.run. */
    public static SagaResult compensateAll(Context ctx, String orderId, List<Step> completed) {
        List<String> names = new ArrayList<>();
        for (int i = completed.size() - 1; i >= 0; i--) {
            Step step = completed.get(i);
            // A compensation that exhausts retries is a real inconsistency — give it a generous policy.
            ctx.options(new Opts().withRetryPolicy(new Exponential(1, 5, 2, 30)))
                    .run(Saga::compensate, step, orderId)
                    .await();
            names.add(step.name());
        }
        return new SagaResult("failed", orderId, names);
    }

    /** Dispatch to the correct undo action via a switch. Each branch must be idempotent. */
    public static String compensate(Context ctx, Step step, String orderId) {
        return switch (step) {
            case SHIPMENT -> "shipment-cancelled:" + orderId;   // cancelShipment(orderId)
            case PAYMENT -> "payment-refunded:" + orderId;      // refundPayment(orderId)
            case INVENTORY -> "inventory-released:" + orderId;  // releaseInventory(orderId)
        };
    }

    // Forward leaf stubs — replace with real service calls.
    public static String reserveInventory(Context ctx, String orderId) {
        return "reserved:" + orderId;
    }

    public static String chargePayment(Context ctx, String orderId) {
        return "charged:" + orderId;
    }

    public static String createShipment(Context ctx, String orderId) {
        // Simulate a transient failure to trigger compensation.
        throw new IllegalStateException("shipment service unavailable");
    }

    public static void main(String[] args) {
        Resonate r = Resonate.builder().url("http://localhost:8001").build();
        r.register(Saga::placeOrder);
        try {
            ResonateHandle<SagaResult> handle = r.run("order-1", Saga::placeOrder, "order-1");
            System.out.println(handle.result());
        } finally {
            r.stop();
        }
    }
}
```

Each `ctx.run` + `await` pair is a durable checkpoint. If the worker crashes mid-saga, Resonate resumes from the last settled step — completed forward steps do not re-execute.

## Catch `Exception`, not `RuntimeException`

`ResonateFuture.await()` re-raises a rejected promise via a *sneaky throw* (`Context.java`): the underlying error is rethrown without being wrapped, even though `await()` declares no checked exception. In the common single-process path a leaf's failure comes back as a `RuntimeException` (a thrown checked exception is wrapped in `ApplicationError`, itself a `RuntimeException`), so `catch (RuntimeException)` would work. But when a rejection was recorded by a *different* worker that serialized a checked exception into the promise (`Codec` can recover an arbitrary originating exception across the durability boundary), `await()` can surface that checked exception directly. Catch `Exception` (it always compiles in a `try` block — `Exception` and `Throwable` are special-cased) so the saga compensates on *any* forward failure, local or cross-worker, at no cost.

## Compensation must be retryable

A compensation that exhausts retries leaves the system in a real inconsistency. Give compensations a generous timeout and an explicit retry policy:

```java
ctx.options(new Opts()
        .withTimeout(java.time.Duration.ofMinutes(10))
        .withRetryPolicy(new Exponential(1, 5, 2, 30))) // 1s base, doubling, capped 30s, 5 retries
        .run(Saga::compensate, step, orderId)
        .await();
```

If that `await()` throws after retries are exhausted, you have a manual-intervention situation — log the step, the order ID, and alert before rethrowing.

## Controlling which steps retry — there is no `NonRetryable`

Unlike the Go SDK (`resonate.NewNonRetryable`), the Java SDK has **no non-retryable error wrapper**. The error hierarchy (`Errors.ResonateError` and its subclasses) is for catching/matching, not for tagging an error as terminal. Control retry behavior with the **retry policy** instead:

```java
import io.resonatehq.resonate.Retry.Never;

// A forward step against a non-idempotent external API that must NOT be retried:
ctx.options(new Opts().withRetryPolicy(new Never())) // single attempt
        .run(Saga::chargeOnce, orderId)
        .await();
```

`new Retry.Never()` runs the leaf exactly once; any throw propagates immediately to the saga's `catch`. For a fast-fail on a programming error (e.g. an unhandled enum case), let the exception propagate — an exhaustive `switch` expression over the enum needs no `default`, so a newly added step that lacks a compensation is a *compile error*, not a silent runtime loop.

## Distinct Java idioms

- **`try` / `catch` forward path, not per-step `(T, error)` checks.** The whole forward sequence sits in one `try`; a single `catch (Exception forwardErr)` triggers `compensateAll`. This is closer to the TypeScript/Python saga than to Go's explicit error returns.
- **Catch `Exception`, not `RuntimeException`** — `await()` sneaky-throws, so a checked exception can surface; only `Exception` guarantees the compensator runs.
- **`enum Step` + exhaustive `switch` expression** — gives the closed-step-set semantics with no `default` branch; adding a step without a compensation fails to compile.
- **`List<Step> completed` rebuilt on replay** — on crash-resume the body re-runs from the top; each `ctx.run` short-circuits its settled promise in order, so the list is reconstructed identically before the failed step.
- **`for (int i = completed.size() - 1; i >= 0; i--)`** — the straightforward reverse loop for unwinding compensations.
- **`record` args/results** — `record SagaResult(...)` and plain `String` step args are JSON-serializable across the durability boundary.
- **Retry policy controls terminal-vs-retried**, since there is no `NonRetryable` — use `new Retry.Never()` for steps that must not retry.

## Avoid

- **Catching `RuntimeException` instead of `Exception`** — `await()` can sneaky-throw a checked exception past it, skipping compensation.
- **Non-idempotent compensations** — each compensation `ctx.run` may execute more than once on retry. A compensation that double-refunds is worse than the original failure.
- **Reading the system-of-record outside a `ctx.run`** — a raw DB/API call in the workflow body re-executes on every replay. Wrap observable side effects in their own `ctx.run` so the result is checkpointed.
- **Swallowing compensation errors** — if a compensation `await()` throws after retries, surface it. Silently continuing produces a partially compensated saga with no audit trail.
- **Inventing a `NonRetryable` error** — it does not exist in the Java SDK; use `Retry.Never()` or a bounded `maxRetries`.

## Related skills

- `resonate-basic-durable-world-usage-java` — `ctx.run`, `Opts`, retry policies, the replay model
- `resonate-recursive-fan-out-pattern-java` — parallel dispatch, composable with saga steps that fan out internally
- `resonate-external-system-of-record-pattern-java` — idempotency keys and SoR reads inside durable leaves
- `durable-execution` — foundational replay semantics that make the `completed` list safe
- `resonate-saga-pattern-typescript` — language-agnostic mental model, choreography vs orchestration
- `resonate-saga-pattern-python` — the closest sibling; the Java API mirrors Python
