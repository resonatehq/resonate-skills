---
name: resonate-saga-pattern-typescript
description: Implement saga patterns for distributed transactions with compensation logic. Use when coordinating multi-step processes that need to maintain consistency across failures by unwinding completed steps.
---

# Resonate Saga Pattern (TypeScript)

## Overview

The Saga pattern coordinates long-running distributed transactions by breaking them into smaller, isolated steps with compensating actions. If any step fails, the saga executes compensation logic to undo completed steps, maintaining eventual consistency without distributed locks or two-phase commits.

**Core principle:** Each step has a forward action (transaction) and a compensating action (rollback). On failure, compensations execute in reverse order to restore consistency.

## Mental Model

```
Forward Execution (Happy Path):
  Step 1 → Step 2 → Step 3 → Success

Failure with Compensation:
  Step 1 ✓ → Step 2 ✓ → Step 3 ✗
         ↓
  Compensate 2 → Compensate 1 → Saga Failed (Consistent)
```

## Basic Saga Structure

```typescript
function* sagaWorkflow(ctx: Context, orderId: string) {
  const completedSteps: string[] = [];

  try {
    // Step 1: Reserve inventory
    yield* ctx.run(reserveInventory, orderId);
    completedSteps.push("inventory");

    // Step 2: Charge payment
    yield* ctx.run(chargePayment, orderId);
    completedSteps.push("payment");

    // Step 3: Create shipment
    yield* ctx.run(createShipment, orderId);
    completedSteps.push("shipment");

    return { status: "success", orderId };

  } catch (error) {
    // Compensate in reverse order
    for (const step of completedSteps.reverse()) {
      yield* ctx.run(compensate, step, orderId);
    }

    return { status: "failed", orderId, compensated: completedSteps };
  }
}

async function compensate(_ctx: Context, step: string, orderId: string) {
  switch (step) {
    case "shipment":
      await cancelShipment(orderId);
      break;
    case "payment":
      await refundPayment(orderId);
      break;
    case "inventory":
      await releaseInventory(orderId);
      break;
  }
}
```

## Pattern Variants

### 1. Explicit Compensation Functions

```typescript
interface SagaStep<T> {
  name: string;
  action: (ctx: Context, data: T) => Generator;
  compensate: (ctx: Context, data: T) => Generator;
}

function* genericSaga<T>(ctx: Context, steps: SagaStep<T>[], data: T) {
  const completed: SagaStep<T>[] = [];

  try {
    for (const step of steps) {
      yield* step.action(ctx, data);
      completed.push(step);
    }

    return { status: "success" };
  } catch (error) {
    console.error(`Saga failed at step ${completed.length + 1}:`, error);

    // Compensate in reverse
    for (const step of completed.reverse()) {
      try {
        yield* step.compensate(ctx, data);
      } catch (compensateError) {
        console.error(`Compensation failed for ${step.name}:`, compensateError);
        // Log but continue compensating other steps
      }
    }

    return { status: "failed", error, compensated: completed.map(s => s.name) };
  }
}

// Usage
function* orderSaga(ctx: Context, order: Order) {
  const steps: SagaStep<Order>[] = [
    {
      name: "inventory",
      action: function* (ctx, order) {
        yield* ctx.run(reserveInventory, order.id);
      },
      compensate: function* (ctx, order) {
        yield* ctx.run(releaseInventory, order.id);
      }
    },
    {
      name: "payment",
      action: function* (ctx, order) {
        yield* ctx.run(chargePayment, order.id, order.total);
      },
      compensate: function* (ctx, order) {
        yield* ctx.run(refundPayment, order.id);
      }
    },
    {
      name: "shipment",
      action: function* (ctx, order) {
        yield* ctx.run(createShipment, order.id);
      },
      compensate: function* (ctx, order) {
        yield* ctx.run(cancelShipment, order.id);
      }
    }
  ];

  return yield* genericSaga(ctx, steps, order);
}
```

### 2. Saga with Partial Compensation

```typescript
function* advancedSaga(ctx: Context, orderId: string) {
  const log: Array<{ step: string; canCompensate: boolean }> = [];

  try {
    // Step 1: Reserve inventory (compensable)
    yield* ctx.run(reserveInventory, orderId);
    log.push({ step: "inventory", canCompensate: true });

    // Step 2: Charge payment (compensable)
    yield* ctx.run(chargePayment, orderId);
    log.push({ step: "payment", canCompensate: true });

    // Step 3: Send to fulfillment service (NOT compensable - external system)
    yield* ctx.run(notifyFulfillmentService, orderId);
    log.push({ step: "fulfillment", canCompensate: false });

    // Step 4: Create shipment label (compensable)
    yield* ctx.run(createShipmentLabel, orderId);
    log.push({ step: "label", canCompensate: true });

    return { status: "success" };

  } catch (error) {
    // Compensate only compensable steps
    for (const entry of log.reverse()) {
      if (entry.canCompensate) {
        yield* ctx.run(compensateStep, entry.step, orderId);
      } else {
        // Log for manual intervention
        yield* ctx.run(logManualCompensationNeeded, entry.step, orderId);
      }
    }

    return { status: "failed", requiresManualReview: log.some(e => !e.canCompensate) };
  }
}
```

### 3. Distributed Saga (Cross-Service)

```typescript
function* distributedSaga(ctx: Context, orderId: string) {
  const compensations: Array<() => Generator> = [];

  try {
    // Step 1: Reserve inventory (inventory-service)
    yield* ctx.rpc(
      "reserveInventory",
      orderId,
      ctx.options({ target: "poll://any@inventory-service" })
    );
    compensations.push(function* () {
      yield* ctx.rpc(
        "releaseInventory",
        orderId,
        ctx.options({ target: "poll://any@inventory-service" })
      );
    });

    // Step 2: Charge payment (payment-service)
    yield* ctx.rpc(
      "chargePayment",
      orderId,
      ctx.options({ target: "poll://any@payment-service" })
    );
    compensations.push(function* () {
      yield* ctx.rpc(
        "refundPayment",
        orderId,
        ctx.options({ target: "poll://any@payment-service" })
      );
    });

    // Step 3: Create shipment (shipping-service)
    yield* ctx.rpc(
      "createShipment",
      orderId,
      ctx.options({ target: "poll://any@shipping-service" })
    );
    compensations.push(function* () {
      yield* ctx.rpc(
        "cancelShipment",
        orderId,
        ctx.options({ target: "poll://any@shipping-service" })
      );
    });

    return { status: "success" };

  } catch (error) {
    // Execute compensations in reverse
    for (const compensate of compensations.reverse()) {
      yield* compensate();
    }

    return { status: "failed", orderId };
  }
}
```

## Real-World Example: E-Commerce Order

```typescript
interface Order {
  id: string;
  items: Array<{ sku: string; quantity: number; price: number }>;
  customerId: string;
  total: number;
  paymentMethod: string;
}

function* processOrderSaga(ctx: Context, order: Order) {
  const state: {
    inventoryReserved?: boolean;
    paymentCharged?: boolean;
    shipmentCreated?: boolean;
    emailSent?: boolean;
  } = {};

  try {
    // 1. Validate order
    yield* ctx.run(validateOrder, order);

    // 2. Reserve inventory
    const reservationId = yield* ctx.run(reserveInventory, order);
    state.inventoryReserved = true;

    // 3. Charge payment
    const transactionId = yield* ctx.run(chargePayment, order);
    state.paymentCharged = true;

    // 4. Create shipment
    const trackingNumber = yield* ctx.run(createShipment, order, transactionId);
    state.shipmentCreated = true;

    // 5. Send confirmation email
    yield* ctx.run(sendConfirmationEmail, order, trackingNumber);
    state.emailSent = true;

    // 6. Update order status
    yield* ctx.run(updateOrderStatus, order.id, "completed");

    return {
      status: "success",
      orderId: order.id,
      transactionId,
      trackingNumber
    };

  } catch (error) {
    console.error("Order saga failed:", error);

    // Compensate in reverse order
    if (state.shipmentCreated) {
      yield* ctx.run(cancelShipment, order.id);
    }

    if (state.paymentCharged) {
      yield* ctx.run(refundPayment, order.id);
    }

    if (state.inventoryReserved) {
      yield* ctx.run(releaseInventory, order.id);
    }

    // Update order status
    yield* ctx.run(updateOrderStatus, order.id, "failed");

    // Notify customer
    yield* ctx.run(sendFailureEmail, order);

    return {
      status: "failed",
      orderId: order.id,
      error: error.message,
      compensated: Object.keys(state).filter(k => state[k])
    };
  }
}

// Helper functions
async function validateOrder(_ctx: Context, order: Order) {
  if (order.total <= 0) {
    throw new Error("Invalid order total");
  }
  if (!order.customerId) {
    throw new Error("Customer ID required");
  }
  return true;
}

async function reserveInventory(_ctx: Context, order: Order) {
  // Call inventory service
  const response = await inventoryClient.reserve({
    items: order.items,
    orderId: order.id
  });

  if (!response.success) {
    throw new Error(`Inventory reservation failed: ${response.error}`);
  }

  return response.reservationId;
}

async function chargePayment(_ctx: Context, order: Order) {
  // Call payment processor
  const response = await paymentClient.charge({
    amount: order.total,
    customerId: order.customerId,
    method: order.paymentMethod,
    orderId: order.id
  });

  if (!response.success) {
    throw new Error(`Payment failed: ${response.error}`);
  }

  return response.transactionId;
}

async function createShipment(_ctx: Context, order: Order, transactionId: string) {
  // Call shipping service
  const response = await shippingClient.createShipment({
    orderId: order.id,
    items: order.items,
    customerId: order.customerId,
    transactionId
  });

  if (!response.success) {
    throw new Error(`Shipment creation failed: ${response.error}`);
  }

  return response.trackingNumber;
}

// Compensation functions
async function releaseInventory(_ctx: Context, orderId: string) {
  await inventoryClient.release({ orderId });
}

async function refundPayment(_ctx: Context, orderId: string) {
  await paymentClient.refund({ orderId });
}

async function cancelShipment(_ctx: Context, orderId: string) {
  await shippingClient.cancel({ orderId });
}
```

## Saga Choreography vs Orchestration

### Orchestration (Recommended with Resonate)

**One workflow coordinates all steps:**

```typescript
function* orderOrchestrator(ctx: Context, order: Order) {
  // Central coordinator decides what happens
  yield* ctx.rpc("inventory.reserve", order);
  yield* ctx.rpc("payment.charge", order);
  yield* ctx.rpc("shipping.create", order);

  return { status: "success" };
}
```

**Pros:**
- Clear control flow
- Easy to understand
- Centralized compensation logic
- Resonate's natural model

### Choreography (Event-Driven)

**Services react to events:**

```typescript
// Each service listens for events and reacts independently
// More complex with Resonate, better suited for message queues
```

**Resonate is optimized for orchestration**, not choreography.

## Idempotency in Sagas

Each saga step should be idempotent to handle retries:

```typescript
async function reserveInventory(_ctx: Context, order: Order) {
  // Check if already reserved
  const existing = await db.getReservation(order.id);
  if (existing) {
    return existing.reservationId;  // Idempotent
  }

  // Create new reservation
  const reservation = await inventoryClient.reserve(order);
  await db.saveReservation(order.id, reservation.id);

  return reservation.id;
}
```

## Timeout Handling

Add timeouts to prevent indefinite waits:

```typescript
function* sagaWithTimeouts(ctx: Context, orderId: string) {
  try {
    // Step with timeout
    yield* ctx.run(
      reserveInventory,
      orderId,
      ctx.options({ timeout: 30_000 })  // 30 seconds
    );

    yield* ctx.run(
      chargePayment,
      orderId,
      ctx.options({ timeout: 60_000 })  // 60 seconds
    );

    return { status: "success" };
  } catch (error) {
    // Timeout or failure - compensate
    yield* ctx.run(compensateAll, orderId);
    return { status: "timeout" };
  }
}
```

## Saga State Persistence

Track saga progress in database for observability:

```typescript
function* observableSaga(ctx: Context, orderId: string) {
  // Initialize saga state
  yield* ctx.run(async () => {
    await db.from("sagas").insert({
      order_id: orderId,
      status: "started",
      steps: []
    });
  });

  try {
    // Step 1
    yield* ctx.run(reserveInventory, orderId);
    yield* ctx.run(async () => {
      await db.from("sagas").update({
        steps: db.raw("array_append(steps, 'inventory')")
      }).where({ order_id: orderId });
    });

    // Step 2
    yield* ctx.run(chargePayment, orderId);
    yield* ctx.run(async () => {
      await db.from("sagas").update({
        steps: db.raw("array_append(steps, 'payment')")
      }).where({ order_id: orderId });
    });

    // Mark complete
    yield* ctx.run(async () => {
      await db.from("sagas").update({
        status: "completed"
      }).where({ order_id: orderId });
    });

    return { status: "success" };
  } catch (error) {
    // Mark failed and compensate
    yield* ctx.run(async () => {
      await db.from("sagas").update({
        status: "compensating"
      }).where({ order_id: orderId });
    });

    // Compensate...

    yield* ctx.run(async () => {
      await db.from("sagas").update({
        status: "failed"
      }).where({ order_id: orderId });
    });

    return { status: "failed" };
  }
}
```

## Common Pitfalls

### 1. Non-Idempotent Steps

```typescript
// ❌ WRONG - Not idempotent
async function chargePayment(_ctx: Context, orderId: string) {
  return await paymentClient.charge({ orderId });  // Charges again on retry!
}

// ✅ CORRECT - Idempotent
async function chargePayment(_ctx: Context, orderId: string) {
  const existing = await db.getTransaction(orderId);
  if (existing) return existing.transactionId;

  const transaction = await paymentClient.charge({ orderId });
  await db.saveTransaction(orderId, transaction.id);
  return transaction.id;
}
```

### 2. Missing Compensation Logic

```typescript
// ❌ WRONG - No way to undo
function* badSaga(ctx: Context, orderId: string) {
  yield* ctx.run(doSomething, orderId);
  yield* ctx.run(doSomethingElse, orderId);
  // If this fails, no compensation!
}

// ✅ CORRECT - Compensation defined
function* goodSaga(ctx: Context, orderId: string) {
  try {
    yield* ctx.run(doSomething, orderId);
    yield* ctx.run(doSomethingElse, orderId);
  } catch (error) {
    yield* ctx.run(undoSomethingElse, orderId);
    yield* ctx.run(undoSomething, orderId);
    throw error;
  }
}
```

### 3. Incorrect Compensation Order

```typescript
// ❌ WRONG - Wrong order
try {
  yield* ctx.run(step1);
  yield* ctx.run(step2);
  yield* ctx.run(step3);
} catch (error) {
  yield* ctx.run(compensate1);  // Should be 3, 2, 1!
  yield* ctx.run(compensate2);
  yield* ctx.run(compensate3);
}

// ✅ CORRECT - Reverse order
try {
  yield* ctx.run(step1);
  yield* ctx.run(step2);
  yield* ctx.run(step3);
} catch (error) {
  yield* ctx.run(compensate3);
  yield* ctx.run(compensate2);
  yield* ctx.run(compensate1);
}
```

## When to Use Sagas

**Use sagas when:**
- Coordinating multi-step distributed transactions
- Each step can be individually compensated
- You need eventual consistency (not strict ACID)
- Steps span multiple services/systems
- Long-running processes (minutes to hours)

**Don't use sagas when:**
- Single database transaction is sufficient (use DB transactions)
- Steps cannot be meaningfully compensated
- You need strict ACID guarantees
- Real-time consistency is required

## Summary

Sagas with Resonate provide:
- Durable multi-step transactions
- Automatic compensation on failure
- Clear orchestration model
- Crash-resistant execution
- Observable progress tracking

**Core pattern:** Forward steps + compensation logic + try/catch + reverse compensation order
