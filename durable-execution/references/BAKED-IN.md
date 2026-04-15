# Baked-In Durability — Framework-Free

Add crash recovery and exactly-once execution to any application using only your existing database. No framework, no infrastructure, no dependencies.

---

## The Schema

Two tables. Works with SQLite, Postgres, MySQL — anything with transactions.

```sql
-- Track completed steps so replay skips them
CREATE TABLE IF NOT EXISTS workflow_steps (
  workflow_id TEXT NOT NULL,
  step_key    TEXT NOT NULL,
  result      TEXT,          -- JSON-serialized return value
  created_at  INTEGER NOT NULL DEFAULT (unixepoch()),
  PRIMARY KEY (workflow_id, step_key)
);

-- Exactly-once delivery of side effects (emails, webhooks, API calls)
CREATE TABLE IF NOT EXISTS outbox (
  id           TEXT PRIMARY KEY,
  payload      TEXT NOT NULL,    -- JSON: { type, data }
  created_at   INTEGER NOT NULL DEFAULT (unixepoch()),
  delivered_at INTEGER           -- NULL until delivered
);
```

---

## Building Block 1: Step Checkpointing

The core primitive. Wrap each step — on replay, completed steps return their cached result.

### SQLite (Bun)

```typescript
import { Database } from "bun:sqlite";

const db = new Database("workflow.db");
db.run(`CREATE TABLE IF NOT EXISTS workflow_steps (
  workflow_id TEXT NOT NULL,
  step_key TEXT NOT NULL,
  result TEXT,
  created_at INTEGER NOT NULL DEFAULT (unixepoch()),
  PRIMARY KEY (workflow_id, step_key)
)`);

async function runStep<T>(
  workflowId: string,
  stepKey: string,
  fn: () => T | Promise<T>
): Promise<T> {
  // Check if step already completed (replay path)
  const existing = db.query(
    "SELECT result FROM workflow_steps WHERE workflow_id = ? AND step_key = ?"
  ).get(workflowId, stepKey) as { result: string } | null;

  if (existing) {
    return JSON.parse(existing.result) as T;
  }

  // First execution — run the function and checkpoint the result
  const result = await fn();
  db.run(
    "INSERT INTO workflow_steps (workflow_id, step_key, result) VALUES (?, ?, ?)",
    [workflowId, stepKey, JSON.stringify(result)]
  );
  return result;
}
```

### Postgres

```typescript
import postgres from "postgres";

const sql = postgres(process.env.DATABASE_URL!);

async function runStep<T>(
  workflowId: string,
  stepKey: string,
  fn: () => T | Promise<T>
): Promise<T> {
  const [existing] = await sql`
    SELECT result FROM workflow_steps
    WHERE workflow_id = ${workflowId} AND step_key = ${stepKey}
  `;

  if (existing) {
    return JSON.parse(existing.result) as T;
  }

  const result = await fn();
  await sql`
    INSERT INTO workflow_steps (workflow_id, step_key, result)
    VALUES (${workflowId}, ${stepKey}, ${JSON.stringify(result)})
    ON CONFLICT DO NOTHING
  `;
  return result;
}
```

### Usage

```typescript
async function processOrder(orderId: string) {
  const wf = orderId; // workflow ID = order ID

  const inventory = await runStep(wf, "reserve",  () => reserveInventory(orderId));
  const payment   = await runStep(wf, "charge",   () => chargeCard(orderId, inventory.total));
  const shipment  = await runStep(wf, "ship",     () => createShipment(orderId, payment.txnId));
  const _email    = await runStep(wf, "confirm",  () => sendConfirmation(orderId, shipment.tracking));

  return { orderId, status: "completed", tracking: shipment.tracking };
}
```

If the process crashes after `charge` but before `ship`, restart calls `processOrder("order-42")` again. The `reserve` and `charge` steps return cached results instantly. Execution resumes at `ship`.

---

## Building Block 2: Idempotency Keys for External Calls

External APIs (Stripe, SendGrid, etc.) support idempotency keys. Use deterministic keys derived from workflow + step identity.

```typescript
async function chargeCard(orderId: string, amount: number): Promise<{ txnId: string }> {
  const idempotencyKey = `charge-${orderId}`;

  const response = await fetch("https://api.stripe.com/v1/payment_intents", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.STRIPE_KEY}`,
      "Idempotency-Key": idempotencyKey,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: `amount=${amount}&currency=usd`,
  });

  const intent = await response.json();
  return { txnId: intent.id };
}
```

Even if `chargeCard` runs twice (process crashed after payment but before checkpoint), Stripe deduplicates using the idempotency key. No double charges.

**Key rules:**
- Derive the key from stable, deterministic inputs (workflow ID, step name, order ID)
- Never use `Date.now()`, `Math.random()`, or `crypto.randomUUID()` in keys
- Most payment APIs (Stripe, Square, Adyen) and many SaaS APIs support idempotency keys natively

---

## Building Block 3: Outbox Pattern

Never send side effects (emails, webhooks) directly from a workflow step. Instead, write to an outbox table. A separate process delivers them.

### Writer (inside your workflow)

```typescript
function enqueueSideEffect(db: Database, id: string, type: string, data: unknown) {
  db.run(
    "INSERT OR IGNORE INTO outbox (id, payload) VALUES (?, ?)",
    [id, JSON.stringify({ type, data })]
  );
}

// In your workflow — write to outbox instead of sending directly
async function processOrder(orderId: string) {
  const wf = orderId;
  const order   = await runStep(wf, "reserve", () => reserveInventory(orderId));
  const payment = await runStep(wf, "charge",  () => chargeCard(orderId, order.total));

  // Don't send email directly — write to outbox
  await runStep(wf, "queue-email", () => {
    enqueueSideEffect(db, `${orderId}:confirmation`, "email", {
      to: order.email,
      subject: "Order confirmed",
      body: `Your order ${orderId} is confirmed. Tracking: ${payment.txnId}`,
    });
    return { queued: true };
  });
}
```

### Delivery Loop (separate process or interval)

```typescript
async function processOutbox(db: Database) {
  const pending = db.query(
    "SELECT id, payload FROM outbox WHERE delivered_at IS NULL ORDER BY created_at LIMIT 100"
  ).all() as Array<{ id: string; payload: string }>;

  for (const msg of pending) {
    const { type, data } = JSON.parse(msg.payload);

    try {
      switch (type) {
        case "email":
          await sendEmail(data.to, data.subject, data.body);
          break;
        case "webhook":
          await fetch(data.url, { method: "POST", body: JSON.stringify(data.body) });
          break;
      }

      db.run("UPDATE outbox SET delivered_at = ? WHERE id = ?", [Date.now(), msg.id]);
    } catch (error) {
      console.error(`Outbox delivery failed for ${msg.id}:`, error);
      // Will retry on next loop iteration
    }
  }
}

// Run every 5 seconds
setInterval(() => processOutbox(db), 5000);
```

**Why outbox, not direct send:**
- Workflow step might replay → email sends again without outbox dedup
- `INSERT OR IGNORE` makes the write idempotent
- Delivery is retried independently of the workflow
- Failed deliveries don't block workflow completion

---

## Complete Example: Durable Checkout (SQLite)

```typescript
import { Database } from "bun:sqlite";

const db = new Database("checkout.db");

// Schema
db.run(`CREATE TABLE IF NOT EXISTS workflow_steps (
  workflow_id TEXT NOT NULL, step_key TEXT NOT NULL,
  result TEXT, created_at INTEGER DEFAULT (unixepoch()),
  PRIMARY KEY (workflow_id, step_key)
)`);
db.run(`CREATE TABLE IF NOT EXISTS outbox (
  id TEXT PRIMARY KEY, payload TEXT NOT NULL,
  created_at INTEGER DEFAULT (unixepoch()), delivered_at INTEGER
)`);

// Step runner
async function runStep<T>(wfId: string, key: string, fn: () => T | Promise<T>): Promise<T> {
  const row = db.query("SELECT result FROM workflow_steps WHERE workflow_id = ? AND step_key = ?")
    .get(wfId, key) as { result: string } | null;
  if (row) return JSON.parse(row.result);
  const result = await fn();
  db.run("INSERT INTO workflow_steps (workflow_id, step_key, result) VALUES (?, ?, ?)",
    [wfId, key, JSON.stringify(result)]);
  return result;
}

// The workflow
async function checkout(orderId: string) {
  const inventory = await runStep(orderId, "reserve", () =>
    reserveInventory(orderId));

  const payment = await runStep(orderId, "charge", () =>
    chargeCard(orderId, inventory.total, `charge-${orderId}`));

  const shipment = await runStep(orderId, "ship", () =>
    createShipment(orderId, payment.txnId));

  await runStep(orderId, "notify", () => {
    db.run("INSERT OR IGNORE INTO outbox (id, payload) VALUES (?, ?)",
      [`${orderId}:email`, JSON.stringify({
        type: "email",
        data: { to: inventory.email, subject: "Shipped!", body: shipment.tracking }
      })]);
    return { queued: true };
  });

  return { orderId, status: "completed", tracking: shipment.tracking };
}

// Run it — idempotent, crash-safe
await checkout("order-42");
```

---

## Saga Pattern (Baked In)

Compensation on failure, without a framework.

```typescript
async function sagaCheckout(orderId: string) {
  const completed: Array<{ step: string; undo: () => Promise<void> }> = [];

  try {
    const inv = await runStep(orderId, "reserve", () => reserveInventory(orderId));
    completed.push({ step: "reserve", undo: () => releaseInventory(orderId) });

    const pay = await runStep(orderId, "charge", () => chargeCard(orderId, inv.total));
    completed.push({ step: "charge", undo: () => refundPayment(pay.txnId) });

    const ship = await runStep(orderId, "ship", () => createShipment(orderId));
    completed.push({ step: "ship", undo: () => cancelShipment(ship.trackingId) });

    return { status: "success" };
  } catch (error) {
    // Compensate in reverse order — each compensation is also checkpointed
    for (const { step, undo } of completed.reverse()) {
      await runStep(orderId, `undo-${step}`, undo);
    }
    return { status: "rolled-back" };
  }
}
```

---

## When to Graduate to Resonate

Baked-in works great for sequential, single-service workflows. You've outgrown it when:

| You need... | Baked-in can't | Resonate can |
|-------------|----------------|--------------|
| Sleep for hours/days without a process alive | No — needs a running process or cron | Durable timers — server resolves them automatically |
| Pause for human approval | Awkward — requires polling | `ctx.promise()` suspends, resumes on external signal |
| Fan out to parallel workers | Manual — spawn threads, track completion | `ctx.rpc()` dispatches to worker groups |
| Cross-service coordination | Complex — custom protocol needed | Built-in task dispatch + target groups |
| Automatic retries with backoff | DIY retry loops | Built into the platform |
| Observability into workflow state | Query your `workflow_steps` table | Promise state graph, execution timeline |

The migration is straightforward: each `runStep()` call becomes a `yield ctx.run()`. The mental model is the same — checkpoint each step, skip on replay. Resonate just handles the hard parts (timers, suspension, distribution, retries) for you.
