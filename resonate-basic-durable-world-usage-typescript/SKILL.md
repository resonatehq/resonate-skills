---
name: resonate-basic-durable-world-usage-typescript
description: The Resonate TypeScript SDK's Context API reference for durable generator functions — ctx.run, ctx.rpc, ctx.sleep, ctx.promise, determinism rules, and structured concurrency. Use once you've decided to use Resonate and are writing code inside function* bodies. For the conceptual decision of whether to use durable execution at all, see the durable-execution skill.
license: Apache-2.0
---

# Resonate Basic Durable World Usage

> **SDK version:** This skill reflects `@resonatehq/sdk` v0.10.0 (current on npm).

## Overview

The **Durable World** is inside generator functions where you write durable, recoverable execution logic using the Context object. Every operation checkpoints progress, enabling automatic recovery after failures.

**Key distinction:** Durable World code uses `ctx` (Context). Ephemeral World code uses `resonate` (Client).

## Mental Model

```
Generator Function (function*)
       ↓
   yield* ctx.run()      ← Checkpoint
       ↓
   yield* ctx.sleep()    ← Checkpoint + Suspend
       ↓
   yield* ctx.rpc()      ← Checkpoint + Cross Process
       ↓
   return result         ← Final Checkpoint
```

**Every `yield*` is a durable checkpoint.**

If the process crashes, the execution replays from the last checkpoint using stored results.

## Core Syntax Rules

### 1. Use `function*` (Generator Functions)

```ts
// ✅ CORRECT - Generator function
function* myWorkflow(ctx: Context, arg: string) {
  // Durable code here
}

// ❌ WRONG - Async function
async function myWorkflow(ctx: Context, arg: string) {
  // Not durable!
}

// ❌ WRONG - Async generator
async function* myWorkflow(ctx: Context, arg: string) {
  // Invalid syntax for Resonate
}
```

### 2. Always `yield*` Context Calls

```ts
function* myWorkflow(ctx: Context) {
  // ✅ CORRECT
  const result = yield* ctx.run(someFunc, "arg");

  // ❌ WRONG - Missing yield*
  const result = ctx.run(someFunc, "arg");

  // ❌ WRONG - Using await
  const result = await ctx.run(someFunc, "arg");
}
```

### 3. First Parameter is Always Context

```ts
// ✅ CORRECT
function* myWorkflow(ctx: Context, userId: string, data: any) {
  // ...
}

// ❌ WRONG - Missing Context
function* myWorkflow(userId: string) {
  // Can't call Context APIs!
}
```

## Context.run() - Local Invocation

**Signature:** `ctx.run(func, ...args) → yield* → result`

**Behavior:**
- Executes in same process
- Synchronous (blocks until complete)
- Arguments can be in-memory (no serialization required)
- Return value must serialize (stored for replay)

### Basic Usage

```ts
function* workflow(ctx: Context, userId: string) {
  // Call synchronously, get result immediately
  const user = yield* ctx.run(fetchUser, userId);
  const orders = yield* ctx.run(fetchOrders, userId);

  return { user, orders };
}

async function fetchUser(_ctx: Context, userId: string) {
  // Regular async function - can throw, use await, etc.
  return { id: userId, name: "Alice" };
}
```

### With Non-Serializable Arguments

```ts
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db"); // DB connection object

  // ✅ OK - db object doesn't need to serialize
  const user = yield* ctx.run(queryDatabase, db, "123");

  return user;
}

async function queryDatabase(_ctx: Context, db: any, userId: string) {
  return await db.query("SELECT * FROM users WHERE id = $1", [userId]);
}
```

### Wrapping Side Effects

```ts
function* workflow(ctx: Context, email: string) {
  // ✅ CORRECT - Wrap side effects in ctx.run
  yield* ctx.run(sendEmail, email);

  // ❌ WRONG - Side effect outside ctx.run
  await sendEmailDirect(email); // Will re-send on replay!
}

async function sendEmail(_ctx: Context, email: string) {
  // Side effect happens here, checkpointed result prevents replay
  await emailService.send(email);
}
```

## Context.beginRun() - Non-Blocking Local

**Signature:** `ctx.beginRun(func, ...args) → yield* → Future`

**Use when:**
- Starting multiple operations concurrently
- Following fork-join pattern

### Fork-Join Concurrency

```ts
function* workflow(ctx: Context, userId: string) {
  // Fork: Start all operations
  const userFuture = yield* ctx.beginRun(fetchUser, userId);
  const ordersFuture = yield* ctx.beginRun(fetchOrders, userId);
  const invoicesFuture = yield* ctx.beginRun(fetchInvoices, userId);

  // Join: Await all results
  const user = yield* userFuture;
  const orders = yield* ordersFuture;
  const invoices = yield* invoicesFuture;

  return { user, orders, invoices };
}
```

**Pattern:** Always fork first, then join. Don't interleave.

### Anti-Pattern: Interleaved Begin/Await

```ts
// ❌ WRONG - Hard to read, unclear concurrency
function* workflow(ctx: Context, id: string) {
  const f1 = yield* ctx.beginRun(step1, id);
  const r1 = yield* f1;
  const f2 = yield* ctx.beginRun(step2, r1);
  const r2 = yield* f2;
  return r2;
}

// ✅ CORRECT - Sequential operations should use run
function* workflow(ctx: Context, id: string) {
  const r1 = yield* ctx.run(step1, id);
  const r2 = yield* ctx.run(step2, r1);
  return r2;
}
```

## Context.rpc() - Remote Invocation

**Signature:** `ctx.rpc(funcName, ...args, options?) → yield* → result`

**Behavior:**
- Executes in different process/group
- Blocks until remote completes
- All arguments must serialize (crossing process boundary)
- Return value must serialize

### Basic Usage

```ts
// Process A
function* workflow(ctx: Context, userId: string) {
  // Call function in process B
  const score = yield* ctx.rpc(
    "computeScore",
    userId,
    { model: "v2" },
    ctx.options({ target: "poll://any@scorers" })
  );

  return score;
}

// Process B (different worker group "scorers")
function* computeScore(ctx: Context, userId: string, options: any) {
  // Expensive computation
  return 850;
}
```

### Target Specification

```ts
ctx.options({
  target: "poll://any@workers"  // Any worker in "workers" group
})

ctx.options({
  target: "poll://any@gpu-workers"  // Specific worker group
})
```

### Serialization Rules

```ts
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db");

  // ❌ WRONG - DB connection can't serialize
  yield* ctx.rpc("processData", db);

  // ✅ CORRECT - Only pass serializable data
  const data = yield* ctx.run(fetchData, db);
  yield* ctx.rpc("processData", data);
}
```

**Serializable:** strings, numbers, booleans, plain objects, arrays, null
**Not serializable:** functions, class instances, DB connections, file handles

## Context.beginRpc() - Non-Blocking Remote

```ts
function* workflow(ctx: Context, userId: string) {
  // Fork: Start remote operations
  const scoreFuture = yield* ctx.beginRpc(
    "computeScore",
    userId,
    ctx.options({ target: "poll://any@scorers" })
  );

  const riskFuture = yield* ctx.beginRpc(
    "assessRisk",
    userId,
    ctx.options({ target: "poll://any@risk-workers" })
  );

  // Join: Await results
  const score = yield* scoreFuture;
  const risk = yield* riskFuture;

  return { score, risk };
}
```

## Context.detached() - Fire and Forget

```ts
function* workflow(ctx: Context, orderId: string) {
  // Start analytics tracking but don't wait
  yield* ctx.detached(
    "trackOrder",
    orderId,
    ctx.options({ target: "poll://any@analytics" })
  );

  // Continue immediately
  return processOrder(orderId);
}
```

**Important:** Detached calls are NOT implicitly awaited, even with structured concurrency.

## Context.sleep() - Durable Sleep

**Signature:** `ctx.sleep(ms | options) → yield* → void`

### Sleep for Duration

```ts
function* workflow(ctx: Context) {
  yield* ctx.sleep(5000);  // Sleep 5 seconds

  yield* ctx.sleep({ for: 60_000 });  // Sleep 1 minute
}
```

### Sleep Until Specific Time

```ts
function* workflow(ctx: Context) {
  const tomorrow8am = new Date();
  tomorrow8am.setDate(tomorrow8am.getDate() + 1);
  tomorrow8am.setHours(8, 0, 0, 0);

  yield* ctx.sleep({ until: tomorrow8am });

  // Resumes at exactly 8am tomorrow
}
```

**Key behaviors:**
- No limit on sleep duration
- Activation terminates during sleep
- Resumes in new activation when time expires
- Sleep is durable - survives crashes

## Context.promise() - External Promises

**Signature:** `ctx.promise(options?) → yield* → Future`

**Use for:** Human-in-the-loop, webhooks, external triggers

### Promise ID Auto-Generation

**IMPORTANT:** In the durable world, Resonate automatically generates deterministic promise IDs based on the root promise ID (from ephemeral world invocation) UNLESS you use `ctx.detached()`.

**When to use explicit IDs:**
- Human-in-the-loop: External resolver needs to know the ID
- Webhooks: ID must be communicated to external system
- Detached operations: Using `ctx.detached()`

**When to use auto-generated IDs:**
- Internal promises that don't need external resolution
- You want maximum replay safety

```ts
// Auto-generated ID (deterministic, based on call tree)
const promise = yield* ctx.promise<T>();
const result = yield* promise;

// Explicit ID (for external resolution)
const promise = yield* ctx.promise<T>({
  id: `approval/${orderId}`
});
```

### Basic HITL Pattern

```ts
function* approvalWorkflow(ctx: Context, orderId: string) {
  // Create promise with explicit ID for external resolution
  const approvalPromise = yield* ctx.promise<Decision>({
    id: `approval/${orderId}` // External resolver needs this ID
  });

  // Send notification with promise ID
  yield* ctx.run(sendApprovalEmail, approvalPromise.id);

  // Block until human approves (via external resolve)
  const decision = yield* approvalPromise;

  if (decision.approved) {
    yield* ctx.run(processOrder, orderId);
  }

  return decision;
}
```

### External Resolution (Ephemeral World)

```ts
// Webhook handler or UI callback
app.post("/approve/:promiseId", async (req, res) => {
  // Base64 encode data before sending to Resonate server
  const response = { approved: true, approver: req.body.userId };
  const encodedData = Buffer.from(JSON.stringify(response)).toString('base64');

  await resonate.promises.settle(req.params.promiseId, "resolved", {
    data: encodedData,
  });

  res.json({ status: "approved" });
});
```

**CRITICAL: Promise ID Determinism**

Promise IDs must be deterministic for workflow replay to work correctly:

```ts
// ❌ BAD - Date.now() changes on every replay
function* approvalLoop(ctx: Context, orderId: string) {
  while (true) {
    const promise = yield* ctx.promise({
      id: `approval/${orderId}/${Date.now()}` // WRONG!
    });
    const result = yield* promise;
    if (result.approved) break;
  }
}

// ✅ GOOD - Use counter/attempt number
function* approvalLoop(ctx: Context, orderId: string) {
  let attemptNumber = 1;
  while (true) {
    const promise = yield* ctx.promise({
      id: `approval/${orderId}/attempt-${attemptNumber}` // Deterministic!
    });
    const result = yield* promise;
    if (result.approved) break;
    attemptNumber++;
  }
}

// ✅ BETTER - Let Resonate auto-generate ID (if external resolver not needed)
function* internalWorkflow(ctx: Context, orderId: string) {
  // No explicit ID = Resonate generates deterministic ID from call tree
  const promise = yield* ctx.promise<Decision>();
  const result = yield* promise;
  return result;
}
```

**Why this matters:** When a workflow replays from a checkpoint, it re-executes the same code. If you use `Date.now()` or `Math.random()` in a promise ID, the ID will be different on replay, and Resonate won't find the resolved promise, causing the workflow to create a new promise instead of resuming from the resolved one.

**Pro tip:** Auto-generated IDs (omitting `id` field) are always deterministic and replay-safe. Only use explicit IDs when you need to communicate the ID to an external system.

**Base64 Encoding:** The Resonate server expects base64-encoded data. Encode at the API boundary (CLI/webhook), and the SDK automatically decodes it in your workflow.

## Deterministic Helpers

### ctx.date.now()

```ts
function* workflow(ctx: Context) {
  // ✅ CORRECT - Deterministic time
  const timestamp = yield* ctx.date.now();

  // ❌ WRONG - Non-deterministic
  const timestamp = Date.now();  // Different on replay!
}
```

### ctx.math.random()

```ts
function* workflow(ctx: Context) {
  // ✅ CORRECT - Deterministic random
  const rand = yield* ctx.math.random();

  // ❌ WRONG - Non-deterministic
  const rand = Math.random();  // Different on replay!
}
```

**Why:** Replay must follow same code path. Non-deterministic values break recovery.

## Options Pattern

```ts
function* workflow(ctx: Context) {
  const result = yield* ctx.run(
    slowOperation,
    "arg",
    ctx.options({
      id: "custom-promise-id",
      timeout: 30_000,  // 30 seconds
      tags: { operation: "slow" }
    })
  );
}
```

## Structured Concurrency

**All started operations are implicitly awaited before return.**

```ts
function* workflow(ctx: Context) {
  yield* ctx.beginRun(task1);  // Started
  yield* ctx.beginRun(task2);  // Started
  return "done";  // Implicitly waits for task1 and task2
}
```

**Equivalent to:**

```ts
function* workflow(ctx: Context) {
  const f1 = yield* ctx.beginRun(task1);
  const f2 = yield* ctx.beginRun(task2);
  yield* f1;  // Explicit wait
  yield* f2;  // Explicit wait
  return "done";
}
```

**No orphaned work.**

## Error Handling

### Regular Errors

```ts
function* workflow(ctx: Context, userId: string) {
  try {
    const user = yield* ctx.run(fetchUser, userId);
    return user;
  } catch (error) {
    console.error("Failed to fetch user:", error);
    // Retry logic or compensation
    return null;
  }
}

async function fetchUser(_ctx: Context, userId: string) {
  if (!userId) {
    throw new Error("User ID required");
  }
  // Fetch logic
}
```

### Invariant Violations

```ts
function* workflow(ctx: Context, amount: number) {
  // Assert fails if condition is false
  yield* ctx.assert(amount > 0, "Amount must be positive");

  // Panic fails if condition is true
  yield* ctx.panic(amount > 1000000, "Amount exceeds limit");

  return amount;
}
```

**Use:** `assert` for preconditions, `panic` for invariant violations

## Complete Example: Order Processing

```ts
import { type Context } from "@resonatehq/sdk";

// Main workflow (durable)
function* processOrder(ctx: Context, orderId: string) {
  // 1. Fetch order data
  const order = yield* ctx.run(fetchOrder, orderId);

  // 2. Parallel validation
  const inventoryFuture = yield* ctx.beginRun(checkInventory, order);
  const fraudFuture = yield* ctx.beginRpc(
    "checkFraud",
    order,
    ctx.options({ target: "poll://any@fraud-workers" })
  );

  const inventory = yield* inventoryFuture;
  const fraud = yield* fraudFuture;

  if (!inventory.available || fraud.flagged) {
    return { status: "rejected", reason: !inventory.available ? "out of stock" : "fraud" };
  }

  // 3. Charge customer
  const payment = yield* ctx.run(chargeCard, order);

  // 4. Create shipment
  const shipment = yield* ctx.run(createShipment, order);

  // 5. Send confirmation (fire and forget)
  yield* ctx.detached(
    "sendConfirmation",
    orderId,
    ctx.options({ target: "poll://any@email-workers" })
  );

  return { status: "complete", payment, shipment };
}

// Helper functions (async, not generators)
async function fetchOrder(_ctx: Context, orderId: string) {
  // DB fetch
  return { id: orderId, items: [...], total: 99.99 };
}

async function checkInventory(_ctx: Context, order: any) {
  // Inventory check
  return { available: true };
}

async function chargeCard(_ctx: Context, order: any) {
  // Payment processing
  return { transactionId: "txn_123", amount: order.total };
}

async function createShipment(_ctx: Context, order: any) {
  // Shipping API
  return { trackingNumber: "1Z999AA1", carrier: "UPS" };
}
```

## Common Pitfalls

### 1. Missing yield*

```ts
// ❌ WRONG
function* workflow(ctx: Context) {
  const result = ctx.run(someFunc);  // Returns LFC object, not result!
}

// ✅ CORRECT
function* workflow(ctx: Context) {
  const result = yield* ctx.run(someFunc);
}
```

### 2. Using await Instead of yield*

```ts
// ❌ WRONG
function* workflow(ctx: Context) {
  const result = await ctx.run(someFunc);  // Breaks determinism!
}

// ✅ CORRECT
function* workflow(ctx: Context) {
  const result = yield* ctx.run(someFunc);
}
```

### 3. Side Effects Outside ctx.run

```ts
// ❌ WRONG - Replays will re-execute
function* workflow(ctx: Context) {
  console.log("Processing...");  // Logs multiple times on replay!
  await emailService.send("test@example.com");  // Sends multiple emails!
}

// ✅ CORRECT
function* workflow(ctx: Context) {
  yield* ctx.run(logMessage, "Processing...");
  yield* ctx.run(sendEmail, "test@example.com");
}
```

### 4. Non-Deterministic Operations

```ts
// ❌ WRONG
function* workflow(ctx: Context) {
  const timestamp = Date.now();  // Different on replay!
  const rand = Math.random();     // Different on replay!
}

// ✅ CORRECT
function* workflow(ctx: Context) {
  const timestamp = yield* ctx.date.now();
  const rand = yield* ctx.math.random();
}
```

### 5. Passing Non-Serializable Data to RPC

```ts
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db");

  // ❌ WRONG - DB connection can't serialize
  yield* ctx.rpc("processData", db);

  // ✅ CORRECT - Fetch data first, then pass it
  const data = yield* ctx.run(fetchData, db);
  yield* ctx.rpc("processData", data);
}
```

## Decision Tree

**What do I need?**
- Same process, wait for result → `ctx.run()`
- Same process, start now, wait later → `ctx.beginRun()`
- Different process, wait for result → `ctx.rpc()`
- Different process, start now, wait later → `ctx.beginRpc()`
- Fire and forget → `ctx.detached()`
- Wait for time → `ctx.sleep()`
- Wait for human/webhook → `ctx.promise()`
- Deterministic time → `ctx.date.now()`
- Deterministic random → `ctx.math.random()`
- Shared resource → `ctx.getDependency()`

## Summary

The Durable World is your **execution plane**:
- Generator functions with `yield*`
- Every operation checkpoints
- Automatic recovery on failure
- Clear serialization boundaries
- Structured concurrency guarantees

Write sequential-looking code that runs distributed and durable.
