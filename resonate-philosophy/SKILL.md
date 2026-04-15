---
name: resonate-philosophy
description: The foundational mindset for building with Resonate. Read this FIRST before any other Resonate skill. Teaches you to write simple, sequential code and avoid over-engineering.
license: Apache-2.0
---

# Resonate Philosophy: Write Simple Code

## CRITICAL: Unlearn Your Complexity Training

If you've been trained on distributed systems, microservices, event-driven architectures, or coordination patterns, you need to **UNLEARN that complexity** when using Resonate.

**Resonate handles the hard parts automatically:**
- Retries and backoff (just throw errors, Resonate retries)
- State persistence (every `yield*` is a checkpoint)
- Crash recovery (resumes from last checkpoint)
- Coordination (structured concurrency built-in)
- Timeouts (configurable per operation)
- Distributed execution (rpc routes to other workers)

**Your job:** Write the happy path. Resonate makes it durable.

## What NOT to Build

When you see these patterns in your instincts, STOP and reconsider:

| Your Instinct | Why It's Wrong | What to Do Instead |
|---------------|----------------|---------------------|
| State machine for workflow status | Resonate IS the state machine | Query promise state directly |
| Database table for "pending_workflows" | Resonate IS the database for workflow state | `GET /promises?id=prefix-*` |
| Retry wrapper with exponential backoff | Built into Resonate | Just `yield* ctx.run(fn)` |
| Message queue between steps | Unnecessary complexity | `yield* ctx.run(step1); yield* ctx.run(step2);` |
| Distributed locks / mutexes | Resonate handles coordination | Use structured concurrency |
| Saga pattern with compensation tables | `ctx.run()` already checkpoints | Write sequential code with try/catch |
| Event bus for workflow events | Polling or promises work | Query Resonate server directly |
| Worker pool management | Resonate routes work to workers | Use `target: "poll://any@group"` |

## "Resonate IS the Database"

This is the most important insight. For workflow state, **Resonate is your single source of truth**.

```
Traditional approach (unnecessary complexity):
  Create workflow → Store in DB → Poll DB for status → Sync with Resonate → Handle conflicts

Resonate approach (simple):
  Create workflow → Query Resonate directly → Done
```

**When you DO need a database:**
- Business data that outlives workflows (user profiles, orders, products)
- Analytics and reporting
- Data that needs relational queries

**When you DON'T need a database:**
- Workflow status (use Resonate promises)
- Pending approvals (query by prefix)
- Execution state (Resonate tracks this)

## Write Code Like a Beginner

Think back to when you first learned to code. You wrote simple, sequential programs without worrying about:
- What happens if the process crashes
- How to coordinate between services
- Retry logic for network failures
- State persistence

**That's exactly what Resonate wants.**

### The Correct Level of Complexity

```ts
// This IS the production-ready code. No additions needed.
function* processOrder(ctx: Context, orderId: string) {
  const order = yield* ctx.run(fetchOrder, orderId);
  const payment = yield* ctx.run(chargeCard, order);
  const shipment = yield* ctx.run(createShipment, order);
  yield* ctx.run(sendConfirmationEmail, order.email);
  return { payment, shipment };
}
```

**What this code does:**
- Fetches order (checkpointed - won't re-fetch on replay)
- Charges card (checkpointed - won't double-charge on replay)
- Creates shipment (checkpointed - won't duplicate on replay)
- Sends email (checkpointed - won't re-send on replay)
- Returns result (final checkpoint)

**What you DON'T need to add:**
- ❌ try/catch around each step (Resonate retries automatically)
- ❌ Transaction wrapper (each step is its own checkpoint)
- ❌ Idempotency keys (Resonate handles replay)
- ❌ Status tracking table (query Resonate)
- ❌ Error logging service (Resonate tracks failures)

## The Generator Pattern is Your Friend

Generators (`function*` with `yield*`) might look unfamiliar, but they're just sequential code with pause points:

```ts
function* workflow(ctx: Context) {
  const a = yield* ctx.run(step1);  // Pause point 1
  const b = yield* ctx.run(step2);  // Pause point 2
  const c = yield* ctx.run(step3);  // Pause point 3
  return c;
}
```

**Mental model:** Each `yield*` is a save point in a video game. If the game crashes, you resume from the last save, not from the beginning.

## Common Over-Engineering Mistakes

### Mistake 1: Building a Status Dashboard with a Database

```ts
// ❌ OVER-ENGINEERED
async function startWorkflow(orderId: string) {
  const workflowId = uuid();

  // Insert into database
  await db.insert("workflows", {
    id: workflowId,
    status: "pending",
    created_at: new Date()
  });

  // Start Resonate workflow
  await resonate.run(workflowId, processOrder, orderId);

  // Update status
  await db.update("workflows", workflowId, { status: "running" });
}

// ✅ SIMPLE - Use Resonate as the source of truth
async function startWorkflow(orderId: string) {
  const workflowId = `order-${orderId}`;
  await resonate.run(workflowId, processOrder, orderId);
  return workflowId;
}

// Dashboard queries Resonate directly
async function listPendingOrders() {
  const response = await fetch(`${RESONATE_URL}/promises?id=order-*&state=pending`);
  return response.json();
}
```

### Mistake 2: Adding Retry Logic

```ts
// ❌ OVER-ENGINEERED
function* workflow(ctx: Context, data: any) {
  let attempts = 0;
  while (attempts < 3) {
    try {
      return yield* ctx.run(callExternalAPI, data);
    } catch (e) {
      attempts++;
      if (attempts >= 3) throw e;
      yield* ctx.sleep(Math.pow(2, attempts) * 1000);
    }
  }
}

// ✅ SIMPLE - Resonate handles retries
function* workflow(ctx: Context, data: any) {
  return yield* ctx.run(callExternalAPI, data);
}
```

### Mistake 3: Event-Driven Workflow Status

```ts
// ❌ OVER-ENGINEERED
function* workflow(ctx: Context, orderId: string) {
  await eventBus.emit("workflow.started", { orderId });

  const result = yield* ctx.run(processOrder, orderId);

  await eventBus.emit("workflow.completed", { orderId, result });
  return result;
}

// ✅ SIMPLE - Query workflow state when needed
function* workflow(ctx: Context, orderId: string) {
  return yield* ctx.run(processOrder, orderId);
}

// UI polls or uses Resonate's promise state
```

## When to Add Complexity

Only add complexity when you have a **concrete, present need**:

| Need | Solution |
|------|----------|
| Human must approve before continuing | `yield* ctx.promise()` |
| Need to wait for external webhook | `yield* ctx.promise()` |
| Need to run multiple things in parallel | `yield* ctx.beginRun()` then `yield* future` |
| Need to call a different service | `yield* ctx.rpc()` |
| Need to wait for a specific time | `yield* ctx.sleep()` |

## Summary: The Resonate Mindset

1. **Write sequential code** - Just describe what should happen, step by step
2. **Trust the checkpoints** - Every `yield*` is a durable save point
3. **Don't build infrastructure** - Resonate IS the infrastructure
4. **Query Resonate for state** - Don't duplicate state in a database
5. **Let errors propagate** - Resonate handles retries
6. **Start simple** - Add complexity only when you hit a real limitation

**The goal:** Code that looks like it couldn't possibly work in production... but does.
