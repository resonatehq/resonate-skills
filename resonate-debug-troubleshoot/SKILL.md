---
name: resonate-debug-troubleshoot
description: Debug and troubleshoot Resonate applications and deployments, especially TypeScript SDK and Resonate Server issues. Use when investigating failures, stuck workflows, error codes, or unexpected replays.
---

# Resonate Debug and Troubleshoot

## Overview

Use this skill to diagnose Resonate runtime issues in the TypeScript SDK or Resonate Server. Focus on reproducible symptoms, promise state, registration, determinism, and routing.

## Triage Flow

1. Classify the failure: build/type error, runtime SDK error, server error code, or workflow stuck/hanging.
2. Confirm worker is running and connected to the correct server URL and group.
3. Inspect the promise state and call graph for the execution ID.
4. Validate registration names and serialization rules.
5. Check determinism and `yield*` usage inside durable functions.

## CLI and Observability Tools

- Start a clean local server: `resonate dev`.
- Invoke a workflow: `resonate invoke <promise-id> --func <name> --arg ...`.
- Inspect call graph: `resonate tree <promise-id>`.
- Inspect promise state: `resonate promises get <id>`.
- Search promises by state or tags: `resonate promises search <id>`.
- Resolve/reject external promises: `resonate promises resolve|reject <id>`.

## Common Symptoms and Fixes

- Function not registered (1103): ensure `register` name matches invocation name.
- Function already registered (1102): avoid duplicate names or versions in one process.
- Args unencodeable (1106): pass only serializable args to RPC or top-level invocations.
- Promise already exists (40900): expected for idempotency; change the ID when you want a fresh run.
- Promise not found (40400): wrong ID or wrong server URL.
- Promise already resolved/rejected/timed out (40300-40303): treat as idempotency; pick a new ID or handle as cached result.
- Workflow hangs: target group mismatch, no active workers, or missing `yield*` before awaiting a future.
- Unexpected replays: expected after each checkpoint; avoid side effects outside `ctx.run`.

## Code Examples

### Minimal repro (worker)

```ts
// worker.ts
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001", group: "worker" });

function* ping(_: Context, name: string) {
  return `pong ${name}`;
}

resonate.register("ping", ping);
```

### Minimal repro (client)

```ts
// client.ts
import { Resonate } from "@resonatehq/sdk";

const client = new Resonate({ url: "http://localhost:8001", group: "client" });

const handle = await client.beginRpc(
  "ping.1",
  "ping",
  "Ada",
  client.options({ target: "poll://any@worker" })
);

console.log(await handle.result());
```

### Inspect promise state

```bash
resonate promises get ping.1
resonate tree ping.1
```

### ID reuse (expected cache)

```ts
const id = "order/123";
await resonate.run(id, processOrder, "123");
await resonate.run(id, processOrder, "123"); // returns cached result
```

### Group mismatch (fix target)

```ts
// worker group is "workers"
const client = new Resonate({ url: "http://localhost:8001", group: "client" });

// wrong target
await client.rpc("job.1", "work", "x", client.options({ target: "poll://any@worker" }));

// correct target
await client.rpc("job.2", "work", "x", client.options({ target: "poll://any@workers" }));
```

### Missing yield* (avoid)

```ts
function* bad(ctx: Context) {
  const x = ctx.run(step); // missing yield*
  return x;
}
```

```ts
function* good(ctx: Context) {
  const x = yield* ctx.run(step);
  return x;
}
```

### Serialization error (avoid non-serializable args for rpc)

```ts
function* badRpc(ctx: Context, db: DbConn) {
  return yield* ctx.rpc("remote", db); // db is not serializable
}
```

```ts
function* goodRpc(ctx: Context, dbId: string) {
  return yield* ctx.rpc("remote", dbId);
}
```

## Determinism Checklist

- Replace `Date.now()` and `Math.random()` with `ctx.date.now()` and `ctx.math.random()`.
- Wrap side effects inside `ctx.run`.
- Ensure return values are serializable.

## Routing and Group Checks

- Verify the worker group matches the target used in `rpc`/`beginRpc`.
- Use `poll://any@<group>` for load balancing across that group.
- Ensure the Resonate Client is configured with the correct `url` and `group`.

## Data to Capture for a Bug Report

- Resonate Server version and SDK version.
- The promise ID and function name.
- Error code and full message.
- The target/group used for RPC calls.
- Minimal reproduction with a single durable function.
