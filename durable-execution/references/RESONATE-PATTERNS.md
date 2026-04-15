# Resonate Patterns — Full Implementations

Six patterns with complete Resonate SDK code. Each pattern is self-contained — copy and adapt.

**Prerequisites:** Resonate server running on port 8001, `@resonatehq/sdk` installed. See `RESONATE-QUICKSTART.md`.

---

## 1. Saga — Multi-Step with Compensating Rollbacks

Each step is a durable checkpoint. If any step fails, compensate completed steps in reverse order.

### Basic Saga

```typescript
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001" });

resonate.register("orderSaga", function* (ctx: Context, orderId: string) {
  const completed: string[] = [];

  try {
    yield ctx.run(reserveInventory, orderId);
    completed.push("inventory");

    yield ctx.run(chargePayment, orderId);
    completed.push("payment");

    yield ctx.run(createShipment, orderId);
    completed.push("shipment");

    return { status: "success", orderId };
  } catch (error) {
    // Compensate in reverse order — each compensation is also checkpointed
    for (const step of completed.reverse()) {
      yield ctx.run(compensate, step, orderId);
    }
    return { status: "rolled-back", orderId, compensated: completed };
  }
});

function* compensate(_ctx: Context, step: string, orderId: string) {
  console.log(`Compensating ${step} for ${orderId}`);
  switch (step) {
    case "shipment":  /* cancelShipment(orderId) */  break;
    case "payment":   /* refundPayment(orderId)  */  break;
    case "inventory": /* releaseInventory(orderId) */ break;
  }
  return { compensated: step };
}

function* reserveInventory(_ctx: Context, orderId: string) {
  console.log(`Reserving inventory for ${orderId}`);
  return { reserved: true };
}

function* chargePayment(_ctx: Context, orderId: string) {
  console.log(`Charging payment for ${orderId}`);
  return { txnId: `txn_${orderId}` };
}

function* createShipment(_ctx: Context, orderId: string) {
  console.log(`Creating shipment for ${orderId}`);
  return { tracking: `TRACK_${orderId}` };
}

await resonate.start();
await resonate.invoke("orderSaga", ["order-42"], { id: "order-42" });
await resonate.stop();
```

### Saga with Explicit Step Definitions

For larger sagas, define steps as data:

```typescript
interface SagaStep {
  name: string;
  forward: (ctx: Context, orderId: string) => Generator;
  compensate: (ctx: Context, orderId: string) => Generator;
}

function* runSaga(ctx: Context, steps: SagaStep[], orderId: string) {
  const completed: SagaStep[] = [];

  try {
    for (const step of steps) {
      yield ctx.run(step.forward, orderId);
      completed.push(step);
    }
    return { status: "success" };
  } catch (error) {
    for (const step of completed.reverse()) {
      try {
        yield ctx.run(step.compensate, orderId);
      } catch (compError) {
        console.error(`Compensation failed for ${step.name}:`, compError);
        // Log but continue — other steps still need compensating
      }
    }
    return { status: "rolled-back", compensated: completed.map(s => s.name) };
  }
}
```

### Distributed Saga (Cross-Service)

Same pattern, but use `ctx.rpc()` instead of `ctx.run()` to dispatch steps to different worker groups. See Pattern 6 (Cross-Service RPC) for the full RPC pattern.

**When to use:** Multi-service transactions (payments, bookings, provisioning). Each step must have a compensating action. Prefer orchestration (one coordinator) over choreography (event-driven).

---

## 2. Fan-Out / Fan-In — Parallel Work with Aggregation

Dispatch work to parallel workers via RPC. Each branch runs independently and is independently durable.

### Sequential Fan-Out (Simple)

```typescript
resonate.register("notifyAll", function* (ctx: Context, orderId: string) {
  const channels = ["email", "sms", "slack", "push"];
  const results: Array<{ channel: string; ok: boolean }> = [];

  for (const channel of channels) {
    try {
      const result = yield ctx.rpc(`send-${channel}`, [orderId], {
        target: `poll://any@${channel}-workers`,
      });
      results.push({ channel, ok: true });
    } catch (err) {
      results.push({ channel, ok: false });
    }
  }
  return { orderId, results };
});
```

### Parallel Fan-Out via Worker Groups

Each `ctx.rpc()` suspends the generator. The server-side execution IS parallel — each RPC runs on its own worker. But the generator awaits them in order:

```typescript
resonate.register("batchScore", function* (ctx: Context, items: string[]) {
  const scores: Array<{ item: string; score: number }> = [];

  for (const item of items) {
    // Each RPC dispatches to a pool of scoring workers.
    // The server runs them on separate workers simultaneously.
    // The generator processes results as each RPC completes.
    const score = yield ctx.rpc("computeScore", [item], {
      target: "poll://any@scoring-workers",
    });
    scores.push({ item, score: score as number });
  }

  return { total: scores.length, scores };
});

// Scoring workers — run as many instances as needed
const scorer = new Resonate({ url: "http://localhost:8001", group: "scoring-workers" });
scorer.register("computeScore", function* (_ctx: Context, item: string) {
  // Your scoring logic here
  return Math.random() * 100;
});
await scorer.start();
```

**Limitation:** The SDK processes RPCs sequentially in the generator. True parallel fan-in (`ctx.all([...])`) is not yet available. Workaround: dispatch multiple independent workflows via the gateway, then aggregate externally.

**When to use:** Batch processing, parallel API calls, map-reduce workloads, multi-channel notifications.

---

## 3. Human-in-the-Loop — Suspend for External Signal

The workflow suspends without holding resources. It resumes when a human (or webhook) resolves the promise — seconds, hours, or days later.

### Worker

```typescript
resonate.register("approvalFlow", function* (ctx: Context, orderId: string) {
  const approvalId = `approval-${orderId}`;

  // Step 1: Send notification (durable checkpoint)
  yield ctx.run(function* () {
    console.log(`Approve at: http://localhost:5001/approve?promise_id=${approvalId}`);
    return { notified: true };
  });

  // Step 2: Suspend until human resolves the promise
  const decision = yield ctx.promise(approvalId, {
    timeoutAt: Date.now() + 48 * 60 * 60 * 1000,  // 48 hours
  });

  // Step 3: Process the decision (resumed from checkpoint)
  if (decision === "approved") {
    yield ctx.run(processOrder, orderId);
    return { status: "approved", orderId };
  } else {
    yield ctx.run(cancelOrder, orderId);
    return { status: "rejected", orderId };
  }
});
```

### Gateway (External Resolution)

To resolve the promise, send a `promise.settle` to the Resonate server:

```typescript
// Encode value as base64(utf8(JSON)) — the Resonate wire encoding
function encode(value: unknown): string {
  return btoa(unescape(encodeURIComponent(JSON.stringify(value))));
}

// HTTP handler — resolves the approval promise
async function handleApproval(promiseId: string, decision: string) {
  await fetch("http://localhost:8001", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      kind: "promise.settle",
      head: { corrId: crypto.randomUUID(), version: "" },
      data: {
        id: promiseId,
        state: "resolved",
        value: { headers: {}, data: encode(decision) },
      },
    }),
  });
}
```

### Multi-Stage Approval

Chain multiple `ctx.promise()` calls for sequential gates (manager → finance → legal). Each gate suspends independently. Pattern: `yield ctx.run(notify, ...) → yield ctx.promise(gateId) → check decision → next gate`.

**Promise ID rules:**
- Must be deterministic (derived from workflow inputs, not `Date.now()` or `Math.random()`)
- Must be known to the settling system (include in approval URLs, store in DB)
- Must be unique per approval gate (`approval-${orderId}` not just `approval`)

**When to use:** Approval workflows, manual review gates, external callbacks, payment confirmation, webhook-driven processes.

---

## 4. Durable Sleep / Cron — Timers That Survive Crashes

`ctx.sleep(ms)` creates a durable timer. The process can die and restart — the timer still fires.

### Drip Campaign

```typescript
resonate.register("onboarding", function* (ctx: Context, userId: string) {
  yield ctx.run(sendEmail, userId, "Welcome to the platform!");

  yield ctx.sleep(24 * 60 * 60 * 1000);  // 1 day
  yield ctx.run(sendEmail, userId, "Getting started tips");

  yield ctx.sleep(6 * 24 * 60 * 60 * 1000);  // 6 days
  yield ctx.run(sendEmail, userId, "How are you finding things?");

  yield ctx.sleep(14 * 24 * 60 * 60 * 1000);  // 14 days
  yield ctx.run(sendEmail, userId, "Upgrade to Pro — 20% off this week");

  return { userId, emailsSent: 4 };
});
```

### SLA Reminder

```typescript
resonate.register("slaMonitor", function* (ctx: Context, ticketId: string) {
  yield ctx.run(assignTicket, ticketId);

  // Wait 4 hours for response
  yield ctx.sleep(4 * 60 * 60 * 1000);

  const status = yield ctx.run(checkTicketStatus, ticketId);
  if (status === "open") {
    yield ctx.run(escalateTicket, ticketId);
    yield ctx.run(notifyManager, ticketId);
  }

  return { ticketId, escalated: status === "open" };
});
```

### Retry with Backoff

```typescript
resonate.register("reliableWebhook", function* (ctx: Context, url: string, payload: string) {
  const maxAttempts = 5;
  const baseDelay = 1000;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const result = yield ctx.run(tryWebhook, url, payload, attempt);
    if (result.success) return { delivered: true, attempts: attempt };

    if (attempt < maxAttempts) {
      // Exponential backoff: 1s, 2s, 4s, 8s — all durable
      yield ctx.sleep(baseDelay * Math.pow(2, attempt - 1));
    }
  }
  return { delivered: false, attempts: maxAttempts };
});
```

**How it works:** `ctx.sleep(ms)` creates a child promise with a `resonate:timer` tag. The server auto-resolves it when the timer fires. The task suspends until then. If the worker dies, another worker picks up the task after the timer resolves.

**When to use:** Drip campaigns, SLA reminders, retry delays, polling loops, scheduled follow-ups.

---

## 5. Entity Lifecycle — Long-Lived Mutable State

Model a domain entity's lifecycle as a durable workflow. Each state transition is a checkpoint.

### Order Lifecycle

```typescript
resonate.register("orderLifecycle", function* (ctx: Context, orderId: string, items: string[]) {
  // Created
  const order = yield ctx.run(createOrder, orderId, items);

  // Validated
  yield ctx.run(validateOrder, order);

  // Payment processed
  const payment = yield ctx.run(processPayment, order);

  // Fulfilled
  const shipment = yield ctx.run(fulfillOrder, order, payment);

  // Notified
  yield ctx.run(notifyCustomer, order, shipment);

  return { orderId, status: "fulfilled", tracking: shipment.tracking };
});
```

**When to use:** Order processing, user onboarding, subscription management, document approval chains, any state machine. Combine with Pattern 4 (durable sleep) for trial periods and grace windows.

---

## 6. Cross-Service RPC — Distributed Coordination

Dispatch work to workers in different groups. Each RPC is a durable checkpoint — if the calling worker dies, it resumes where it left off.

### Orchestrator + Workers

```typescript
// --- Orchestrator (group: "api") ---
const api = new Resonate({ url: "http://localhost:8001", group: "api" });

api.register("handleOrder", function* (ctx: Context, orderId: string) {
  // Dispatch to payment workers — suspends until complete
  const charge = yield ctx.rpc("chargeCard", [orderId, 99.99], {
    target: "poll://any@payment-workers",
  });

  // Dispatch to shipping workers
  const shipment = yield ctx.rpc("createShipment", [orderId, charge.txnId], {
    target: "poll://any@shipping-workers",
  });

  return { orderId, charge, shipment };
});

await api.start();

// --- Payment workers (separate process) ---
const payments = new Resonate({ url: "http://localhost:8001", group: "payment-workers" });
payments.register("chargeCard", function* (_ctx: Context, orderId: string, amount: number) {
  console.log(`Charging $${amount} for ${orderId}`);
  return { txnId: `txn_${orderId}`, amount };
});
await payments.start();

// --- Shipping workers (separate process) ---
const shipping = new Resonate({ url: "http://localhost:8001", group: "shipping-workers" });
shipping.register("createShipment", function* (_ctx: Context, orderId: string, txnId: string) {
  console.log(`Shipping ${orderId} (txn: ${txnId})`);
  return { tracking: `TRACK_${orderId}` };
});
await shipping.start();
```

### Gateway Dispatch (Non-Workflow Context)

From an HTTP handler, dispatch to workers via the wire protocol using `promise.create` with a `resonate:target` tag. See the `resonate-gateway.ts` asset template for a complete working gateway.

**RPC semantics:** Creates a child promise with `resonate:target` tag → suspends current task → server dispatches to target group → on completion, calling task replays with preload cache.

**When to use:** Microservice coordination, worker pools, cross-team step dispatch.

---

## Pattern Decision Tree

```
What does your workflow need?

├─ Multiple steps that must all succeed or all roll back?
│  └─ Pattern 1: Saga
│
├─ Dispatch work to parallel workers and collect results?
│  └─ Pattern 2: Fan-Out/Fan-In
│
├─ Pause for human decision, webhook, or external event?
│  └─ Pattern 3: Human-in-the-Loop
│
├─ Wait for hours/days without a process staying alive?
│  └─ Pattern 4: Durable Sleep/Cron
│
├─ Model a domain entity's state machine?
│  └─ Pattern 5: Entity Lifecycle
│
└─ Coordinate across multiple services/teams?
   └─ Pattern 6: Cross-Service RPC
```

Most real workflows combine patterns. An order saga (Pattern 1) might fan out notifications (Pattern 2), wait for a fraud review (Pattern 3), and sleep before a follow-up email (Pattern 4).

---

## Common Mistakes

See `RESONATE-SDK.md` § Anti-Patterns for the full list. The critical ones for patterns:

- **Missing `yield`** — `ctx.run(fn)` without `yield` creates a descriptor but never executes the step.
- **`async function` instead of `function*`** — The engine drives execution via generators, not async/await.
- **Non-deterministic logic between yields** — `Math.random()`, `Date.now()` produce different values on replay.
- **Wrong compensation order** — Always reverse. Step 1, 2, 3 → compensate 3, 2, 1.
