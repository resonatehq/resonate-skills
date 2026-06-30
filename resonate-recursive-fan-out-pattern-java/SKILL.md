---
name: resonate-recursive-fan-out-pattern-java
description: Implement recursive fan-out / parallel workflow execution in Java with Resonate — dispatch all children via ctx.run or ctx.rpc collecting a List of ResonateFuture, then await each. Covers single-workflow fan-out, cross-worker recursion via ctx.rpc to a named worker group, and the worker/client split. Use when processing batches, trees, or graphs where each child is independently durable and optionally recurses. Verified against example-fan-out-fan-in-java and example-recursive-factorial-java and develop/java.mdx (docs PR #230) at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** (virtual threads, a feature generally available in Java 21). Every code block here is compile-verified against `0.1.1` and drawn from the `example-fan-out-fan-in-java` / `example-recursive-factorial-java` repos and `develop/java.mdx` (docs PR #230).

## Overview

Recursive fan-out dispatches multiple child invocations in parallel, awaits them individually, and optionally recurses deeper. The Java expression is a two-loop pattern: a dispatch loop builds a `List<ResonateFuture<T>>`, then a separate await loop reads each result. Mixing dispatch and await serializes the children — the single most common mistake.

For the language-agnostic mental model (promise deduplication, load-balancing, detached vs. result-gathering), see `resonate-recursive-fan-out-pattern-typescript`.

## When to use

- Batch processing where items are independent and each needs durability
- Map-reduce shaped workflows (fan out N workers, aggregate results)
- Recursive tree / graph traversal with dynamic depth
- Any case where a crash mid-fan-out must resume, not restart from zero

## Dispatch-then-await shape

The canonical fan-out from `example-fan-out-fan-in-java` — dispatch all children first, await all second:

```java
import io.resonatehq.resonate.Context;
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

The dispatch loop runs serially in user code, but each `ctx.run` creates an independent durable promise — all children execute concurrently. The await loop only reads results; it does not gate execution of siblings.

**Partial failure.** As written, an exception from any child's `future.await()` propagates and stops the loop (fail-fast). To collect every result instead — recording per-child failures rather than aborting at the first — wrap each `future.await()` in its own `try`/`catch` inside the await loop and append a failure record on the `catch`. Choose fail-fast when one bad child should abort the batch, collect-all when you want a summary of every delivery.

## Recursive fan-out across workers via `ctx.rpc`

A workflow can call `ctx.rpc(NAME, smallerArgs)` to recurse through the server. Each recursive call is its own durable promise; with multiple workers registered under the same name, the recursion fans out across them. From `example-recursive-factorial-java`:

```java
import io.resonatehq.resonate.Context;
import io.resonatehq.resonate.Context.Opts;

public final class Factorial {
    private Factorial() {}

    public static final String NAME = "factorial";
    public static final String WORKER_GROUP = "factorial-workers";

    /**
     * Compute n! by recursively dispatching factorial(n-1) to the worker group via ctx.rpc.
     * Returns long — 13! overflows int. The by-name RPC result is read via Number because Jackson
     * decodes a small JSON integer as Integer and a large one as Long.
     */
    public static long factorial(Context ctx, int n) {
        if (n <= 1) {
            return 1L;
        }
        long sub = ((Number) ctx.options(new Opts().withTarget(WORKER_GROUP))
                .rpc(NAME, n - 1).await()).longValue();
        return (long) n * sub;
    }
}
```

`ctx.rpc(NAME, n - 1)` dispatches by name to the worker group, returning a `ResonateFuture<Object>` — hence the `Number` read. The base case (`if (n <= 1)`) must always terminate the recursion.

## Worker / client split

The recursion runs across processes that register the function under a named group; a client targets that group without registering anything.

Worker — joins the group and registers the function, then blocks (never `stop`):

```java
import io.resonatehq.resonate.Resonate;
import java.util.concurrent.CountDownLatch;

public static void main(String[] args) throws InterruptedException {
    String url = System.getenv().getOrDefault("RESONATE_URL", "http://localhost:8001");
    Resonate r = Resonate.builder().url(url).group(Factorial.WORKER_GROUP).build();
    r.register(Factorial::factorial);
    new CountDownLatch(1).await(); // keep the worker alive to receive tasks
}
```

Client — uses a *distinct* group so it cannot be assigned worker tasks, and dispatches by name to the worker group:

```java
import io.resonatehq.resonate.Context.Opts;
import io.resonatehq.resonate.Handle.ResonateHandle;
import io.resonatehq.resonate.Resonate;

public static void main(String[] args) {
    int n = args.length > 0 ? Integer.parseInt(args[0]) : 6;
    String url = System.getenv().getOrDefault("RESONATE_URL", "http://localhost:8001");

    Resonate r = Resonate.builder().url(url).group("factorial-client").build();
    try {
        String id = "factorial-" + n; // stable id — a second run returns the cached result
        ResonateHandle<Object> handle =
                r.options(new Opts().withTarget(Factorial.WORKER_GROUP)).rpc(id, Factorial.NAME, n);
        long result = ((Number) handle.result()).longValue();
        System.out.printf("factorial(%d) = %d%n", n, result);
    } finally {
        r.stop();
    }
}
```

The client never registers `factorial` — it only dispatches by name to the worker group. Routing through a non-default group keeps clients out of the task pool so only real workers execute the recursion. Start a second worker before invoking and the intermediate steps distribute across both processes.

## `ctx.run` vs `ctx.rpc` for fan-out

| | `ctx.run` | `ctx.rpc` |
|---|---|---|
| Execution target | Same process | Remote process (by name or method reference) |
| Fan-out across workers | No | Yes — server distributes to the named group |
| Recursion across workers | No | Yes — each `ctx.rpc(NAME, ...)` re-dispatches |
| Best for | In-process leaf tasks | Distributed branches, cross-worker recursion |

Both methods return a `ResonateFuture` immediately and both support the two-loop fan-out pattern. `ctx.run` functions must return promptly — long-running or blocking work belongs in `ctx.rpc`.

## Distinct Java idioms

- **`List<ResonateFuture<T>>` collected in the dispatch loop, drained in the await loop** — the two-loop pattern. A single loop that dispatches and awaits in the same iteration serializes children.
- **Typed futures from method references, untyped from by-name `rpc`** — `ctx.run(Owner::send, ...)` gives `ResonateFuture<Delivery>`; `ctx.rpc(NAME, ...)` gives `ResonateFuture<Object>`, so read numerics through `Number`.
- **`group(...)` on the builder for the worker group**, and a *distinct* group for the client — Java sets the group directly on the builder, not on a separate transport.
- **Shared `NAME` / `WORKER_GROUP` constants** in a class imported by both worker and client — prevents string drift between the worker's `group(...)` and the client's `withTarget(...)`.
- **Stable id for idempotent recursion** — `"factorial-" + n` means a second invocation returns the cached result instead of recomputing.
- **`new CountDownLatch(1).await()`** keeps a worker alive; never `r.stop()` a worker.
- **Replay safety is automatic** — the body re-runs on resume, the dispatch loop deterministically re-issues the same calls, and settled children short-circuit. No replay-guard code needed.

## Avoid

- **Awaiting inside the dispatch loop** — `ctx.run`/`ctx.rpc` immediately followed by `await` in the same iteration runs children sequentially. Collect all futures first, then await.
- **Unbounded recursion without a base case** — the server-side promise graph is not free; always guard `ctx.rpc(NAME, ...)` with a termination condition (`if (n <= 1)`).
- **Sharing the default group between client and worker** — a client in the worker group joins the task pool and steals dispatches it cannot fulfill. Give the client its own group.
- **Casting a by-name `rpc` numeric result straight to `Long`** — Jackson may have decoded it as `Integer`. Read through `Number`.
- **Reading a `List<MyRecord>` (or any generic) back from a by-name `rpc`** — the by-name form decodes against `Object`, so Jackson hands back `List<LinkedHashMap>`, not your record type. Use the method-reference form (`ctx.rpc(Owner::fn, ...)` / `ctx.run(Owner::fn, ...)`) for a typed future. See `resonate-basic-debugging-java`.

## Related skills

- `resonate-basic-durable-world-usage-java` — `ctx.run`, `ctx.rpc`, `ResonateFuture.await`, `Opts`
- `resonate-basic-ephemeral-world-usage-java` — the worker/client builder setup, `group(...)`, typed vs untyped handles
- `resonate-saga-pattern-java` — fan-out alongside compensating transactions
- `durable-execution` — foundational replay semantics
- `resonate-recursive-fan-out-pattern-typescript` — canonical mental model, deduplication strategy
- `resonate-recursive-fan-out-pattern-python` — the closest sibling; the Java API mirrors Python
