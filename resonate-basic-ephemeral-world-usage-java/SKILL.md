---
name: resonate-basic-ephemeral-world-usage-java
description: Use when writing Java code in main(), an HTTP handler, or any entry point that launches or coordinates Resonate workflows from the ephemeral world (the Client API surface, not code inside a durable function). Core patterns for the Resonate Java SDK's Client APIs — building a Resonate instance with the fluent builder (or new Resonate() for local mode), registering durable functions via method references, invoking them top-level with r.run, dispatching remotely with r.rpc, reconnecting to an existing execution with r.get, reading typed vs untyped handles, the promises and schedules sub-clients, per-call options, and stop semantics. Requires Java 21+ (virtual threads). Verified against example-hello-world-java / example-countdown-java / example-recursive-factorial-java and develop/java.mdx (docs PR #230) at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Basic Ephemeral World Usage — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1` (the only release, confirmed against `maven-metadata.xml`). The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** — the SDK uses virtual threads, a feature generally available in Java 21; Java 8/11/17 will not work. Every code block here is compile-verified against `0.1.1` with a Java 21 toolchain, and cross-checked against the `resonatehq-examples/*-java` repos and `develop/java.mdx` (docs PR #230).

## Overview

The ephemeral world is wherever your Java program starts: `main()`, an HTTP handler, a CLI entry point, a background thread. You use a `Resonate` instance to register durable functions and invoke them top-level. Once an invocation starts, it crosses into the durable world and Resonate guarantees its completion — retries on failure, resumes after crashes, continues across process restarts.

This skill covers the Client API surface only. The Durable World (Context APIs inside workflow functions) is covered in `resonate-basic-durable-world-usage-java`.

## Install

The build tool pulls the SDK from Maven Central — there is nothing to install separately.

```kotlin
// build.gradle.kts
plugins {
    application // lets you run with ./gradlew run
}
dependencies {
    implementation("io.resonatehq:resonate-sdk-java:0.1.1")
}
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
application { mainClass = "io.resonatehq.examples.Main" }
```

Maven is equivalent — add the dependency to `pom.xml` (and target Java 21 in the compiler plugin):

```xml
<dependency>
    <groupId>io.resonatehq</groupId>
    <artifactId>resonate-sdk-java</artifactId>
    <version>0.1.1</version>
</dependency>
```

Run a program with `./gradlew run` (Gradle) or `mvn compile exec:java -Dexec.mainClass=...` (Maven). A one-shot program exits when `main` returns; a worker blocks on a `CountDownLatch` (see [`Resonate.stop`](#resonatestop)).

The SDK lives under a single package; import the types you need:

```java
import io.resonatehq.resonate.Resonate;
import io.resonatehq.resonate.Context;
import io.resonatehq.resonate.Handle.ResonateHandle;
```

The SDK pulls in Jackson (`com.fasterxml.jackson.core:jackson-databind`) transitively for the JSON durability boundary — no extra dependency to declare.

## Basic shape

A complete one-file program: register, run, read, stop.

```java
package io.resonatehq.examples.helloworld;

import io.resonatehq.resonate.Context;
import io.resonatehq.resonate.Handle.ResonateHandle;
import io.resonatehq.resonate.Resonate;

public final class HelloWorld {
    private HelloWorld() {}

    public static String greet(Context ctx, String name) {
        return "hello, " + name + "!";
    }

    public static void main(String[] args) {
        String url = System.getenv().getOrDefault("RESONATE_URL", "http://localhost:8001");
        Resonate r = Resonate.builder().url(url).build();
        r.register(HelloWorld::greet);
        try {
            String id = "hello-" + System.nanoTime();
            ResonateHandle<String> handle = r.run(id, HelloWorld::greet, "world");
            System.out.println(handle.result()); // hello, world!
        } finally {
            r.stop();
        }
    }
}
```

(This is `example-hello-world-java` verbatim.)

## Initialization

### Remote server

The instance is built with a fluent builder. `url(...)` builds a default HTTP transport:

```java
Resonate r = Resonate.builder()
        .url("http://localhost:8001")
        .build();
```

The builder selects its network by precedence: `url(...)` → `network(...)` → the `RESONATE_URL` (or `RESONATE_HOST`/`RESONATE_PORT`/`RESONATE_SCHEME`) environment variable → an in-process `LocalNetwork`. The full builder surface:

| Builder method | Default | Purpose |
|---|---|---|
| `url(String)` | _unset_ | Default HTTP transport at this address. Takes precedence over `network` and `RESONATE_URL`. |
| `network(Network)` | _auto_ | An explicitly constructed transport. Used when `url` is empty. |
| `group(String)` | `"default"` | The routing group this instance subscribes to (the anycast target other workers dispatch to). |
| `token(String)` | `RESONATE_TOKEN` | Sent as Bearer auth on every protocol request. |
| `ttl(Duration)` | `60s` | Per-task lease duration. |
| `prefix(String)` | `RESONATE_PREFIX` | Prepended (with `:`) to every promise/task ID. |
| `pid(String)` | _auto_ | Worker process ID for the lease/heartbeat endpoint. |
| `heartbeat(Hb)` | _auto_ | Keeps an acquired task's lease alive (async at `TTL/2` remote, no-op local). |
| `encryptor(Encryptor)` | `NoopEncryptor` | Codec encryption at the durability boundary. |
| `maxConcurrentTasks(Integer)` | `64` | Ceiling on tasks holding a lease at once in this process. |
| `retryPolicy(RetryPolicy)` | `Exponential` | SDK-wide default retry policy for leaf functions. |

### Zero-dependency development (local mode)

With no `url`, no `network`, and no `RESONATE_URL`, the SDK runs the server state machine in-process with in-memory storage — no server required:

```java
Resonate r = new Resonate(); // in-process local mode
```

Local mode automatically uses a no-op heartbeat (there is no server lease endpoint to keep alive), so there is nothing extra to configure. Use a real server when you want multiple worker processes, persistence across restarts, or work shared between machines.

### Named worker group

The Java SDK sets the routing group directly on the builder:

```java
Resonate r = Resonate.builder()
        .url("http://localhost:8001")
        .group("worker-group-a")
        .build();
```

### Authentication

```java
Resonate r = Resonate.builder()
        .url("https://resonate.example.com")
        .token(System.getenv("RESONATE_TOKEN"))
        .build();
```

When omitted, `token` falls back to the `RESONATE_TOKEN` environment variable.

## `Resonate.register`

Register a durable function so it can be activated by the client, the CLI, or another durable function. `register` takes a **method reference** and returns it unchanged:

```java
r.register(Greeter::greetWorkflow);
```

By default a function registers under its own method name at version `1`. Override the name, version, and an optional per-function retry policy with the longer overloads:

```java
import io.resonatehq.resonate.Retry.Constant;

r.register(Greeter::greetWorkflow, "greet", 1);
r.register(Greeter::greetWorkflow, "greet", 2, new Constant(3, 1)); // per-function retry
```

A function you only ever hand to `ctx.run` as a method reference does **not** need to be registered — registration is for by-name entry points (`rpc`, the CLI, a top-level `run` target).

## `Resonate.run`

`run` starts a durable invocation in the **same process** and returns a typed handle. Passing a method reference makes the handle typed:

```java
ResonateHandle<String> handle = r.run("greet-1", Greeter::greetWorkflow, "world");
String result = handle.result(); // typed: String
```

There is also a by-name form returning an untyped handle (`ResonateHandle<Object>`), useful when the function is registered elsewhere:

```java
ResonateHandle<Object> handle = r.run("greet-1", "greet", "world");
Object result = handle.result();
```

The id is a unique key per invocation. **Reusing an id returns the first run's recorded result** instead of re-running — that idempotency is what makes a stable id (e.g. `factorial-10`) safe to re-invoke.

## `Resonate.rpc`

`rpc` invokes a function in a **remote process** by registered name and returns a handle. The target does not need to be registered locally:

```java
import io.resonatehq.resonate.Context.Opts;

ResonateHandle<Object> handle = r.options(new Opts().withTarget("backend"))
        .rpc("greet-1", "greet", "world");
Object result = handle.result();
```

`withTarget(...)` routes the call to a named group's anycast address. `example-recursive-factorial-java` uses this to keep clients out of the worker pool:

```java
// The client joins a distinct group and only dispatches — it never registers factorial.
Resonate r = Resonate.builder().url(url).group("factorial-client").build();
ResonateHandle<Object> handle =
        r.options(new Opts().withTarget(Factorial.WORKER_GROUP)).rpc(id, Factorial.NAME, n);
long result = ((Number) handle.result()).longValue();
```

Like `run`, `rpc` also accepts a method reference for a typed handle.

## `Resonate.get`

Get a handle to an existing execution by its promise ID. The handle is untyped:

```java
ResonateHandle<Object> handle = r.get("greet-1");
Object result = handle.result();
```

## Reading a handle's result

A handle blocks until its underlying promise settles, then decodes the value.

```java
String value = handle.result();   // blocks until settled, returns the decoded value
String id = handle.id().join();   // the durable promise id, once creation is confirmed
boolean done = handle.done();     // non-blocking: has the promise settled?
```

- **Typed** — `run` / `rpc` given a method reference return a `ResonateHandle<R>` whose `result()` returns `R`.
- **Untyped** — the by-name `run` / `rpc` forms and `get` return a `ResonateHandle<Object>` whose `result()` returns `Object`.

A rejected promise re-throws the application error from the rejection payload — by its real type where reconstructable across the durability boundary, otherwise an `ApplicationError` carrying the message.

**Decoding numbers from untyped handles.** Jackson decodes a small JSON integer as `Integer` and a large one as `Long`, so read numeric results from an untyped handle through `Number`:

```java
long result = ((Number) handle.result()).longValue();
```

## The `promises` sub-client

`r.promises` is a sub-client for direct durable-promise manipulation — the building block for the human-in-the-loop and external-coordination patterns, where an external party settles a promise a workflow created with `ctx.promise`:

```java
import io.resonatehq.resonate.Types.Value;

r.promises.resolve("approval-1", new Value(null, "approved")).join();
```

The full `promises` surface is `resolve`, `reject`, `cancel`, `get`, `create`, and `search`. Each returns a `CompletableFuture`, so call `.join()` (or compose with `thenApply`) to wait for it. See `resonate-human-in-the-loop-pattern-java`.

## The `schedules` sub-client

`r.schedule(...)` creates a schedule that periodically invokes a registered function on a cron expression:

```java
import java.util.List;
import java.util.Map;

Resonate.ResonateSchedule daily = r.schedule(
        "daily-report",   // schedule id
        "0 0 * * *",      // cron expression
        "generateReport", // registered function name
        List.of(),        // positional args
        Map.of(),         // keyword args
        null,             // promise timeout (null = default)
        1);               // function version

daily.delete(); // remove the schedule when no longer needed
```

The lower-level `r.schedules` sub-client (`create` / `get` / `delete` / `search`) is available for direct schedule manipulation. See `resonate-durable-sleep-scheduled-work-java`.

## `Resonate.stop`

```java
r.stop();
```

`stop` closes the network connection, joins outstanding background jobs, cancels the subscription-refresh loop, and shuts down the heartbeat. It is idempotent.

Call `stop` from any process that should exit after its work finishes — demo binaries, one-shot jobs, CI tasks. Without it, the background virtual threads keep the process from exiting after `main` would otherwise return.

**Do not call `stop` on a long-running worker.** It tears down the channels the worker uses to receive and hold work: the server connection closes (no more dispatched tasks), the heartbeat stops (in-flight task leases expire and the server reassigns them), and the subscription-refresh loop is cancelled (listeners on awaited promises are no longer re-registered). Workers should stay up; let SIGINT / SIGTERM end the lifecycle.

A worker that should stay alive blocks instead of stopping — `example-quickstart-java` and `example-recursive-factorial-java` use a latch:

```java
import java.util.concurrent.CountDownLatch;

r.register(Quickstart::countdown);
new CountDownLatch(1).await(); // keep the worker alive to receive CLI-invoked work
```

## CLI invocation binds arguments positionally

When a function is invoked from the CLI (`resonate invoke <id> --func <name> --arg ... --arg ...`), each repeated `--arg` binds to **one** declared parameter, in order — Java has no single-array-param shorthand. A CLI-invokable workflow must declare one parameter per `--arg`:

```java
// resonate invoke countdown.1 --func countdown --arg 5 --arg 1  →  count=5, delaySeconds=1
public static String countdown(Context ctx, int count, int delaySeconds) { ... }
```

A single `int[]` parameter would arity-mismatch the two `--arg` values. `example-quickstart-java` is the canonical reference. (This is the opposite of the Go SDK, where the same CLI invoke binds the whole list to one `[]int`.)

## Distinct Java idioms

- **Fluent builder, terminated by `build()`** — `Resonate.builder().url(...).group(...).build()`. `new Resonate()` is the local-mode shortcut.
- **Method references everywhere** — functions are supplied as `Owner::fn` (`Fn.F0`–`Fn.F5`), never a `String` name or reflected `Method` at the typed call sites. The functional-interface arity bounds a durable function to **at most five arguments** beyond the `Context`.
- **`static` methods** — durable functions are ordinary `public static` methods whose first parameter is a `Context`. Arg and return types must be JSON-serializable.
- **`try { ... } finally { r.stop(); }`** in one-shot programs — the `finally` guarantees the background threads are torn down so the process exits.
- **`ResonateHandle<R>` is typed via the reference** — the by-name form is `ResonateHandle<Object>`; cast or read through `Number` for numerics.
- **`new CountDownLatch(1).await()`** keeps a worker process alive — do not call `r.stop()` on it.

## Avoid

- Calling `r.run` / `r.rpc` / `r.get` (Client APIs) from inside a durable function — those belong to the ephemeral world. Use `ctx.run` / `ctx.rpc` inside workflows.
- Calling `r.stop()` on a long-running worker (see stop semantics) — it silently kills task processing.
- Declaring a single `int[]` (or `List`) parameter for a CLI-invoked function and expecting the repeated `--arg` values to land in it — they bind positionally, one per declared parameter.
- Reading a numeric result off an untyped `ResonateHandle<Object>` by casting straight to `Long` — Jackson may have decoded it as `Integer`. Read through `Number`.
- Passing more than five application arguments to a durable function — the `Fn.F0`–`Fn.F5` arity caps it at five beyond the `Context`. Bundle extra args into a record.
- Running on Java 8/11/17 — the SDK requires Java 21 virtual threads.

## Related skills

- `resonate-basic-durable-world-usage-java` — Context APIs: `ctx.run`, `ctx.rpc`, `ctx.sleep`, `ctx.promise`, `ctx.detached`, fan-out, retry policies, dependency injection
- `durable-execution` — foundational concepts; read this first if new to Resonate
- `resonate-defaults` — SDK-wide defaults (TTL, retry policy, top-level timeout, child timeout)
- `resonate-basic-ephemeral-world-usage-python` — the closest sibling; the Java API mirrors Python
- `resonate-basic-ephemeral-world-usage-typescript` / `-go` — siblings for comparison
