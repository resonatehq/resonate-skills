---
name: resonate-async-await-engine-typescript
description: The Resonate TypeScript SDK's async/await execution engine — import from @resonatehq/sdk/async, register async functions, eager ctx.run fan-out with Promise.all, Never-default retries and Exponential opt-in, and when to choose the async engine vs the generator engine. Introduced in v0.11.0. Use when writing new TypeScript workflows with async/await instead of function*/yield*.
license: Apache-2.0
---

# Resonate Async/Await Engine (TypeScript)

> **SDK version:** This skill reflects `@resonatehq/sdk` v0.11.4 (current on npm). The async/await engine was introduced in v0.11.0.

## Overview

The TypeScript SDK ships two execution engines:

| Engine | Import path | Function style | Release |
|---|---|---|---|
| **Generator engine** | `@resonatehq/sdk` | `function*` / `yield*` | v0.10.x and earlier; still current |
| **Async/await engine** | `@resonatehq/sdk/async` | `async function` / `await` | v0.11.0+ |

Both engines connect to the same Resonate Server, share the same durable-promise substrate, and support the same patterns (fan-out, human-in-the-loop, saga, etc.). The async engine is opt-in — existing generator-engine code is unaffected.

For the generator engine, see `resonate-basic-ephemeral-world-usage-typescript` and `resonate-basic-durable-world-usage-typescript`.

---

## When to use the async engine

- Writing new TypeScript workflows and prefer `async/await` over `function*`/`yield*`
- Integrating Resonate into a codebase that already uses `async/await` throughout
- Browser or edge environments: `@resonatehq/sdk/async` exports are browser-compatible

## When to keep the generator engine

- Existing generator-engine code — there is no reason to migrate; the generator engine is fully supported
- You need `ctx.detached()` with the full bounded-replay pattern (async engine also has `detach()` but check the SDK changelog for your version)

---

## Installation

```bash
npm install @resonatehq/sdk@0.11.4
```

Both engines are in the same package. The generator engine is the default export; the async engine is the `/async` sub-export.

---

## Basic usage

```typescript
import { Resonate } from "@resonatehq/sdk/async";

const resonate = new Resonate({ url: "http://localhost:8001" });

// Register an async function
resonate.register("greet", async (ctx, name: string) => {
  return `Hello, ${name}!`;
});

// Start a workflow — resonate.run(id, funcOrName, ...args) → Promise<ResonateHandle<T>>
const handle = await resonate.run("greet-001", "greet", "world");

// Await the result via the durable-promise subscription
const result = await handle.result();
// result === "Hello, world!"
```

**Key differences from the generator engine:**

- Import is `from "@resonatehq/sdk/async"` not `from "@resonatehq/sdk"`
- Functions are `async function` not `function*`
- `ctx.run(func, ...args)` in the async engine has **no id argument** (ID is auto-generated); in the generator engine `ctx.run(fn, ...args)` is also ID-less at the context level, but the API shape differs
- No `beginRun` / `beginRpc` on the async-engine client — every `run` and `rpc` already returns a `ResonateHandle<T>` directly, so `beginRun` would be redundant

---

## Eager ctx.run and fan-out with Promise.all

Inside an async workflow function, `ctx.run(func, ...args)` returns a `DurablePromise<T>` immediately — the child starts executing right away. Hold several and await them together for parallel fan-out:

```typescript
resonate.register("processItems", async (ctx, items: string[]) => {
  // Start all children eagerly — they run in parallel
  const promises = items.map((item) =>
    ctx.run(processOne, item)
  );

  // Await all results
  const results = await Promise.all(promises);
  return results;
});

resonate.register("processOne", async (ctx, item: string) => {
  // ... durable leaf work
  return `processed: ${item}`;
});
```

This is the async-engine equivalent of the generator engine's `ctx.beginRun` + sequential `yield*` pattern.

---

## Retries: Never default, Exponential opt-in

The async/await engine defaults to **no retry** (`Never`) when no retry policy is specified. To opt into retries on a specific `ctx.run` call, pass `ctx.options({ retryPolicy: ... })`:

```typescript
import { Resonate, Exponential, Never } from "@resonatehq/sdk/async";

resonate.register("reliableStep", async (ctx, taskId: string) => {
  // Default: Never retry — a failure propagates immediately
  const noRetryResult = await ctx.run(fragileOp, taskId);

  // Explicit Exponential retry on a specific child
  const retryResult = await ctx.run(
    flakeyOp,
    taskId,
    ctx.options({ retryPolicy: new Exponential() }),
  );

  return { noRetryResult, retryResult };
});
```

`Never`, `Exponential`, `Linear`, and `Constant` are all exported from `@resonatehq/sdk/async` and can be used with both engines.

**Tip:** For saga-style compensation, let steps fail immediately with the default `Never` policy and catch in the outer function.

---

## ctx.options() in the async engine

`ctx.options()` accepts the same fields as the generator engine:

```typescript
ctx.options({
  retryPolicy: new Exponential(),   // retry policy (default: Never)
  target: "poll://any@workers",     // worker group routing
  timeout: 30_000,                  // ms; defaults to 24h
  tags: { "env": "prod" },          // arbitrary tags on the child promise
  version: 1,                       // function version pin
})
```

---

## Migration from generator engine

| Generator engine | Async/await engine |
|---|---|
| `import { Resonate } from "@resonatehq/sdk"` | `import { Resonate } from "@resonatehq/sdk/async"` |
| `function* workflow(ctx: Context, ...)` | `async function workflow(ctx: Context, ...)` |
| `yield* ctx.run(fn, args)` | `await ctx.run(fn, args)` |
| `yield* ctx.rpc(fn, args)` | `await ctx.rpc(fn, args)` |
| `yield* ctx.sleep(ms)` | `await ctx.sleep(ms)` |
| `const p = yield* ctx.promise(opts)` | `const dp = ctx.promise(opts)` — no await; returns `DurablePromise` with `.id` |
| `const val = yield* p` | `const val = await dp` |
| `resonate.beginRun(id, fn, ...args)` | `resonate.run(id, fn, ...args)` (returns handle) |
| `await resonate.run(id, fn, ...args)` | `(await resonate.run(id, fn, ...args)).result()` |

The server-side promise structure is identical; existing generator-engine workflows on the server are unaffected by the async engine.

See the [SDK README migration guide](https://github.com/resonatehq/resonate-sdk-ts#migrating-from-generators) for the authoritative changelog.

---

## External promise resolution (human-in-the-loop)

The async engine uses the same `resonate.promises.resolve()` API as the generator engine:

```typescript
// Inside a workflow: park on an external decision
resonate.register("awaitApproval", async (ctx, orderId: string) => {
  const dp = ctx.promise<{ approved: boolean }>();
  // dp.id is available synchronously — surface it to whoever needs to approve
  console.log("Approval promise id:", dp.id);
  const decision = await dp;
  return decision;
});

// In a webhook handler: resolve the parked promise
const data = Buffer.from(JSON.stringify({ approved: true })).toString("base64");
await resonate.promises.resolve(promiseId, { data });
```

---

## Full runnable example

```typescript
import { Resonate, Exponential } from "@resonatehq/sdk/async";

const resonate = new Resonate({ url: "http://localhost:8001" });

resonate.register("summarize", async (ctx, texts: string[]) => {
  // Fan-out: summarize each text in parallel
  const summaries = await Promise.all(
    texts.map((t) =>
      ctx.run(
        summarizeOne,
        t,
        ctx.options({ retryPolicy: new Exponential() }),
      )
    )
  );
  return summaries.join("\n");
});

resonate.register("summarizeOne", async (ctx, text: string) => {
  // ... call an LLM or summarization service
  return `Summary of: ${text.slice(0, 50)}`;
});

const handle = await resonate.run("summarize-job-001", "summarize", [
  "First document ...",
  "Second document ...",
]);
console.log(await handle.result());

await resonate.stop();
```
