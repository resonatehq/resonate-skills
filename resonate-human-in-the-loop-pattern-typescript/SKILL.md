---
name: resonate-human-in-the-loop-pattern-typescript
description: Implement human-in-the-loop workflows where durable functions pause for human decisions, approvals, or reviews. Use this pattern for approval gates, manual review workflows, and human-assisted processes.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern (TypeScript)

> **SDK version:** This skill reflects `@resonatehq/sdk` v0.11.4 (current on npm).

## Overview

The Human-in-the-Loop (HITL) pattern enables workflows to pause execution and wait for human input—decisions, approvals, reviews, or interventions. The workflow suspends (not blocks resources) and resumes exactly where it left off when the human responds, whether that's seconds, hours, or days later.

**Core mechanism:** Create a durable promise, read the ID the SDK generates for it, communicate that ID to a human (via email, UI, webhook), and `yield*` await the promise until it's externally resolved.

## Mental Model

```
Workflow                          Human
   │                                │
   ├─ Create promise, read its ID  │
   ├─ Send email with links        │
   │  (accept_link, reject_link)   │
   │                                │
   ├─ yield* promise               │
   │  [SUSPENDED - not consuming    │
   │   resources, durable state]    │
   │                                │
   │                                ├─ Click "Approve"
   │                                ├─ HTTP POST resolves promise
   │  [RESUMES from checkpoint]    │
   │                                │
   ├─ Process approval decision
   └─ Complete workflow
```

## Core Pattern

### Step 1: Create the Durable Promise

```typescript
function* approvalWorkflow(ctx: Context, orderId: string) {
  // ctx.promise() generates the ID itself — you don't choose one
  const approvalPromise = yield* ctx.promise<Decision>({
    timeout: 24 * 60 * 60 * 1000  // 24 hours
  });

  // Continue...
}
```

**Why read the ID back?** The human (or webhook) needs to know which promise to resolve, and the SDK's auto-generated ID is the only ID there is. It's deterministic across replay — the sequence advances in call order — so `approvalPromise.id` is safe to hand to anything outside the workflow.

### Step 2: Communicate Promise ID

```typescript
function* approvalWorkflow(ctx: Context, orderId: string) {
  const approvalPromise = yield* ctx.promise<Decision>();

  // Send email with accept/reject links containing promise ID
  yield* ctx.run(sendApprovalEmail, orderId, approvalPromise.id);

  // Continue...
}

async function sendApprovalEmail(_ctx: Context, orderId: string, promiseId: string) {
  const acceptLink = `https://example.com/approve/${promiseId}?action=accept`;
  const rejectLink = `https://example.com/approve/${promiseId}?action=reject`;

  await emailService.send({
    to: "manager@example.com",
    subject: `Approval needed for order ${orderId}`,
    body: `Accept: ${acceptLink}\nReject: ${rejectLink}`
  });
}
```

**Alternative:** Store promise ID in database for UI-based workflows.

### Step 3: Await Promise

```typescript
function* approvalWorkflow(ctx: Context, orderId: string) {
  const approvalPromise = yield* ctx.promise<Decision>({
    timeout: 24 * 60 * 60 * 1000
  });

  yield* ctx.run(sendApprovalEmail, orderId, approvalPromise.id);

  // SUSPEND HERE - workflow pauses until promise resolves
  const decision = yield* approvalPromise;

  // RESUMES HERE when human responds
  if (decision.approved) {
    yield* ctx.run(processOrder, orderId);
    return { status: "approved", orderId };
  } else {
    yield* ctx.run(cancelOrder, orderId);
    return { status: "rejected", orderId, reason: decision.reason };
  }
}
```

### Step 4: External Resolution (Ephemeral World)

```typescript
// In Express route handler or webhook
app.post("/approve/:promiseId", async (req, res) => {
  const { promiseId } = req.params;
  const { action } = req.query;

  const decision = {
    approved: action === "accept",
    timestamp: Date.now(),
    approver: req.user?.email
  };

  // CRITICAL: Base64 encode data for Resonate server
  const encodedData = Buffer.from(JSON.stringify(decision)).toString('base64');

  await resonate.promises.resolve(promiseId, {
    data: encodedData,
  });

  res.json({ status: "recorded" });
});
```

**Note:** The Resonate server expects base64-encoded data. The SDK automatically decodes it when the workflow receives it.

## Complete Example: Order Approval

```typescript
import { Resonate, type Context } from "@resonatehq/sdk";
import express from "express";

const resonate = new Resonate({
  url: "http://localhost:8001",
  group: "workflows"
});

// Workflow: Create order and await approval
function* createOrderWithApproval(ctx: Context, orderData: any) {
  // 1. Create order record
  const order = yield* ctx.run(createOrderRecord, orderData);

  // 2. Create approval promise — the SDK generates the ID
  const approvalPromise = yield* ctx.promise<ApprovalDecision>({
    timeout: 48 * 60 * 60 * 1000  // 48 hours
  });

  // 3. Send approval request
  yield* ctx.run(sendApprovalRequest, order, approvalPromise.id);

  // 4. SUSPEND and wait for human decision
  try {
    const decision = yield* approvalPromise;

    // 5. Process based on decision
    if (decision.approved) {
      yield* ctx.run(chargePayment, order);
      yield* ctx.run(createShipment, order);
      yield* ctx.run(sendConfirmation, order, decision.approver);
      return { status: "approved", order };
    } else {
      yield* ctx.run(cancelOrder, order);
      yield* ctx.run(sendRejectionNotice, order, decision.reason);
      return { status: "rejected", order, reason: decision.reason };
    }
  } catch (error) {
    // Promise timed out or was rejected
    yield* ctx.run(expireOrder, order);
    return { status: "expired", order };
  }
}

// Helper functions
async function createOrderRecord(_ctx: Context, data: any) {
  // Create DB record
  return { id: `order-${Date.now()}`, ...data, status: "pending" };
}

async function sendApprovalRequest(_ctx: Context, order: any, promiseId: string) {
  const acceptLink = `http://localhost:3000/approve/${promiseId}?action=accept`;
  const rejectLink = `http://localhost:3000/approve/${promiseId}?action=reject`;

  await emailService.send({
    to: "approver@example.com",
    subject: `Order approval needed: ${order.id}`,
    html: `
      <p>Order ${order.id} requires approval.</p>
      <p>Amount: $${order.total}</p>
      <p><a href="${acceptLink}">Approve</a> | <a href="${rejectLink}">Reject</a></p>
    `
  });
}

// Express routes for human interaction
const app = express();

app.post("/orders", async (req, res) => {
  const orderId = `order-${Date.now()}`;

  await resonate.beginRun(
    orderId,
    createOrderWithApproval,
    req.body
  );

  res.status(202).json({ orderId });
});

app.get("/approve/:promiseId", async (req, res) => {
  const { promiseId } = req.params;
  const { action } = req.query;

  const decision = {
    approved: action === "accept",
    approver: "manager@example.com",
    timestamp: Date.now(),
    reason: action === "reject" ? "Budget exceeded" : null
  };

  const encoded = Buffer.from(JSON.stringify(decision)).toString('base64');

  await resonate.promises.resolve(promiseId, { data: encoded });

  res.send(`Decision recorded: ${action}`);
});

resonate.register(createOrderWithApproval);
app.listen(3000);
```

## Pattern Variants

### Multiple Approvers (Sequential)

```typescript
function* multiStageApproval(ctx: Context, orderId: string) {
  // Stage 1: Manager approval — each ctx.promise() call gets its own auto-generated ID
  const managerPromise = yield* ctx.promise();
  yield* ctx.run(sendManagerApproval, orderId, managerPromise.id);
  const managerDecision = yield* managerPromise;

  if (!managerDecision.approved) {
    return { status: "rejected", stage: "manager" };
  }

  // Stage 2: Finance approval
  const financePromise = yield* ctx.promise();
  yield* ctx.run(sendFinanceApproval, orderId, financePromise.id);
  const financeDecision = yield* financePromise;

  if (!financeDecision.approved) {
    return { status: "rejected", stage: "finance" };
  }

  return { status: "approved", stages: ["manager", "finance"] };
}
```

### Multiple Approvers (Parallel - Any Approve)

```typescript
function* parallelApproval(ctx: Context, orderId: string) {
  // Create one promise per approver — the SDK generates a unique ID for each
  const alice = yield* ctx.promise();
  const bob = yield* ctx.promise();
  const carol = yield* ctx.promise();

  // Pair each approver's name with their promise ID when sending requests,
  // since the ID itself no longer tells you whose approval it is
  yield* ctx.run(sendApprovalRequests, orderId, [
    { approver: "alice", promiseId: alice.id },
    { approver: "bob", promiseId: bob.id },
    { approver: "carol", promiseId: carol.id }
  ]);

  // Race: first to respond wins
  // Note: Resonate doesn't have built-in race() yet, so implement via timeout polling
  const aliceFuture = alice;
  const bobFuture = bob;
  const carolFuture = carol;

  // For now, await first (or implement custom race logic)
  const decision = yield* aliceFuture;

  return { status: decision.approved ? "approved" : "rejected", approver: "alice" };
}
```

### Approval with Retry Loop

```typescript
function* approvalWithRetry(ctx: Context, orderId: string, maxAttempts: number = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    // ctx.promise() at the same point in the sequence gets a distinct,
    // replay-stable ID on every iteration — no manual attempt-numbering needed
    const promise = yield* ctx.promise({
      timeout: 24 * 60 * 60 * 1000
    });

    yield* ctx.run(sendApprovalRequest, orderId, promise.id, attempt);

    try {
      const decision = yield* promise;

      if (decision.approved) {
        return { status: "approved", attempt };
      }
    } catch (error) {
      // Timeout or rejection
      if (attempt === maxAttempts) {
        return { status: "failed", attempts: maxAttempts };
      }
      // Continue to next attempt
    }
  }
}
```

## Worked Example: Multi-Participant Approval

A complete workflow that creates one promise per participant, records their auto-generated IDs, emails each participant an accept/reject link, suspends until every participant responds (or times out), and returns the collective decision:

```typescript
function* approvalWorkflow(ctx: Context, input: ApprovalInput) {
  // Create one promise per participant first — we need their auto-generated
  // IDs before we can create the database record or send the emails
  const promises = [];
  for (const email of input.participant_emails) {
    const handle = yield* ctx.promise({
      timeout: input.participant_timeout_ms,
      tags: { approval_id: input.approval_id, participant_email: email }
    });
    promises.push({ email, handle });
  }

  // Create approval record in database, now that every promise ID is known
  const approval = yield* ctx.rpc("dbCreateApproval", {
    ...input,
    promise_ids: promises.map(p => p.handle.id)
  }, ctx.options({
    target: "poll://any@database-service"
  }));

  // Email each participant their accept/reject link
  for (const { email, handle } of promises) {
    yield* ctx.rpc("sendApprovalEmail", {
      participant_email: email,
      promise_id: handle.id,
      approval_name: input.approval_name
    }, ctx.options({ target: "poll://any@emailer-service" }));
  }

  // SUSPEND: Workflow pauses here for hours/days
  // Participants click links, resolve promises externally
  const results = [];
  for (const { handle } of promises) {
    const decision = yield* handle;
    results.push({ accept: decision?.data?.accept === true });
  }

  // All promises resolved, continue workflow
  const rejected = results.find(r => !r.accept);
  yield* ctx.rpc("dbUpdateApprovalStatus", approval.approval_id, "resolved");

  return rejected
    ? { status: "REJECTED", results }
    : { status: "ACCEPTED", results };
}
```

## Promise IDs Are Auto-Generated

`ctx.promise()` doesn't take an `id` option. The SDK generates one from a per-workflow sequence counter, and that ID is stable across replay because the counter advances in call order — the same call in the same position always produces the same ID:

```typescript
// ❌ WRONG - `id` isn't a valid ctx.promise() option; TypeScript rejects
// the excess property, and if you force it through anyway, it's silently
// discarded at runtime
const promise = yield* ctx.promise({
  id: `approval/${orderId}`
});

// ✅ CORRECT - create the promise, then read the generated ID back off it
const promise = yield* ctx.promise<Decision>();
yield* ctx.run(sendApprovalEmail, orderId, promise.id);
```

If you need a deterministic value for something else in the workflow — a dedupe key, a filename, a random sample — use `ctx.date.now()` and `ctx.math.random()` instead of `Date.now()`/`Math.random()`. Both are recorded on first execution and replayed to the same value, so anything derived from them stays reproducible:

```typescript
// ❌ BAD - not recorded, a different value on every replay
const label = `attempt-${Date.now()}`;

// ✅ GOOD - recorded on first run, identical on replay
const label = `attempt-${yield* ctx.date.now()}`;
```

## Timeout Handling

Always set timeouts for HITL promises to prevent indefinite suspension.

**IMPORTANT:** Timeout values differ between SDK and HTTP API:

| Context | Timeout Format | Example |
|---------|----------------|---------|
| SDK (`ctx.promise()`) | Duration in milliseconds | `48 * 60 * 60 * 1000` (48 hours) |
| HTTP API (`POST /promises`) | Absolute epoch milliseconds | `Date.now() + (48 * 60 * 60 * 1000)` |

```typescript
// SDK usage - duration from now
function* approvalWithTimeout(ctx: Context, orderId: string) {
  const promise = yield* ctx.promise({
    timeout: 48 * 60 * 60 * 1000  // 48 hours (duration)
  });

  yield* ctx.run(sendApprovalRequest, orderId, promise.id);

  try {
    const decision = yield* promise;
    return { status: "approved" };
  } catch (error) {
    // Timeout occurred
    yield* ctx.run(handleTimeout, orderId);
    return { status: "timeout" };
  }
}

// HTTP API usage - absolute timestamp
const response = await fetch(`${RESONATE_URL}/promises`, {
  method: "POST",
  body: JSON.stringify({
    id: `approval/${orderId}`,
    timeout: Date.now() + (48 * 60 * 60 * 1000)  // 48 hours from NOW (absolute)
  })
});
```

## Database Integration

For UI-based workflows, store promise IDs in database:

```typescript
function* uiApprovalWorkflow(ctx: Context, orderId: string) {
  const promise = yield* ctx.promise();

  // Store in database for UI to query
  yield* ctx.run(async () => {
    await db.from("pending_approvals").insert({
      order_id: orderId,
      promise_id: promise.id,
      status: "pending",
      created_at: new Date().toISOString()
    });
  });

  const decision = yield* promise;

  // Update database
  yield* ctx.run(async () => {
    await db.from("pending_approvals")
      .update({ status: "resolved", resolved_at: new Date().toISOString() })
      .eq("promise_id", promise.id);
  });

  return decision;
}
```

## Common Pitfalls

### 1. Forgetting Base64 Encoding

```typescript
// ❌ WRONG - Resonate server expects base64; settle() is private since v0.10.2
await (resonate.promises as any).settle(promiseId, "resolved", { data: { approved: true } });

// ✅ CORRECT
const data = Buffer.from(JSON.stringify({ approved: true })).toString('base64');
await resonate.promises.resolve(promiseId, { data });
```

### 2. Passing an `id` to `ctx.promise()`

```typescript
// ❌ WRONG - `id` isn't a valid option; TypeScript rejects the excess
// property, and even forced through it, the SDK silently discards it and
// generates its own ID anyway
const promise = yield* ctx.promise({
  id: `approval-${orderId}`
});

// ✅ CORRECT - let the SDK generate the ID, then read it back off the promise
const promise = yield* ctx.promise();
yield* ctx.run(notifyApprover, orderId, promise.id);
```

### 3. Missing Timeout

```typescript
// ❌ WRONG - Can hang forever
const promise = yield* ctx.promise();

// ✅ CORRECT
const promise = yield* ctx.promise({
  timeout: 24 * 60 * 60 * 1000
});
```

## Decision Tree

**When to use HITL pattern:**
- Human approval/review required
- Manual intervention needed
- External webhook callback expected
- UI-driven decision workflows
- Compliance/audit trails needed

**When NOT to use:**
- Fully automated decisions
- Time-based triggers (use `ctx.sleep()`)
- Polling external APIs (use regular RPC)

## Summary

The Human-in-the-Loop pattern enables workflows to:
- Pause execution indefinitely without consuming resources
- Resume exactly where they left off when humans respond
- Handle approvals, reviews, and manual interventions naturally
- Scale to thousands of concurrent pending decisions
- Maintain full durability across crashes and restarts

**Core recipe:** Create promise → Read generated ID → Communicate ID → Await promise → Process decision
