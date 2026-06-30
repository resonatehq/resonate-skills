---
name: resonate-basic-debugging-java
description: Debug and troubleshoot Resonate applications using the Java SDK. Use when investigating stuck or never-resuming workflows, duplicated side effects after replay, untyped-handle decode surprises (Integer vs Long), CLI positional-argument arity mismatches, the detached by-name-only constraint, Java 21 / virtual-thread requirements, rejected-promise error handling, or r.stop() silently killing a live worker. Verified against the resonatehq-examples/*-java repos and develop/java.mdx (docs PR #230) at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Basic Debugging — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** — the SDK uses virtual threads, a feature generally available in Java 21. Every code block here is compile-verified against `0.1.1` with a Java 21 toolchain, and cross-checked against the `resonatehq-examples/*-java` repos and `develop/java.mdx` (docs PR #230).

## Overview

Java's type system catches many bugs at compile time — typed method references and a typed `ResonateHandle` close off whole classes of error the dynamically-typed SDKs hit. The traps that remain are mostly at the durability boundary: untyped-handle decoding, CLI argument binding, replay double-fires, and the by-name-only constraints. This skill is a symptom-first guide to those failure modes.

For the language-agnostic replay and recovery mental model, read `durable-execution` first.

## Triage flow

1. Does it even build / start? Confirm a **Java 21+** toolchain — the SDK uses virtual threads and will not run on Java 8/11/17.
2. Is the worker connected? Confirm the builder's `url(...)` (or `RESONATE_URL`) points at a running server, and `r` was built without throwing.
3. Is the function registered? By-name dispatch (`rpc`, the CLI) needs `r.register(Owner::fn)` on the executing group.
4. Is the promise stuck? Run `resonate promise get <id>` to check state (`pending` / `resolved` / `rejected` / `timedout`).
5. Is the workflow replaying but producing duplicates? An un-checkpointed side effect is re-running above a durable boundary.
6. Is a CLI-invoked function arity-mismatching? Each `--arg` binds to one positional parameter.
7. Is the worker up but not picking up work? `r.stop()` may have been called on a live worker.

---

## Stuck / never-resuming workflows

### Latent promise never settled

**Symptom:** `future.await()` blocks indefinitely; `resonate promise get <id>` shows state `pending`.

**Causes:**
- The external actor never settled the promise. Resolve it via the CLI (`resonate promise resolve <id> --data '"approved"'`) or `r.promises.resolve(id, new Value(null, "approved"))`.
- The decoded type does not match what the awaiter expects, so `await` throws a decode error rather than returning.

Unlike the Go SDK (which has no `promises` sub-client and needs a manual base64 dance), the **Java SDK ships `r.promises`** — resolve cleanly:

```java
import io.resonatehq.resonate.Types.Value;

r.promises.resolve("approval-1", new Value(null, "approved")).join();
```

See `resonate-human-in-the-loop-pattern-java` for the full mechanics.

### `ctx.run` leaf blocks indefinitely

**Symptom:** the workflow task lease expires and the server reassigns the task; the workflow appears to restart rather than resume; attempts never complete.

**Cause:** the runtime drains every `ctx.run`-spawned future before it can suspend or settle the parent task. A function that does external I/O, waits on a lock, or sleeps for a long time inside `ctx.run` holds the lease open until the TTL expires (default 60s).

**Fix:** move long-running or external-blocking work into `ctx.rpc` (remote dispatch — the workflow suspends cleanly) or `ctx.promise` (latent promise settled by an external actor). Reserve `ctx.run` for in-process computation that returns promptly.

---

## Duplicated side effects (replay)

**Symptom:** emails, charges, log entries, or DB writes happen more than once per logical invocation.

**Cause:** the entire workflow body re-runs from the top on every resume. Durable child promises short-circuit work that already settled, but code that runs *before* reaching a durable boundary (`ctx.sleep`, `ctx.rpc`, `ctx.promise`) executes again on each replay pass.

```java
// BAD — the log line re-executes on every replay pass.
public static String myWorkflow(Context ctx, String id) {
    System.out.println("charging card for order " + id); // runs on every replay
    return ctx.run(MyWorkflow::chargeCard, id).await();
}

// GOOD — the side effect is inside a checkpointed ctx.run; it runs once.
public static String myWorkflow(Context ctx, String id) {
    return ctx.run(MyWorkflow::chargeCard, id).await(); // result is checkpointed
}
```

**Rule:** any observable side effect (network call, write, notification) belongs inside its own `ctx.run` or `ctx.rpc` so the durable promise records the result and short-circuits on replay.

---

## Decode and error handling

### Untyped handle: `Integer` vs `Long`

**Symptom:** `ClassCastException` reading a numeric result from an untyped handle, e.g. `(Long) handle.result()` throws on a small number.

**Cause:** the by-name `r.run` / `r.rpc` forms and `r.get` return `ResonateHandle<Object>`. Jackson decodes a small JSON integer as `Integer` and a large one as `Long`. A direct cast to one or the other is fragile.

**Fix:** read through `Number` (this is exactly what `example-recursive-factorial-java` does, because `13!` overflows `int`):

```java
long result = ((Number) handle.result()).longValue();
```

A method-reference invocation returns a typed `ResonateHandle<R>` and avoids the issue entirely — prefer it where the function is registered locally.

### CLI invocation arity mismatch

**Symptom:** invoking from the CLI fails to bind arguments, or the function receives the wrong values / an arity error.

**Cause:** `resonate invoke <id> --func f --arg 5 --arg 1` passes the args as a positional list, and the Java SDK binds them **one per declared parameter**. A function declaring a single `int[]` (or `List`) parameter arity-mismatches two `--arg` values.

**Fix:** declare one parameter per `--arg`:

```java
// resonate invoke countdown.1 --func countdown --arg 5 --arg 1  →  count=5, delaySeconds=1
public static String countdown(Context ctx, int count, int delaySeconds) { ... }
```

`example-quickstart-java` is the canonical reference. (This is the opposite of Go, where the same invoke binds the whole list to a single `[]int`.)

### Rejected promise re-throws the application error

**Symptom:** `handle.result()` or `future.await()` throws even though no local exception was raised at the call site.

**Cause:** the promise was rejected — either by the registered function throwing, or by an external `resonate promise reject <id>`. The error is re-thrown by its real type where reconstructable across the durability boundary, otherwise as an `ApplicationError` carrying the message.

```java
import io.resonatehq.resonate.Errors.ApplicationError;

try {
    String result = handle.result();
} catch (ApplicationError ae) {
    System.err.println("workflow rejected: " + ae.getMessage());
}
```

### `ctx.detached` with a method reference does not compile

**Symptom:** `ctx.detached(Owner::fn, args)` fails to compile.

**Cause:** `ctx.detached` is **by-name `String` only** — there is no method-reference overload (verified `Context.java:600`), unlike `ctx.run` / `ctx.rpc`.

**Fix:** pass the registered name as a `String`, and ensure the target is registered on whichever group executes it:

```java
String auditId = ctx.detached("writeAuditLog", orderId).id();
```

---

## Setup footguns

### Wrong Java version

**Symptom:** `UnsupportedClassVersionError`, or virtual-thread APIs missing at runtime.

**Cause:** the SDK requires Java 21 (virtual threads). Java 8/11/17 will not work.

**Fix:** set the toolchain to 21:

```kotlin
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
```

### `r.stop()` on a long-running worker

**Symptom:** the worker process is running and healthy-looking, but stops picking up new tasks.

**Cause:** `r.stop()` closes the server connection, stops the heartbeat loop, and cancels the subscription-refresh loop. In-flight leased tasks have their TTL expire and the server reassigns them. The process keeps running, but the dispatch pipeline is dead.

**Rule:** call `r.stop()` only in one-shot binaries, demos, and CI tasks that exit after their work finishes. A long-running worker should stay up — block on a latch and let SIGINT / SIGTERM end the process:

```java
import java.util.concurrent.CountDownLatch;

// Correct for a worker: register, then block — never stop.
r.register(Quickstart::countdown);
new CountDownLatch(1).await();
```

---

## Inspection tools

```shell
resonate dev                                      # local dev server
resonate promise get <id>                         # single promise state + value
resonate promise search 'order:*'                 # prefix search across promises
resonate promise resolve <id> --data '"approved"' # settle a pending latent promise
resonate tree <id>                                # call graph for an invocation
```

See the `resonate-cli` skill for the full command surface. The CLI is SDK-agnostic; the same commands work against any worker language.

**Durable sleep tolerance:** a 24h `ctx.sleep` firing in 23–25h is within the server's timer tolerance window, not a bug.

---

## Avoid

- Branching on `System.currentTimeMillis()`, `Math.random()`, or `UUID.randomUUID()` directly inside a workflow body — non-deterministic values change between replay passes and cause divergent execution. Move them into a leaf so the result is checkpointed.
- Casting a numeric result from an untyped `ResonateHandle<Object>` straight to `Long` (or `Integer`) — read through `Number`.
- Declaring a single array/`List` parameter for a CLI-invoked function — `--arg` values bind positionally, one per parameter.
- Passing more than five application arguments to a durable function — `Fn.F0`–`Fn.F5` caps it at five beyond the `Context`. Bundle extras into a record.
- Passing non-serializable types (open streams, thread handles) as workflow args — they encode into the durable promise via JSON.

---

## Related skills

- `resonate-basic-durable-world-usage-java` — Context APIs (`ctx.run`, `ctx.rpc`, `ctx.sleep`, `ctx.promise`, `ctx.detached`), the replay model
- `resonate-basic-ephemeral-world-usage-java` — the builder, `register`, typed vs untyped handles, `stop` semantics
- `resonate-human-in-the-loop-pattern-java` — latent-promise resolution via `r.promises.resolve`
- `resonate-cli` — full CLI command surface for promise inspection and settlement
- `resonate-defaults` — default TTL, retry policy, and timeout values across all SDKs
- `durable-execution` — foundational replay and recovery model
- `resonate-basic-debugging-python` — the closest sibling; the Java API mirrors Python
