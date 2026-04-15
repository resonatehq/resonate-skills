---
name: resonate-advanced-reasoning
description: Advanced reasoning bridge between the Distributed Async Await specification and the Resonate TypeScript SDK. Use when mapping spec concepts (processes, executions, promises, coordination, recovery) to concrete SDK patterns and when validating correctness, durability, and failure semantics.
license: Apache-2.0
---

# Resonate Advanced Reasoning

## Overview

Use this skill to translate Distributed Async Await concepts into Resonate TypeScript SDK usage. Treat the spec as the mental model and the SDK as its concrete expression.

## Spec to SDK Translation Map

### System Model -> Runtime

- Process -> a single Node.js process running `new Resonate(...)`.
- Logical process -> a group of interchangeable workers (`group` + `target`).
- Message passing -> Async RPC over HTTP long poll (or Kafka/SQS transport).
- Addressing -> `poll://any@<group>` or `poll://id@<group>` targets.

### Execution Model -> SDK Semantics

- Execution -> a durable function invocation.
- Invoke event -> `run`, `rpc`, `beginRun`, or `beginRpc`.
- Return event -> durable promise resolution stored by the server.
- Durable promise -> the promise ID for each invocation step.
- Resume -> internal callback that replays and continues after a promise resolves.

### Programming Model -> SDK APIs

- Durable function -> `resonate.register("name", function* ...)`.
- Local invocation -> `ctx.run` / `ctx.beginRun`.
- Remote invocation -> `ctx.rpc` / `ctx.beginRpc` with target options.
- External promise -> `ctx.promise()` and `resonate.promises.resolve/reject`.
- Call graph -> inspect with `resonate tree <id>`.
- Root promise -> the top-level invocation ID passed to `run` or `rpc`.

## Coordination Semantics (Spec -> SDK)

### Eventual resumption

Spec: caller awaits a promise that is not yet resolved.

SDK:
- Caller yields `ctx.rpc(...)` or `ctx.beginRpc(...)`.
- Resonate Server stores promise and registers resume callback.
- Caller resumes after the promise is resolved.

```ts
function* parent(ctx: Context, id: string) {
  const child = yield* ctx.beginRpc(
    "child",
    id,
    ctx.options({ target: "poll://any@workers" })
  );
  const result = yield* child; // eventual resumption
  return result;
}
```

### Immediate resumption

Spec: caller awaits a promise already resolved.

SDK:
- On replay, `yield* ctx.run(...)` returns stored result immediately.
- No new execution is spawned for the already-completed promise.

```ts
function* replayed(ctx: Context, id: string) {
  const cached = yield* ctx.run(step, id); // returns stored value
  return cached;
}
```

## Recovery Semantics (Spec -> SDK)

Spec: interruption-transparent execution.

SDK:
- Each `yield*` checkpoint writes a durable promise.
- On crash, Resonate replays the function and reuses stored results.
- Side effects must be wrapped in durable calls to avoid duplication.

```ts
function* durable(ctx: Context, id: string) {
  const v1 = yield* ctx.run(step1, id);
  const v2 = yield* ctx.run(step2, v1);
  return v2;
}
```

## Determinism Requirements (Spec -> SDK)

Spec: execution should be equivalent with or without interruptions.

SDK:
- Use `ctx.date.now()` and `ctx.math.random()`.
- Do not call `Date.now()` or `Math.random()` inside durable code.
- Ensure all return values are serializable.

```ts
function* deterministic(ctx: Context) {
  return {
    now: yield* ctx.date.now(),
    rand: yield* ctx.math.random(),
  };
}
```

## Idempotency and Promise IDs

Spec: durable promises deduplicate re-invocation.

SDK:
- Top-level invocation ID is the idempotency key.
- Reusing the same ID returns cached results.
- Generate new IDs for a fresh execution.

```ts
await resonate.run("order/123", processOrder, "123");
await resonate.run("order/123", processOrder, "123"); // returns cached
await resonate.run("order/124", processOrder, "124"); // new execution
```

## Local vs Remote Execution Reasoning

Spec: locality defines whether an invocation stays in-process or crosses processes.

SDK:
- `ctx.run` stays local.
- `ctx.rpc` crosses to another process group and resumes when resolved.

```ts
function* workflow(ctx: Context, input: string) {
  const local = yield* ctx.run(stepLocal, input);
  const remote = yield* ctx.rpc(
    "stepRemote",
    local,
    ctx.options({ target: "poll://any@workers" })
  );
  return remote;
}
```

## Message Passing and Addressing

Spec: messages may be sent to a process or a group.

SDK:
- Unicast: use a specific `poll://id@group` target.
- Anycast: use `poll://any@group` to let the server route.

```ts
const result = yield* ctx.rpc(
  "task",
  data,
  ctx.options({ target: "poll://any@workers" })
);
```

## Advanced Reasoning Checklist

- Is each side effect behind a durable step (`ctx.run` or `ctx.rpc`)?
- Are all durable return values serializable?
- Are promise IDs stable and meaningful?
- Does the target group match the worker group?
- Are retries bounded with timeouts where appropriate?
- Is concurrency structured (fork then join)?

## When to Hand Off

- Use `resonate-develop-typescript` for SDK usage patterns and hands-on TS authoring.
- Use `resonate-debug-troubleshoot` for runtime diagnosis.
