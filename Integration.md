# Writing Resonate Functions

Resonate enables stateful, durable executions on stateless, ephemeral infrastructure. You write the functions, Resonate handles coordination and supervision.

## 1 Setup

### 1.1 Required Secrets

- `RESONATE_URL`

Always ask the user for their Resonate Server URL before developing, debugging, and deploying resonate functions. You MUST use the `add_secret` tool to add the `RESONATE_URL` secret.

### 1.2 File Structure

```
supabase/
├── config.toml
└── functions/
    ├── flows/
    │   ├── deno.json
    │   └── index.ts     # Your resonate functions go here
    ├── probe/
    │   ├── deno.json
    │   └── index.ts     # Do not modify
    └── start/
    │   ├── deno.json
        └── index.ts     # Do not modify
```

All resonate functions are written in `flows/index.ts` and registered with `resonate.register()`.

## 2 Syntax

### 2.1 Function Signature

Resonate functions are generator functions. The first parameter is always `context`.

```typescript
function* foo(context: Context, arg1: string, arg2: number) {
  // Your logic here
  return { success: true };
}
```

### 2.2 Invoking Functions

Always use `yield*` before context methods.

```typescript
// Invoke and await immediately
function* foo(context: Context, a: string, b: string) {
  const x = yield* context.run(bar, a);
  const y = yield* context.run(baz, b);
  return { x, y };
}

// Invoke now, await later (for concurrency)
function* foo(context: Context, a: string, b: string) {
  const futureX = yield* context.beginRun(bar, a);
  const futureY = yield* context.beginRun(baz, b);

  const x = yield* futureX;
  const y = yield* futureY;

  return { x, y };
}
```

**Structured Concurrency:** Children started with `beginRun` are implicitly awaited before the parent returns. No orphaned work — the parent cannot complete until all children complete.

```typescript
function* foo(context: Context, a: string) {
  yield* context.beginRun(bar, a);  // Started, not explicitly awaited
  yield* context.beginRun(baz, a);  // Started, not explicitly awaited
  return "done";  // foo does NOT return until bar and baz complete
}
```

### 2.3 Sleeping

```typescript
function* foo(context: Context, arg: string) {
  yield* context.run(bar, arg);

  // Sleep for 24 hours
  yield* context.sleep(24 * 60 * 60 * 1000);

  yield* context.run(baz, arg);
}
```

### 2.4 Registration

```typescript
import { Resonate, type Context } from "@resonatehq/supabase";

const resonate = new Resonate();

function* foo(context: Context, arg: string) {
  // ...
}

function* bar(context: Context, arg: string) {
  // ...
}

resonate.register(foo);
resonate.register(bar);

resonate.httpHandler();
```

## 3 Rules

### 3.1 Serialization

Return values must be serializable. Arguments to `context.rpc()` must be serializable.

| | Arguments | Return values |
|---|---|---|
| `context.run()` | No restriction | Must serialize |
| `context.rpc()` | Must serialize | Must serialize |

**Serializable:** plain objects, arrays, strings, numbers, booleans, null

**Not serializable:** functions, class instances, database connections, circular references

### 3.2 Determinism

Functions may replay. Non-deterministic operations must use context helpers.

```typescript
// WRONG — different value on replay
function* foo(context: Context, arg: string) {
  const now = Date.now();           // Different on replay!
  const id = Math.random();         // Different on replay!
  return { id, now, arg };
}

// CORRECT — consistent across replays
function* foo(context: Context, arg: string) {
  const now = yield* context.date.now();
  const id = yield* context.math.random();
  return { id, now, arg };
}
```

### 3.3 Side Effects

Side effects must be wrapped in `context.run()` to be durable. Unwrapped side effects will re-execute on replay.

```typescript
// WRONG — side effect runs again on every replay
function* foo(context: Context, arg: string) {
  await bar(arg);  // Not durable! Runs multiple times on replay.
  return { done: true };
}

// CORRECT — side effect runs exactly once
function* foo(context: Context, arg: string) {
  yield* context.run(bar, arg);
  return { done: true };
}
```

## 4 Patterns

### 4.1 Sequential Steps

```typescript
function* processOrder(context: Context, orderId: string) {
  const order = yield* context.run(getOrder, orderId);
  const payment = yield* context.run(chargeCard, order);
  const shipment = yield* context.run(createShipment, order);
  yield* context.run(sendConfirmation, order, shipment);
  return { orderId, status: "complete" };
}
```

### 4.2 Concurrent Steps (Fork-Join)

```typescript
function* validateOrder(context: Context, orderId: string) {
  // Fork: start all checks
  const inventoryFuture = yield* context.beginRun(checkInventory, orderId);
  const fraudFuture = yield* context.beginRun(checkFraud, orderId);
  const creditFuture = yield* context.beginRun(checkCredit, orderId);

  // Join: await all results
  const inventory = yield* inventoryFuture;
  const fraud = yield* fraudFuture;
  const credit = yield* creditFuture;

  return { inventory, fraud, credit };
}
```

### 4.3 Delayed Execution

```typescript
function* reminder(context: Context, userId: string) {
  yield* context.run(sendInitialEmail, userId);

  // Wait 24 hours
  yield* context.sleep(24 * 60 * 60 * 1000);

  yield* context.run(sendFollowUpEmail, userId);
}
```

## 5 Anti-Patterns

### 5.1 Forgetting yield*

```typescript
// WRONG — context.run() returns a generator, not a result
function* foo(context: Context, arg: string) {
  const x = context.run(bar, arg);  // Missing yield*!
  return { x };  // x is a generator, not data
}

// CORRECT
function* foo(context: Context, arg: string) {
  const x = yield* context.run(bar, arg);
  return { x };
}
```

### 5.2 Using async/await Instead of Generators

```typescript
// WRONG — not a generator function
async function foo(context: Context, arg: string) {
  const x = await context.run(bar, arg);  // Won't work!
  return { x };
}

// CORRECT — generator function with yield*
function* foo(context: Context, arg: string) {
  const x = yield* context.run(bar, arg);
  return { x };
}
```

### 5.3 Returning Non-Serializable Values

```typescript
// WRONG — functions are not serializable
function* foo(context: Context) {
  return () => console.log("hello");  // Cannot serialize a function!
}

// CORRECT — return data, not functions
function* foo(context: Context) {
  return { message: "hello" };
}
```

### 5.4 Interleaving Begin and Await

```typescript
// WRONG — hard to follow, no actual concurrency
function* foo(context: Context, arg: string) {
  const futureX = yield* context.beginRun(bar, arg);
  const x = yield* futureX;  // Awaiting immediately defeats the purpose
  const futureY = yield* context.beginRun(baz, arg);
  const y = yield* futureY;
  return { x, y };
}

// CORRECT — fork all, then join all
function* foo(context: Context, arg: string) {
  const futureX = yield* context.beginRun(bar, arg);
  const futureY = yield* context.beginRun(baz, arg);

  const x = yield* futureX;
  const y = yield* futureY;
  return { x, y };
}
```

## 6 Application Integration

### 6.1 Starting a Resonate Function

From your app, call the `start/` endpoint:

```typescript
const response = await fetch(`${SUPABASE_URL}/functions/v1/start`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${SUPABASE_ANON_KEY}`
  },
  body: JSON.stringify({
    uuid: "unique-execution-id",  // You control this ID
    func: "processOrder",
    args: [orderId, userId]
  })
});

const { uuid } = await response.json();
// Store this uuid to check status later
```

**Important:** The `uuid` you provide becomes `context.id` inside the Resonate function. Use a meaningful ID (e.g., `order-${orderId}`) so you can correlate the execution with your domain objects.

### 6.2 Checking Execution Status

Poll the `probe/` endpoint to check if an execution is still running:

```typescript
const response = await fetch(`${SUPABASE_URL}/functions/v1/probe`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${SUPABASE_ANON_KEY}`
  },
  body: JSON.stringify({
    uuid: "unique-execution-id"
  })
});

const { status, value } = await response.json();
// status: "pending" | "resolved" | "rejected"
// value: the return value (if resolved)
```

### 6.3 Getting Results

When `status` is `"resolved"`, the `value` field contains the function's return value. When `status` is `"rejected"`, the `value` field contains error information.

## 7 Application Patterns

### 7.1 Use context.id for Database Records

The execution ID (`context.id`) should be used as the primary key or foreign key when writing to the database. This allows the UI to query progress by execution ID.

```typescript
function* processOrder(context: Context, orderId: string) {
  const supabase = context.getDependency("supabase");

  // Use context.id as the record ID
  yield* context.run(async () => {
    await supabase.from("order_progress").upsert({
      id: context.id,           // Same as the uuid passed to start/
      order_id: orderId,
      status: "started",
      updated_at: new Date().toISOString()
    });
  });

  const order = yield* context.run(getOrder, orderId);

  yield* context.run(async () => {
    await supabase.from("order_progress").update({
      status: "order_fetched",
      updated_at: new Date().toISOString()
    }).eq("id", context.id);
  });

  // ... continue processing
}
```

### 7.2 Update Database at Each Step

Write progress to the database after each significant step. The UI reads from the database, not from the Resonate function directly.

```typescript
function* processOrder(context: Context, orderId: string) {
  const supabase = context.getDependency("supabase");

  // Helper to update progress
  function* updateProgress(status: string, data?: object) {
    yield* context.run(async () => {
      await supabase.from("order_progress").update({
        status,
        data,
        updated_at: new Date().toISOString()
      }).eq("id", context.id);
    });
  }

  yield* updateProgress("started");

  const order = yield* context.run(getOrder, orderId);
  yield* updateProgress("order_fetched", { order });

  const payment = yield* context.run(chargeCard, order);
  yield* updateProgress("payment_complete", { payment });

  const shipment = yield* context.run(createShipment, order);
  yield* updateProgress("shipment_created", { shipment });

  yield* updateProgress("complete");
  return { orderId, status: "complete" };
}
```

### 7.3 UI Reads from Database

The UI should poll the database table, not the `probe/` endpoint, for real-time progress. The database contains richer status information.

```typescript
// In your frontend
const { data } = await supabase
  .from("order_progress")
  .select("*")
  .eq("id", executionId)
  .single();

// data.status: "started" | "order_fetched" | "payment_complete" | ...
// data.data: step-specific data
```

Or use Supabase Realtime for live updates:

```typescript
supabase
  .channel("order_progress")
  .on("postgres_changes", {
    event: "UPDATE",
    schema: "public",
    table: "order_progress",
    filter: `id=eq.${executionId}`
  }, (payload) => {
    console.log("Progress update:", payload.new);
  })
  .subscribe();
```

### 7.4 Suggested Database Schema

```sql
CREATE TABLE execution_progress (
  id TEXT PRIMARY KEY,              -- Same as context.id / uuid
  function_name TEXT NOT NULL,
  status TEXT NOT NULL,
  data JSONB,
  error TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Enable realtime
ALTER PUBLICATION supabase_realtime ADD TABLE execution_progress;
```
