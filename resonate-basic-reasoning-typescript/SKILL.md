---
name: resonate-basic-reasoning-typescript
description: Decision framework for when to use Resonate, which APIs to choose, and how to structure applications. Use this before writing code to determine the right approach for your use case.
---

# Resonate Basic Reasoning

## Overview

This skill helps you reason about:
1. **When** to use Resonate
2. **Which** APIs to use (Client vs Context, run vs rpc, etc.)
3. **How** to structure your application
4. **Why** certain patterns exist

## When to Use Resonate

### Use Resonate When Your Process Involves ANY Of:

#### 1. Duration Beyond Timeout

```
Your process: Transcoding a 2-hour video
Timeout: 60 seconds

❌ Regular function: Times out
✅ Resonate: Spans multiple activations
```

**Examples:**
- Long-running batch jobs
- Multi-step data processing
- Video/image processing
- Report generation

#### 2. Steps with Side Effects

```
Your process:
  1. Charge credit card ✓
  2. [CRASH]
  3. Send confirmation email (needs to happen)
  4. Update inventory (needs to happen)

❌ Regular function: Side effects lost
✅ Resonate: Resumes from checkpoint
```

**Examples:**
- Payment processing
- Order fulfillment
- Account provisioning
- Multi-step integrations

#### 3. Waiting for External Events

```
Your process:
  1. Submit job to API
  2. Wait for webhook callback
  3. Process results

❌ Regular function: Can't wait across activations
✅ Resonate: Suspends and resumes on webhook
```

**Examples:**
- Human approvals
- External API callbacks
- Scheduled delays
- Event-driven workflows

#### 4. Reliability Requirements

```
Your process: Must complete even if:
  - Process crashes
  - Server restarts
  - Network fails temporarily

❌ Regular function: Lost on failure
✅ Resonate: Recovers and continues
```

**Examples:**
- Critical business processes
- Financial transactions
- Compliance workflows
- SLA-bound operations

#### 5. Coordination Across Services

```
Your process:
  1. Service A: Validate order
  2. Service B: Check inventory
  3. Service C: Process payment
  4. Service D: Create shipment

❌ Regular functions: Complex orchestration code
✅ Resonate: Distributed coordination built-in
```

**Examples:**
- Microservice choreography
- Multi-tenant operations
- Distributed transactions
- Cross-team workflows

### Don't Use Resonate When:

#### Simple Request/Response

```
Your process:
  1. Fetch user by ID
  2. Return user

Duration: 100ms
Complexity: Low

✅ Regular function: Perfect fit
❌ Resonate: Unnecessary overhead
```

#### Pure Computation

```
Your process:
  1. Calculate hash of string
  2. Return hash

State: None
Side effects: None

✅ Regular function: Fast and simple
❌ Resonate: No benefit
```

#### Real-Time Requirements

```
Your process: Sub-millisecond latency required

❌ Resonate: Checkpointing adds latency
✅ In-memory: Use regular functions
```

## Which API to Use

### Decision Tree: Ephemeral vs Durable

```
Where am I writing code?
│
├─ Application entry point (main, CLI, HTTP handler)
│  └─ Use: Resonate Client (ephemeral world)
│     APIs: resonate.run(), resonate.rpc(), resonate.register()
│
└─ Inside function* with Context parameter
   └─ Use: Context APIs (durable world)
      APIs: ctx.run(), ctx.rpc(), ctx.promise()
```

### Decision Tree: run() vs rpc()

```
Where should the function execute?
│
├─ Same process as caller
│  └─ Do I need the result immediately?
│     ├─ Yes: ctx.run() or resonate.run()
│     └─ No: ctx.beginRun() or resonate.beginRun()
│
└─ Different process/group
   └─ Do I need the result immediately?
      ├─ Yes: ctx.rpc() or resonate.rpc()
      ├─ No: ctx.beginRpc() or resonate.beginRpc()
      └─ Never: ctx.detached()
```

### Why Use run()?

**Use `ctx.run()` when:**
- Function is lightweight (milliseconds)
- No benefit to separate process
- Want simpler code (no networking)
- Need non-serializable arguments

```ts
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db");

  // Quick DB query - use run()
  const user = yield* ctx.run(fetchUser, db, "123");

  // Simple validation - use run()
  const valid = yield* ctx.run(validateUser, user);

  return { user, valid };
}
```

### Why Use rpc()?

**Use `ctx.rpc()` when:**
- Function is expensive (seconds/minutes)
- Benefits from dedicated resources (GPU, lots of RAM)
- Enables horizontal scaling
- Different failure domain desired

```ts
function* workflow(ctx: Context, userId: string) {
  // CPU-intensive ML inference - use rpc()
  const prediction = yield* ctx.rpc(
    "runModel",
    userId,
    ctx.options({ target: "poll://any@gpu-workers" })
  );

  // Quick local work - use run()
  const formatted = yield* ctx.run(formatResults, prediction);

  return formatted;
}
```

### Why Use beginRun/beginRpc()?

**Use `begin*` variants for concurrency:**

```ts
function* workflow(ctx: Context, userId: string) {
  // All independent - run concurrently
  const profileFuture = yield* ctx.beginRun(fetchProfile, userId);
  const ordersFuture = yield* ctx.beginRun(fetchOrders, userId);
  const invoicesFuture = yield* ctx.beginRun(fetchInvoices, userId);

  // Wait for all
  const profile = yield* profileFuture;
  const orders = yield* ordersFuture;
  const invoices = yield* invoicesFuture;

  return { profile, orders, invoices };
}
```

**Pattern recognition:**
- Multiple independent operations → `begin*`
- Operations depend on each other → regular `run/rpc`

### Why Use detached()?

**Use `ctx.detached()` for fire-and-forget:**

```ts
function* workflow(ctx: Context, orderId: string) {
  // Critical: Process order
  const result = yield* ctx.run(processOrder, orderId);

  // Non-critical: Track analytics
  yield* ctx.detached(
    "trackOrderMetrics",
    orderId,
    ctx.options({ target: "poll://any@analytics" })
  );

  // Return immediately, don't wait for analytics
  return result;
}
```

**Use when:**
- Operation is nice-to-have, not required
- Don't care about failure
- Want to minimize latency

## Serialization Boundaries

### Visual Guide

```
┌─────────────────────────────────────────┐
│          Process A                      │
│                                         │
│  ctx.run(func, dbConnection, data)     │
│            ↓                            │
│            ✓ dbConnection (in-memory)  │
│            ✓ data (in-memory)          │
│            ↓                            │
│         func executes                   │
│            ↓                            │
│         return result                   │
│            ↓                            │
│         ✓ result (MUST serialize!)     │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│          Process A                      │
│                                         │
│  ctx.rpc("func", dbConnection, data)   │
│            ↓                            │
│            ✗ dbConnection (can't cross) │
│            ✓ data (MUST serialize!)    │
└─────────────────────────────────────────┘
               │
               ↓ [Network]
               │
┌─────────────────────────────────────────┐
│          Process B                      │
│                                         │
│         func receives data              │
│            ↓                            │
│         func executes                   │
│            ↓                            │
│         return result                   │
│            ↓                            │
│         ✓ result (MUST serialize!)     │
└─────────────────────────────────────────┘
```

### Serialization Rules Summary

| API | Arguments | Return Value |
|-----|-----------|--------------|
| `ctx.run()` | In-memory (any) | Must serialize |
| `ctx.rpc()` | Must serialize | Must serialize |
| `ctx.beginRun()` | In-memory (any) | Must serialize |
| `ctx.beginRpc()` | Must serialize | Must serialize |
| `resonate.run()` | In-memory (any) | Must serialize |
| `resonate.rpc()` | Must serialize | Must serialize |

### Common Mistakes

```ts
// ❌ WRONG - Passing DB connection to RPC
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db");
  yield* ctx.rpc("processData", db);  // Error: Can't serialize
}

// ✅ CORRECT - Fetch first, then pass data
function* workflow(ctx: Context) {
  const db = ctx.getDependency("db");
  const data = yield* ctx.run(fetchData, db);  // Local call OK
  yield* ctx.rpc("processData", data);  // Pass serializable data
}
```

## Application Structure Patterns

### Pattern 1: Single Process (All Local)

```
┌─────────────────────────────────────┐
│        main.ts (ephemeral)          │
│                                     │
│  resonate.register(workflow)        │
│  resonate.run("id", workflow)       │
└─────────────────────────────────────┘
               ↓
┌─────────────────────────────────────┐
│      workflow.ts (durable)          │
│                                     │
│  function* workflow(ctx)            │
│    ctx.run(step1)                   │
│    ctx.run(step2)                   │
│    ctx.run(step3)                   │
└─────────────────────────────────────┘
```

**When to use:**
- Development/testing
- Simple applications
- All operations fast enough for one process

### Pattern 2: Gateway + Workers (Distributed)

```
┌─────────────────────────────────────┐
│     gateway.ts (ephemeral)          │
│                                     │
│  POST /jobs → resonate.beginRpc()   │
│  GET /jobs/:id → resonate.get()     │
└─────────────────────────────────────┘
               ↓ RPC
┌─────────────────────────────────────┐
│     worker.ts (ephemeral)           │
│                                     │
│  resonate.register(workflow)        │
│  [polls for work]                   │
└─────────────────────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     workflow.ts (durable)           │
│                                     │
│  function* workflow(ctx)            │
│    ctx.run(localStep)               │
│    ctx.rpc("remoteStep")            │
└─────────────────────────────────────┘
```

**When to use:**
- HTTP API + background processing
- Scaling workers independently
- Separating API concerns from work

### Pattern 3: Specialized Workers (Multi-Group)

```
┌─────────────────────────────────────┐
│   main-worker (group: "main")       │
│                                     │
│  function* mainWorkflow(ctx)        │
│    ctx.rpc("mlTask",                │
│      options({ target:              │
│        "poll://any@ml-workers" })   │
│    )                                │
│    ctx.rpc("dbTask",                │
│      options({ target:              │
│        "poll://any@db-workers" })   │
│    )                                │
└─────────────────────────────────────┘
         ↓                    ↓
┌──────────────┐      ┌──────────────┐
│  ml-worker   │      │  db-worker   │
│  (group:     │      │  (group:     │
│   "ml-...")  │      │   "db-...")  │
│              │      │              │
│  [GPU box]   │      │  [DB box]    │
└──────────────┘      └──────────────┘
```

**When to use:**
- Different resource requirements (GPU vs CPU)
- Different scaling profiles
- Fault isolation

## Common Scenarios

### Scenario: REST API with Background Work

**Problem:** User submits job, you process it async, they poll for results

**Solution:**

```ts
// gateway.ts (ephemeral)
app.post("/jobs", async (req, res) => {
  const jobId = `job/${crypto.randomUUID()}`;

  await resonate.beginRpc(
    jobId,
    "processJob",
    req.body,
    resonate.options({ target: "poll://any@workers" })
  );

  res.status(202).json({ jobId });
});

app.get("/jobs/:id", async (req, res) => {
  try {
    const handle = await resonate.get(req.params.id);
    const done = await handle.done();

    if (done) {
      const result = await handle.result();
      res.json({ status: "complete", result });
    } else {
      res.json({ status: "processing" });
    }
  } catch (err) {
    res.status(404).json({ error: "Job not found" });
  }
});

// worker.ts (ephemeral)
resonate.register(processJob);

// workflow.ts (durable)
function* processJob(ctx: Context, input: any) {
  const validated = yield* ctx.run(validate, input);
  const processed = yield* ctx.run(process, validated);
  const stored = yield* ctx.run(store, processed);
  return stored;
}
```

### Scenario: Human-in-the-Loop Approval

**Problem:** Workflow needs human decision before proceeding

**Solution:**

```ts
// workflow.ts (durable)
function* expenseApproval(ctx: Context, expense: any) {
  // Create promise for approval
  const approvalPromise = yield* ctx.promise({
    id: `approval/${expense.id}`
  });

  // Notify approver
  yield* ctx.run(sendApprovalEmail, approvalPromise.id, expense);

  // Block until approved
  const decision = yield* approvalPromise;

  if (decision.approved) {
    yield* ctx.run(processExpense, expense);
    return { status: "approved", processed: true };
  } else {
    return { status: "rejected", reason: decision.reason };
  }
}

// webhook.ts (ephemeral)
app.post("/approve/:promiseId", async (req, res) => {
  await resonate.promises.resolve(req.params.promiseId, {
    approved: req.body.approved,
    reason: req.body.reason,
    approver: req.body.userId
  });

  res.json({ status: "ok" });
});
```

### Scenario: Fan-Out/Fan-In

**Problem:** Process multiple items concurrently, wait for all

**Solution:**

```ts
function* processOrders(ctx: Context, orderIds: string[]) {
  // Fan-out: Start all orders
  const futures = [];
  for (const orderId of orderIds) {
    const future = yield* ctx.beginRun(processOrder, orderId);
    futures.push(future);
  }

  // Fan-in: Wait for all
  const results = [];
  for (const future of futures) {
    const result = yield* future;
    results.push(result);
  }

  return results;
}
```

### Scenario: Retry with Backoff

**Problem:** External API sometimes fails, need automatic retries

**Solution:**

```ts
// Resonate handles retries automatically!
function* callExternalAPI(ctx: Context, data: any) {
  // Will retry with exponential backoff on failure
  const result = yield* ctx.run(
    makeAPICall,
    data,
    ctx.options({
      timeout: 30_000,
      // Built-in retry policy handles backoff
    })
  );

  return result;
}

async function makeAPICall(_ctx: Context, data: any) {
  const response = await fetch("https://api.example.com/data", {
    method: "POST",
    body: JSON.stringify(data)
  });

  if (!response.ok) {
    throw new Error(`API error: ${response.status}`);
  }

  return await response.json();
}
```

## Debugging Strategy

### Check 1: Are you in the right world?

```
Error: "ctx is not defined"
→ You're in ephemeral world, use resonate.run()

Error: "resonate is not defined"
→ You're in durable world, use ctx.run()
```

### Check 2: Are you using yield* correctly?

```
Result is LFC/RFI/etc object, not the actual value
→ Add yield* before ctx.run()

Error: "Cannot read property..."
→ Missing yield*, got promise-like object instead
```

### Check 3: Serialization error?

```
Error: "Cannot serialize..."
→ Check if passing DB connections, functions, etc. to rpc()
→ Solution: Fetch data first, then pass serializable data
```

### Check 4: Registration issue?

```
Error: "Function not found"
→ Check if function is registered
→ Check if using correct function name (not custom string)
→ Check if target group is correct
```

### Check 5: Array wrapping in local mode?

```
State/args are undefined or wrong type
→ Check if using new Resonate() without URL (in-memory mode)
→ Solution: Extract from array: Array.isArray(state) ? state[0] : state
```

## Summary Decision Framework

**Start here:**

1. **Do I need Resonate?**
   - Long-running? → Yes
   - Side effects that must complete? → Yes
   - Need to wait for external events? → Yes
   - Simple request/response? → No

2. **Where am I writing code?**
   - Entry point / HTTP handler → Ephemeral (Client APIs)
   - Inside function* → Durable (Context APIs)

3. **Local or remote execution?**
   - Same process → run/beginRun
   - Different process → rpc/beginRpc
   - Don't care about result → detached

4. **Sequential or concurrent?**
   - One after another → run/rpc
   - Multiple at once → beginRun/beginRpc

5. **What can I pass as arguments?**
   - To run: Anything
   - To rpc: Only serializable data

6. **Do I need to wait?**
   - For time → ctx.sleep()
   - For human/webhook → ctx.promise()
   - For function → ctx.run/rpc()

**Remember:**
- Every `yield*` is a checkpoint
- All return values must serialize
- Structured concurrency: no orphans
- Ephemeral = control, Durable = execution
