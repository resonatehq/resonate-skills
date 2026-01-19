---
name: resonate-develop-typescript
description: Build Resonate applications in TypeScript with correct determinism, structured concurrency, and clean separation between client (ephemeral world) APIs and Context (durable world) APIs. Use for end-to-end TS development: registration, invocation, RPC routing, durable promises, and workflow design.
---

# Resonate Develop TypeScript

## Overview

Use this skill to build Resonate apps in TypeScript. It combines SDK usage, workflow design, and correctness rules in one place. For spec-level reasoning that maps to these APIs, use `resonate-advanced-reasoning`.

## Mental Model (SDK View)

- Resonate Client lives in the ephemeral world and controls registration, top-level invocations, and promise admin.
- Resonate Context lives in the durable world and is used inside generator-based durable functions.
- Each `yield*` is a durable checkpoint. Replays are expected.

## API Boundary: Client vs Context

Use the Resonate Client for:
- `new Resonate(...)` initialization
- `register(...)`
- top-level `run`, `beginRun`, `rpc`, `beginRpc`, `schedule`, `get`
- `promises.create/get/resolve/reject`
- `setDependency`

Use Context inside generator functions for:
- `ctx.run`, `ctx.beginRun` (local durable calls)
- `ctx.rpc`, `ctx.beginRpc` (remote durable calls)
- `ctx.promise`, `ctx.sleep` (external waits)
- `ctx.date.now`, `ctx.math.random` (determinism)
- `ctx.getDependency` (dependency access)
- `ctx.panic`, `ctx.assert` (invariants)

## Core Correctness Rules

- Durable functions are `function*`, not `async`.
- Always `yield*` Context calls and futures.
- Return only serializable data.
- `ctx.rpc` arguments must be serializable; `ctx.run` args can be in-memory but returns must serialize.
- Wrap side effects in `ctx.run` or `ctx.rpc`.
- Use `ctx.date.now()` and `ctx.math.random()` for deterministic time and randomness.
- Prefer structured concurrency: fork then join.

## Naming and Registration Rules

- Always use `camelCase` for function names.
- Always register functions without custom names: `resonate.register(myFunction)`.
- Use the function name string for RPC (`\"myFunction\"`) and keep it camelCase.
- Do not register snake_case or override names with strings.

```ts
// WRONG
resonate.register(\"db_get_account_by_email\", db_get_account_by_email);
```

```ts
// RIGHT
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate();

function* dbGetAccountByEmail(ctx: Context, email: string) {
  return yield* ctx.run((_: Context, addr: string) => ({ email: addr }), email);
}

resonate.register(dbGetAccountByEmail);
```

## Decision Guide: Which API to Use

- Same process: `ctx.run` or `ctx.beginRun`.
- Different process/group: `ctx.rpc` or `ctx.beginRpc` with target.
- Fire-and-forget: `ctx.detached`.
- Human-in-the-loop or webhook: `ctx.promise` and external resolve.
- Wait for time: `ctx.sleep`.
- Schedule: `resonate.schedule`.

## Promise ID Strategy

- Top-level invocation ID is the idempotency key.
- Reuse the ID to return cached results; change it to force a fresh run.
- Make IDs readable and unique: `job/<uuid>`, `order/<id>`, `report/2025-01`.

## SDK Patterns with Code Examples

### Client initialization (local vs remote)

```ts
import { Resonate } from "@resonatehq/sdk";

const local = new Resonate();

const remote = new Resonate({
  url: "http://localhost:8001",
  group: "worker",
  auth: { username: "user", password: "pass" },
});
```

### Register and run locally

```ts
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate();

function* greetUser(_: Context, name: string) {
  return `hello ${name}`;
}

const greetR = resonate.register(greetUser);
const result = await greetR.run("greetUser.1", "Ada");
```

### Run in a separate process (RPC)

```ts
const handle = await resonate.beginRpc(
  "greetUser.2",
  "greetUser",
  "Ada",
  resonate.options({ target: "poll://any@workers" })
);

const result = await handle.result();
```

### Durable function with local and remote calls

```ts
function* workflow(ctx: Context, userId: string) {
  const profile = yield* ctx.run(loadProfile, userId);
  const score = yield* ctx.rpc(
    "computeScore",
    profile,
    ctx.options({ target: "poll://any@scorers" })
  );
  return { profile, score };
}
```

### Deterministic helpers

```ts
function* stable(ctx: Context) {
  return {
    now: yield* ctx.date.now(),
    rand: yield* ctx.math.random(),
  };
}
```

### Durable sleep

```ts
function* reminder(ctx: Context) {
  yield* ctx.sleep(5_000);
  yield* ctx.sleep({ until: new Date(Date.now() + 60_000) });
}
```

### Human-in-the-loop gate

```ts
function* approval(ctx: Context, orderId: string) {
  const p = yield* ctx.promise({ id: `approve/${orderId}` });
  yield* ctx.run(sendEmail, p.id);
  const ok = yield* p;
  return ok as boolean;
}
```

```ts
await resonate.promises.resolve("approve/123", true);
```

### Fork-join concurrency

```ts
function* fanout(ctx: Context, input: string) {
  const a = yield* ctx.beginRun(stepA, input);
  const b = yield* ctx.beginRun(stepB, input);
  const c = yield* ctx.beginRun(stepC, input);

  const ra = yield* a;
  const rb = yield* b;
  const rc = yield* c;

  return { ra, rb, rc };
}
```

### Options: timeout, tags, targets

```ts
function* bounded(ctx: Context, input: string) {
  return yield* ctx.run(
    step,
    input,
    ctx.options({ timeout: 30_000, tags: { job: "bounded" } })
  );
}
```

### Dependency injection

```ts
resonate.setDependency("db", dbConn);

function* handler(ctx: Context) {
  const db = ctx.getDependency("db");
  return yield* ctx.run(queryDb, db);
}
```

### Schedule a workflow

```ts
const schedule = await resonate.schedule(
  "hourly-report",
  "0 * * * *",
  generateReport,
  "acct-1"
);
```

### Async HTTP entrypoint (submit + poll)

```ts
app.post("/jobs", async (req, res) => {
  const id = `job/${crypto.randomUUID()}`;
  await resonate.beginRpc(
    id,
    "processJob",
    req.body,
    resonate.options({ target: "poll://any@workers" })
  );
  res.status(202).json({ id });
});

app.get("/jobs/:id", async (req, res) => {
  const handle = await resonate.get(req.params.id);
  res.json({ id: req.params.id, result: await handle.result() });
});
```

## Common Pitfalls (and Fixes)

- Missing `yield*` -> always `yield* ctx.run(...)` and `yield* future`.
- Using `async function*` -> use `function*` only.
- Non-determinism -> replace `Date.now()` and `Math.random()` with Context helpers.
- Side effects outside `ctx.run` -> wrap them so replays do not duplicate effects.
- Wrong target group -> match `poll://any@<group>` with the worker group.
- Custom string registrations -> register functions directly and keep names camelCase.

## When to Use Other Skills

- Use `resonate-advanced-reasoning` for spec-to-SDK semantics and correctness reasoning.
- Use `resonate-debug-troubleshoot` for runtime failures and error codes.
- Use `resonate-http-service-design` for HTTP gateway architecture patterns.
