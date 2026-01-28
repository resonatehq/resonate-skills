---
name: resonate-basic-ephemeral-world-usage-typescript
description: Core patterns for using Resonate Client APIs in the Ephemeral World - initialization, registration, top-level invocations, promise management, and dependency injection. Use this for application entry points and orchestration code outside of durable functions.
---

# Resonate Basic Ephemeral World Usage

## Overview

The **Ephemeral World** is where your application code lives that is NOT inside generator functions. This is where you:
- Initialize the Resonate Client
- Register functions
- Start top-level executions
- Manage promises externally
- Set dependencies

**Key distinction:** Ephemeral World code uses `resonate` (the Client). Durable World code uses `ctx` (the Context).

## Mental Model

```
┌─────────────────────────────────────────┐
│      EPHEMERAL WORLD (Stateless)        │
│                                         │
│  • Resonate Client initialization      │
│  • Function registration                │
│  • Top-level run/rpc calls             │
│  • Promise management (create/resolve)  │
│  • Dependency injection                 │
│                                         │
│  Uses: resonate.run(), resonate.rpc()  │
└─────────────────────────────────────────┘
              ↓ invokes ↓
┌─────────────────────────────────────────┐
│       DURABLE WORLD (Stateful)          │
│                                         │
│  • Generator functions with yield*      │
│  • Context APIs for sub-invocations    │
│  • Durable coordination and recovery    │
│                                         │
│  Uses: ctx.run(), ctx.rpc()            │
└─────────────────────────────────────────┘
```

## Core Rule

**You CANNOT use Context APIs in the ephemeral world, and you CANNOT use Client APIs in durable functions.**

## Initialization Patterns

### Local Development Mode (Zero Dependencies)

```ts
import { Resonate } from "@resonatehq/sdk";

// No URL = local in-memory mode
const resonate = new Resonate();
```

**Key behaviors:**
- In-memory storage (no persistence across restarts)
- No external dependencies required
- Perfect for testing and development
- **CRITICAL:** When using `new Resonate()` without a URL, function arguments are wrapped in an array

### Connected to Resonate Server

```ts
import { Resonate } from "@resonatehq/sdk";

const resonate = new Resonate({
  url: "http://localhost:8001",
  group: "workers",
  auth: {
    username: "user",
    password: "pass"
  }
});
```

**When to use:**
- Multiple worker processes
- Persistence across restarts
- Distributed execution
- Production deployments

### Environment Variables

```bash
export RESONATE_URL="http://localhost:8001"
export RESONATE_USERNAME="user"
export RESONATE_PASSWORD="pass"
```

```ts
const resonate = new Resonate(); // Picks up env vars automatically
```

**Resolution order:**
1. Constructor arguments (highest priority)
2. Environment variables
3. Built-in defaults (local mode)

## Registration Patterns

### Correct: Register with Function Reference

```ts
function* myWorkflow(ctx: Context, arg: string) {
  return `Hello ${arg}`;
}

// ✅ CORRECT
const stub = resonate.register(myWorkflow);
```

### Wrong: Custom String Names

```ts
// ❌ WRONG - Don't use custom string names
resonate.register("my_workflow", myWorkflow);

// ❌ WRONG - Don't use snake_case
resonate.register("db_get_user", dbGetUser);
```

**Why this matters:**
- Resonate uses function names for routing
- Custom names break RPC resolution
- Snake_case violates naming conventions

### Using the Returned Stub

```ts
const workflowStub = resonate.register(myWorkflow);

// Later, invoke directly via stub
const result = await workflowStub.run("execution-1", "World");
```

## Top-Level Invocation Patterns

### Run (Local Execution)

```ts
// Blocks until result is ready
const result = await resonate.run(
  "execution-id",
  myWorkflow,
  "arg1",
  "arg2"
);
```

**Use when:**
- Running in the same process
- Need the result immediately

### Begin Run (Non-Blocking Local)

```ts
// Returns immediately with handle
const handle = await resonate.beginRun(
  "execution-id",
  myWorkflow,
  "arg1"
);

// Do other work...

// Get result later
const result = await handle.result();
```

**Use when:**
- Starting work but not waiting immediately
- Running multiple executions concurrently

### RPC (Remote Execution)

```ts
// Blocks until remote execution completes
const result = await resonate.rpc(
  "execution-id",
  "myWorkflow", // String name!
  "arg1",
  resonate.options({
    target: "poll://any@workers"
  })
);
```

**Critical differences from run:**
- Uses string function name (not function reference)
- Requires target specification
- Executes in different process/group

### Begin RPC (Non-Blocking Remote)

```ts
const handle = await resonate.beginRpc(
  "execution-id",
  "myWorkflow",
  "arg1",
  resonate.options({
    target: "poll://any@workers"
  })
);

const result = await handle.result();
```

## Options Pattern

```ts
await resonate.run(
  "execution-id",
  myWorkflow,
  "arg1",
  resonate.options({
    timeout: 60_000,        // 60 seconds in ms
    tags: { userId: "123" },
    version: 1
  })
);
```

**Available options:**
- `timeout`: Max execution time in milliseconds
- `tags`: Metadata for filtering/searching
- `version`: Function version for schema evolution
- `target`: RPC routing (poll://any@group-name)

## Promise Management

### Create Promise

```ts
await resonate.promises.create(
  "approval-123",
  Date.now() + 30000 // timeout 30s from now
);
```

### Get Promise

```ts
const promise = await resonate.promises.get("approval-123");
```

### Resolve Promise (Human-in-the-Loop)

```ts
// Elsewhere (webhook, UI, CLI):
await resonate.promises.resolve("approval-123", {
  approved: true,
  approver: "alice@example.com"
});
```

### Reject Promise

```ts
await resonate.promises.reject("approval-123", {
  reason: "Insufficient funds"
});
```

## Dependency Injection

### Set Dependencies

```ts
import { createClient } from "@supabase/supabase-js";

const db = createClient(url, key);

// Set in ephemeral world
resonate.setDependency("db", db);
resonate.setDependency("config", { apiKey: "..." });
```

**Rules:**
- Only set dependencies in ephemeral world
- Set before registering functions that use them
- Pass non-serializable objects (DB connections, etc.)

### Access in Durable World

```ts
function* myWorkflow(ctx: Context, userId: string) {
  // Get in durable world
  const db = ctx.getDependency("db");

  const user = yield* ctx.run(async (_ctx, id) => {
    return await db.from("users").select().eq("id", id).single();
  }, userId);

  return user;
}
```

## Scheduling

```ts
const schedule = await resonate.schedule(
  "daily-report",
  "0 8 * * *",      // Every day at 8am
  generateReport,
  "arg1"
);

// Later, delete the schedule
await schedule.delete();
```

## Subscription (Get Existing Execution)

```ts
// Subscribe to existing execution
const handle = await resonate.get("execution-id");

// Wait for result
const result = await handle.result();

// Check if done
const isDone = await handle.done();
```

**Use when:**
- Checking status of long-running executions
- Reconnecting to executions after restart
- Monitoring from external systems

## Common Pitfalls

### 1. Mixing Ephemeral and Durable APIs

```ts
// ❌ WRONG - Using ctx in ephemeral world
async function main() {
  const result = await ctx.run(myFunc);  // ctx doesn't exist here!
}

// ✅ CORRECT
async function main() {
  const result = await resonate.run("id", myFunc);
}
```

### 2. Using Client APIs Inside Generators

```ts
// ❌ WRONG
function* myWorkflow(ctx: Context) {
  const result = await resonate.run("id", otherFunc);  // Wrong API!
}

// ✅ CORRECT
function* myWorkflow(ctx: Context) {
  const result = yield* ctx.run(otherFunc);
}
```

### 3. Missing Target on RPC

```ts
// ❌ WRONG - RPC needs target
await resonate.rpc("id", "funcName", "arg");

// ✅ CORRECT
await resonate.rpc(
  "id",
  "funcName",
  "arg",
  resonate.options({ target: "poll://any@workers" })
);
```

### 4. Forgetting In-Memory Mode Wraps Args in Array

```ts
// In-memory mode (no URL)
const resonate = new Resonate();

function* workflow(ctx: Context, state: MyState) {
  // ❌ WRONG - state will be [MyState], not MyState!
  console.log(state.someField);  // undefined!
}

// ✅ CORRECT - Handle array wrapping in local mode
function* workflow(ctx: Context, state: any) {
  const actualState = Array.isArray(state) ? state[0] : state;
  console.log(actualState.someField);  // Works!
}
```

### 5. Using Async Instead of Generators

```ts
// ❌ WRONG - Durable functions must be generators
async function* workflow(ctx: Context) { }

// ✅ CORRECT
function* workflow(ctx: Context) { }
```

## Complete Example

```ts
import "dotenv/config";
import { Resonate, type Context } from "@resonatehq/sdk";

// Initialize client (ephemeral world)
const resonate = new Resonate({
  url: process.env.RESONATE_URL,
  group: "workers"
});

// Set dependencies (ephemeral world)
resonate.setDependency("apiKey", process.env.API_KEY);

// Register functions (ephemeral world)
const workflowStub = resonate.register(myWorkflow);

// Start execution (ephemeral world)
async function main() {
  const handle = await resonate.beginRun(
    "workflow-1",
    myWorkflow,
    { userId: "123" }
  );

  console.log("Started:", handle.id);

  const result = await handle.result();
  console.log("Result:", result);
}

// Durable function (durable world - uses Context APIs)
function* myWorkflow(ctx: Context, input: any) {
  const apiKey = ctx.getDependency("apiKey");

  const data = yield* ctx.run(fetchData, input.userId);
  const processed = yield* ctx.run(processData, data);

  return processed;
}

async function fetchData(_ctx: Context, userId: string) {
  // Fetch logic
  return { userId, data: "..." };
}

async function processData(_ctx: Context, data: any) {
  // Process logic
  return { processed: true };
}

main().catch(console.error);
```

## Decision Tree

**Where am I?**
- In `main()` or app entry point → Ephemeral World (use Client APIs)
- In `function*` with `ctx` parameter → Durable World (use Context APIs)

**What do I need to do?**
- Initialize Resonate → `new Resonate()`
- Register a function → `resonate.register(func)`
- Start a workflow → `resonate.run()` or `resonate.beginRun()`
- Start remote work → `resonate.rpc()` or `resonate.beginRpc()`
- Manage promises → `resonate.promises.*()`
- Share resources → `resonate.setDependency()`
- Schedule cron → `resonate.schedule()`
- Check execution status → `resonate.get()`

## Summary

The Ephemeral World is your **control plane**:
- One Client per process
- Stateless orchestration
- Entry and exit points
- External promise resolution
- Dependency setup

Keep it simple, keep it separate from durable functions.
