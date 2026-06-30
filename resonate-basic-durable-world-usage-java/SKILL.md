---
name: resonate-basic-durable-world-usage-java
description: Core patterns for writing Resonate durable functions in Java — the Context first-parameter signature, Context APIs (ctx.run, ctx.rpc, ctx.sleep, ctx.promise, ctx.detached, ResonateFuture.await), per-call options via ctx.options(new Opts()), context accessors (ctx.info), type-keyed dependency injection (ctx.getDependency), retry policies, dispatch-then-await fan-out, and the replay model. Requires Java 21+ (virtual threads). Verified against the resonatehq-examples/*-java repos and develop/java.mdx (docs PR #230) at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Basic Durable World Usage — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** — the SDK uses virtual threads, a feature generally available in Java 21. Every code block here is compile-verified against `0.1.1` with a Java 21 toolchain, and cross-checked against the `resonatehq-examples/*-java` repos and `develop/java.mdx` (docs PR #230).

## Overview

A durable function in Java is an ordinary `static` method whose **first parameter is a `Context`**, followed by up to five JSON-serializable application arguments. Every Context method returns a `ResonateFuture` right away — the future object is allocated immediately, while the durable promise behind it is created on the server (inline in local mode, a network round-trip against a real server). You call `future.await()` to read the result. If the awaited promise is still pending, `await` suspends the workflow and re-enters it when the promise settles — and suspension is durable, so any worker (including this one after a restart) can resume the continuation.

This skill covers the Context API surface used **inside** durable functions. The ephemeral-world counterpart (the builder, `register`, top-level `run`/`rpc`/`get`) lives in `resonate-basic-ephemeral-world-usage-java`.

## When to use

Use this skill whenever you are writing the body of a Java method that will be registered with `r.register` or handed as a method reference to `ctx.run`. The moment you touch `ctx.run`, `ctx.rpc`, `ctx.sleep`, `ctx.promise`, or `ctx.detached`, this skill applies.

## Function shape

A durable function takes a `Context` first, then up to five args. Both leaf functions and workflows share the shape — a "workflow" is just a function that itself performs durable ops:

```java
import io.resonatehq.resonate.Context;

public final class Greeter {
    private Greeter() {}

    // A leaf: no durable op, just computes a value.
    public static String formatGreeting(Context ctx, String name) {
        return "hello, " + name + "!";
    }

    // A workflow: orchestrates sub-tasks via the Context.
    public static String greetWorkflow(Context ctx, String name) {
        return ctx.run(Greeter::formatGreeting, name).await();
    }
}
```

Functions are always supplied as **method references** (`Owner::fn`). The reference's functional-interface arity (`Fn.F0` through `Fn.F5`) bounds a durable function to at most five arguments beyond the `Context`. Argument and return types must be JSON-serializable — they cross the durability boundary so the workflow can resume on any worker.

## `ctx.run` — same-process invocation

`ctx.run` dispatches a function in the current process and returns a `ResonateFuture` immediately. Awaiting right after is sequential:

```java
public static String myWorkflow(Context ctx, String input) {
    return ctx.run(Greeter::formatGreeting, input).await();
}
```

To run in parallel, dispatch every child first, then await each future (see Fan-out below).

**`ctx.run` functions must return promptly.** The runtime drains every `ctx.run`-spawned child future before the workflow can suspend or settle. A function that blocks indefinitely holds the task lease open until it expires. Long-running or external-blocking work belongs in `ctx.rpc` (remote dispatch) or `ctx.promise` (latent durable promise), not `ctx.run`.

## Fan-out / fan-in

Dispatch ALL children first (collect futures), then await each. Awaiting inside the dispatch loop serializes them — never what you want for a fan-out. From `example-fan-out-fan-in-java`:

```java
import io.resonatehq.resonate.Context.ResonateFuture;
import java.util.ArrayList;
import java.util.List;

public record Delivery(String channel, boolean ok) {}

public static List<Delivery> fanout(Context ctx, List<String> channels, String message) {
    // Fan out: spawn a child per channel, collecting futures without awaiting yet.
    List<ResonateFuture<Delivery>> futures = new ArrayList<>();
    for (String channel : channels) {
        futures.add(ctx.run(FanOutFanIn::send, channel, message));
    }
    // Fan in: await every child and aggregate.
    List<Delivery> delivered = new ArrayList<>();
    for (ResonateFuture<Delivery> future : futures) {
        delivered.add(future.await());
    }
    return delivered;
}

public static Delivery send(Context ctx, String channel, String message) {
    return new Delivery(channel, true);
}
```

The dispatch loop runs in series in user code, but each `ctx.run` creates an independent durable promise — the children execute concurrently. The await loop only reads results. The same pattern applies to `ctx.rpc`.

## `ctx.rpc` — remote-process invocation

`ctx.rpc` dispatches a registered function by method reference or by name to a remote process. `await` suspends the workflow until the remote function returns.

```java
public static String myWorkflow(Context ctx, String input) {
    return (String) ctx.rpc("remote-func", input).await();
}
```

The by-name form returns a `ResonateFuture<Object>` (hence the cast) and does **not** require the target to be registered locally — it dispatches to a function registered on the executing group. The method-reference form requires a local registration (it recovers name and version from the registry) and gives a typed future. `ctx.rpc` takes per-call options (e.g. `target`) the same way `ctx.run` does.

## `ctx.sleep` — durable sleep

`ctx.sleep` takes a `java.time.Duration` and returns a future with no value to decode. There is no upper bound; sleeps survive process restarts.

```java
import java.time.Duration;

public static void reminder(Context ctx) {
    ctx.sleep(Duration.ofHours(1)).await(); // no value to decode
    // ... send the reminder
}
```

## `ctx.promise` — latent durable promise

`ctx.promise()` creates a durable promise no registered function backs — it settles only when an external actor (webhook, human, CLI, or `r.promises.resolve`) settles it by ID. `await` suspends until then.

```java
import io.resonatehq.resonate.Context.ResonateFuture;

public static String approval(Context ctx) {
    ResonateFuture<Object> promise = ctx.promise(); // 1-day default timeout, capped at the workflow deadline
    String approvalId = promise.id();
    // hand approvalId to whoever resolves it
    Object decision = promise.await(); // suspends until settled externally
    return decision.toString();
}
```

`ctx.promise(Duration)` sets an explicit timeout; `ctx.promise()` uses a 1-day default capped at the parent's remaining deadline. For full external-resolution mechanics — including `r.promises.resolve(id, new Value(...))` — see `resonate-human-in-the-loop-pattern-java`.

## `ctx.detached` — fire-and-forget remote dispatch

`ctx.detached` dispatches a remote function without suspending the parent. It returns only the new promise's ID; the detached execution's lifecycle is independent of the parent.

```java
public static void placeOrder(Context ctx, String orderId) {
    String auditId = ctx.detached("writeAuditLog", orderId).id();
    // auditId references the detached execution; the parent does not await it
}
```

**`ctx.detached` is by-name `String` only** — unlike `ctx.run` and `ctx.rpc`, which accept either a method reference or a `String`, there is no method-reference overload (verified `Context.java:600`). The detached target must be registered on whichever group executes it.

## Per-call options

`run` / `rpc` (on the instance and on the Context) take options from a fresh handle minted by `options(...)`, carrying a `Context.Opts` value. `Opts` is an immutable record with builder-style `with*` methods — override one field, inherit the rest:

```java
import io.resonatehq.resonate.Context.Opts;
import io.resonatehq.resonate.Retry.Constant;

public static String checkout(Context ctx) {
    return ctx.options(new Opts().withRetryPolicy(new Constant(5, 0)))
            .run(Greeter::formatGreeting, "world")
            .await();
}
```

| `Opts` field | Method | Purpose |
|---|---|---|
| `timeout` | `withTimeout(Duration)` | Caps the promise deadline (bounded by the parent's remaining deadline). |
| `target` | `withTarget(String)` | Logical routing address for `rpc`. Empty falls back to the configured group. |
| `version` | `withVersion(int)` | Selects the registered version for by-name dispatch. |
| `retryPolicy` | `withRetryPolicy(RetryPolicy)` | Per-call retry policy for a `ctx.run` leaf. |

The options handle is a fresh instance — the originating `Context` is untouched, so held references never interfere.

## Context accessors

Inside a workflow, `ctx.info()` exposes read-only metadata about the current execution:

```java
import io.resonatehq.resonate.Types.Info;

public static void inspect(Context ctx) {
    Info info = ctx.info();
    String id = info.id();             // this execution's promise ID
    String parent = info.parentId();   // parent promise ID
    String origin = info.originId();   // root workflow ID — stable across the whole tree
    String fn = info.funcName();       // registered function name
    long deadline = info.timeoutAt();  // promise deadline, epoch milliseconds
    // info.branchId() and info.tags() are also available
}
```

`ctx.info().originId()` is stable across the whole call tree — useful for distributed tracing.

## Dependency injection

Register application dependencies (database handles, HTTP clients, gateways) on the instance with `withDependency` and fetch them inside a function by type with `ctx.getDependency`:

```java
r.withDependency(new PaymentGateway());

public static String charge(Context ctx, int amount) {
    PaymentGateway gateway = ctx.getDependency(PaymentGateway.class);
    return gateway.submit(amount);
}
```

Dependencies are keyed by their concrete class — register at most one instance per type, before processing starts. This is real type-keyed DI; you do not have to close clients over the workflow.

## Retries

When a function passed to `ctx.run` (or a top-level `run`) returns by throwing, Resonate re-executes it according to a retry policy. **Retries apply only to leaf functions** — a function that itself performs a durable op (`ctx.run` / `ctx.rpc` / `ctx.sleep` / `ctx.promise`) is a workflow, recovered by replay, and is never retried.

The SDK ships four policies, all implementing `Retry.RetryPolicy` (delays are in **seconds**):

| Policy | Behavior |
|---|---|
| `new Retry.Exponential(delay, maxRetries, factor, maxDelay)` | Delay `delay * factor^attempt`, capped at `maxDelay`, for `maxRetries` retries. |
| `new Retry.Linear(maxRetries, delay)` | Attempt N waits `delay * N`. |
| `new Retry.Constant(maxRetries, delay)` | Fixed `delay` between attempts. |
| `new Retry.Never()` | A single attempt (no retries). |

```java
import io.resonatehq.resonate.Context.Opts;
import io.resonatehq.resonate.Retry.Exponential;

public static String chargeStep(Context ctx, int amount) {
    return ctx.options(new Opts().withRetryPolicy(
                    new Exponential(1, 5, 2, 30))) // 1s base, doubling, capped at 30s, 5 retries
            .run(Greeter::formatGreeting, "card")
            .await();
}
```

A retry policy can be set three ways, in increasing specificity: SDK-wide (`Resonate.builder().retryPolicy(...)`), per-function (`register(ref, name, version, retryPolicy)`), or per-call (`ctx.options(new Opts().withRetryPolicy(...)).run(...)`). The SDK-wide default is `new Exponential(1, 30, 2, Long.MAX_VALUE)` — 1-second base, 30 retries, doubling, with an effectively unbounded `maxDelay` cap.

## The replay model

This is the most important section — read it before writing your first workflow.

Whenever a workflow suspends and resumes — after a `ctx.sleep`, a `ctx.rpc` await, or a pending `ctx.promise` await — **the entire workflow body re-runs from the top.** Resonate short-circuits already-settled child promises (their stored results replay without re-executing the function), but any code that runs *before* reaching a settled durable boundary executes again on every resume.

**Consequence: side effects before a durable boundary run more than once.**

```java
// WRONG — the println and the DB write re-run on every resume.
public static String badWorkflow(Context ctx, String id) {
    System.out.println("starting order"); // runs on every replay
    db.insert(id);                         // inserts a duplicate row on replay!
    ctx.sleep(Duration.ofHours(1)).await();
    return "done";
}

// CORRECT — the side effect is wrapped in ctx.run, so it is checkpointed:
// on resume the settled promise short-circuits and insertOrder does not re-run.
public static String goodWorkflow(Context ctx, String id) {
    ctx.run(MyWorkflow::insertOrder, id).await(); // checkpointed write
    ctx.sleep(Duration.ofHours(1)).await();
    return "done";
}
```

Rules of thumb:
- DB writes, payment calls, emails, and any observable I/O must live inside a `ctx.run` or `ctx.rpc` so the result is checkpointed.
- Pure computation (parsing, formatting, record construction) before a durable boundary is fine — it is idempotent.
- The Resonate instance (`r`) cannot be used inside durable functions; the Context (`ctx`) cannot be used in the ephemeral world.

## Distinct Java idioms

- **Method references, not strings, at typed call sites** — `ctx.run(Owner::fn, args)` gives a typed `ResonateFuture<R>`. The by-name `ctx.rpc("name", args)` form returns `ResonateFuture<Object>`.
- **Immutable `Opts` record with `with*` methods** — `new Opts().withRetryPolicy(...).withTarget(...)`. Each `with*` returns a fresh record; chain them. Pass via `ctx.options(opts).run(...)`.
- **`java.time.Duration` for time** — `Duration.ofHours(1)`, `Duration.ofSeconds(5)`. Retry-policy delays, by contrast, are plain `long` **seconds**.
- **`ctx.sleep(...).await()` returns nothing** — there is no value to decode; just `await()` to suspend.
- **`record` types for args and results** — `public record Delivery(String channel, boolean ok) {}` is the idiomatic JSON-serializable payload.
- **Type-keyed DI via `ctx.getDependency(Foo.class)`** — no need to close clients over the workflow; register once with `r.withDependency(new Foo())`.
- **At most five args beyond `Context`** — `Fn.F0`–`Fn.F5`. Bundle more into a record.

## Avoid

- Placing observable side effects (DB writes, emails, charges) outside a `ctx.run` / `ctx.rpc` — they re-execute on every replay.
- Branching on `System.currentTimeMillis()`, `Math.random()`, or `UUID.randomUUID()` directly in a workflow body — non-deterministic values differ across replay passes and diverge execution. Compute them inside a leaf so the result is checkpointed.
- Awaiting inside a fan-out dispatch loop — this serializes what should be concurrent. Collect all futures, then await.
- Using a method reference with `ctx.detached` — it is by-name `String` only.
- Long-blocking work inside `ctx.run` — it holds the task lease open until the TTL expires. Use `ctx.rpc` or `ctx.promise`.
- Calling the Resonate instance (`r`) from inside a durable function — use Context APIs only.

## Related skills

- `resonate-basic-ephemeral-world-usage-java` — the builder, `register`, top-level `run`/`rpc`/`get`, `promises` / `schedules` sub-clients, `stop` semantics
- `resonate-recursive-fan-out-pattern-java` — dynamic-depth tree fan-outs, cross-worker recursion via `ctx.rpc`
- `resonate-human-in-the-loop-pattern-java` — full `ctx.promise` resolution mechanics via `r.promises.resolve`
- `resonate-durable-sleep-scheduled-work-java` — recurring work loops, `ctx.sleep`, the `schedules` sub-client
- `durable-execution` — foundational concepts: checkpointing, replay, the ephemeral/durable split
- `resonate-defaults` — full defaults table across all SDKs with source citations
- `resonate-basic-durable-world-usage-python` — the closest sibling; the Java Context API mirrors Python
