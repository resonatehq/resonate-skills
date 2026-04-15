---
name: durable-execution
description: >-
  Conceptual introduction to durable execution — what it is, why you'd want it,
  and the tradeoffs between rolling your own (just your DB) versus using a
  framework like Resonate. Read this BEFORE picking an implementation. For the
  Resonate SDK's concrete Context API (ctx.run, ctx.sleep, ctx.promise,
  structured concurrency), use resonate-basic-durable-world-usage-typescript.
license: Apache-2.0
---

# Durable Execution

Your code will crash. Durable execution means: when it restarts, it picks up where it left off — not from the beginning.

```
Traditional:  start → step1 → step2 → crash → lost
Queue-based:  start → step1 → step2 → crash → retry from top → duplicates
Durable:      start → step1 → step2 → crash → replay to step2 → continue from step3
```

Everything else — frameworks, protocols, infrastructure — is implementation detail.

---

## Do You Need This?

Score 1–5 on each dimension:

| Dimension | 1 (low) | 5 (high) |
|-----------|---------|----------|
| **Failure cost** | Retry is cheap, no side effects | Retry causes duplicates, data loss, or revenue loss |
| **Duration** | Milliseconds, single request | Hours/days/weeks, spans process lifetimes |
| **Coordination** | Single service, single step | Multiple services, human gates, external callbacks |
| **State complexity** | Stateless or simple key-value | Branching workflows, conditional logic, fan-out/fan-in |

| Total | Recommendation |
|-------|---------------|
| 4–8 | Use a task queue with idempotent handlers |
| 9–12 | Durabilize the critical path only |
| 13–20 | Durable execution is the right primitive |

**Quick check — you need this if any are true:**
- A crash mid-workflow means a customer gets charged but never receives the product
- Your workflow spans multiple HTTP requests or process lifetimes
- You need human approval gates that pause for hours or days
- Duplicate execution of a step causes real-world harm (double charges, duplicate emails)
- You have multi-step processes across multiple services that must all succeed or all roll back

---

## Choose Your Path

```
Does your workflow need...
  ├─ Just crash recovery + idempotency?
  │   └─ BAKED IN: add checkpointing to your existing DB. Zero infra cost.
  │      See: references/BAKED-IN.md
  │
  ├─ Fan-out parallelism, durable sleep, human-in-the-loop,
  │  or cross-service coordination?
  │   └─ RESONATE: single-binary server + tiny SDK. ~$5/mo on a VPS.
  │      See: references/RESONATE-QUICKSTART.md
  │
  └─ Enterprise-scale with managed infrastructure?
      └─ Temporal Cloud ($520/mo+ at scale) or AWS Step Functions.
         But consider: do you actually need that complexity?
```

| | Baked In | Resonate | Temporal |
|---|---|---|---|
| **Infra cost** | $0 (your existing DB) | ~$5/mo (VPS) to ~$170/mo (1M tasks/day) | ~$520/mo (1M tasks/day) |
| **Dependencies** | None | Single binary + SQLite | Cluster + multiple services |
| **Serverless** | Yes (any runtime) | Yes (Lambda, Edge Functions) | No (requires always-on cluster) |
| **Setup time** | Minutes | 5 minutes | Hours to days |
| **Patterns** | Checkpoint, idempotency, outbox | All 5 patterns + distributed coordination | All patterns + enterprise features |
| **Learning curve** | Low (just SQL + your code) | Low (generator functions) | High (proprietary DSL + concepts) |
| **When it fits** | Single-service, sequential workflows | Multi-service, any complexity | Large teams with dedicated infra staff |

---

## Baked In — Framework-Free Durability

Three building blocks. No framework. Just your database.

### 1. Idempotency Keys

Every operation gets a deterministic ID. Before executing, check if it already ran.

```typescript
async function runOnce<T>(db: Database, key: string, fn: () => Promise<T>): Promise<T> {
  const existing = db.query("SELECT result FROM completed_steps WHERE key = ?").get(key);
  if (existing) return JSON.parse(existing.result);

  const result = await fn();
  db.run("INSERT INTO completed_steps (key, result) VALUES (?, ?)", [key, JSON.stringify(result)]);
  return result;
}
```

### 2. Step-Level Checkpointing

Wrap each step. On crash and restart, completed steps return cached results. Execution resumes from the first incomplete step.

```typescript
async function durableCheckout(db: Database, orderId: string) {
  const inventory = await runOnce(db, `${orderId}:reserve`, () => reserveInventory(orderId));
  const payment   = await runOnce(db, `${orderId}:charge`,  () => chargeCard(orderId, inventory));
  const shipment  = await runOnce(db, `${orderId}:ship`,    () => createShipment(orderId, payment));
  const email     = await runOnce(db, `${orderId}:notify`,  () => sendConfirmation(orderId, shipment));
  return { inventory, payment, shipment, email };
}
```

### 3. Outbox Pattern

Side effects (emails, webhooks, API calls) go to a table first. A separate process delivers them exactly once.

```typescript
// Inside your workflow — write to outbox, don't send directly
db.run("INSERT OR IGNORE INTO outbox (id, payload) VALUES (?, ?)",
  [`${orderId}:confirmation-email`, JSON.stringify({ to: email, subject: "Order confirmed" })]);

// Separate delivery loop — idempotent, retryable
const pending = db.query("SELECT * FROM outbox WHERE delivered_at IS NULL").all();
for (const msg of pending) {
  await deliver(msg);  // your send logic
  db.run("UPDATE outbox SET delivered_at = ? WHERE id = ?", [Date.now(), msg.id]);
}
```

**When baked-in hits its limits:**
- You need a workflow to sleep for days without a process staying alive → need durable timers
- You need to suspend and wait for a human to approve something → need external promise resolution
- You need to fan out work across multiple services in parallel → need distributed coordination
- You want the framework to handle retries, timeouts, and replay for you → use Resonate

**Full implementation with templates:** See `references/BAKED-IN.md` and `assets/baked-in-checkpoint.ts`.

---

## With Resonate — Durable Execution Platform

Resonate is a single-binary server (Bun + SQLite, zero external deps) paired with a tiny SDK. Runs anywhere — VPS, serverless, edge functions. Costs ~$5/mo on a small VPS.

Your code is a generator function. Each `yield` is a durable checkpoint. If the process crashes, the server re-dispatches the work to any available worker, which replays from the last checkpoint.

```typescript
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001" });

resonate.register("processOrder", function* (ctx: Context, orderId: string) {
  const order   = yield ctx.run(fetchOrder, orderId);
  const payment = yield ctx.run(chargeCard, order);
  const shipment = yield ctx.run(createShipment, order);
  yield ctx.run(sendConfirmation, order.email);
  return { payment, shipment };
});

await resonate.start();
await resonate.invoke("processOrder", ["order-42"], { id: "order-42" });
```

That's it. Each `yield ctx.run(...)` is checkpointed. Crash recovery, retries, and replay are automatic.

**Get running in 5 minutes:** See `references/RESONATE-QUICKSTART.md`.
**All patterns with full code:** See `references/RESONATE-PATTERNS.md`.
**SDK API reference:** See `references/RESONATE-SDK.md`.
**Starter template:** Copy `assets/resonate-worker.ts`.

---

## The 5 Patterns

### 1. Saga — Multi-step with compensating rollbacks

Each step is checkpointed. On failure, compensate in reverse order.

```typescript
function* orderSaga(ctx: Context, orderId: string) {
  const completed: string[] = [];
  try {
    yield ctx.run(reserveInventory, orderId); completed.push("inventory");
    yield ctx.run(chargePayment, orderId);    completed.push("payment");
    yield ctx.run(createShipment, orderId);   completed.push("shipment");
    return { status: "success" };
  } catch (error) {
    for (const step of completed.reverse()) {
      yield ctx.run(compensate, step, orderId);
    }
    return { status: "rolled-back", compensated: completed };
  }
}
```

**Use when:** Multiple services must all succeed or all roll back (payments, bookings, provisioning).

### 2. Fan-Out / Fan-In — Parallel work with aggregation

Dispatch work to parallel workers via RPC. Each branch is independently durable.

```typescript
function* batchProcess(ctx: Context, items: string[]) {
  const results: string[] = [];
  for (const item of items) {
    const result = yield ctx.rpc("processItem", [item], {
      target: "poll://any@item-workers"
    });
    results.push(result);
  }
  return results;
}
```

**Use when:** Batch processing, parallel API calls, map-reduce workloads.

### 3. Human-in-the-Loop — Suspend for external signal

Workflow suspends without holding resources. Resumes when a human (or webhook) resolves the promise.

```typescript
function* approvalFlow(ctx: Context, orderId: string) {
  const approvalId = `approval-${orderId}`;
  yield ctx.run(sendApprovalEmail, orderId, approvalId);
  const decision = yield ctx.promise(approvalId, {
    timeoutAt: Date.now() + 48 * 60 * 60 * 1000  // 48 hours
  });
  if (decision === "approved") yield ctx.run(processOrder, orderId);
  return decision;
}
```

**Use when:** Approval workflows, manual review gates, external callbacks, payment confirmation.

### 4. Scheduled / Cron — Durable timers

Sleep is durable. Process can die and restart — the timer still fires.

```typescript
function* onboarding(ctx: Context, userId: string) {
  yield ctx.run(sendEmail, userId, "Welcome!");
  yield ctx.sleep(24 * 60 * 60 * 1000);  // 1 day — survives crashes
  yield ctx.run(sendEmail, userId, "Getting started tips");
  yield ctx.sleep(6 * 24 * 60 * 60 * 1000);  // 6 days
  yield ctx.run(sendEmail, userId, "How are we doing?");
}
```

**Use when:** Drip campaigns, SLA reminders, retry delays, polling loops, recurring jobs.

### 5. Entity — Long-lived mutable state

Each method call is a durable step on a persistent entity.

```typescript
function* orderLifecycle(ctx: Context, orderId: string) {
  const order = yield ctx.run(createOrder, orderId);
  yield ctx.run(validateOrder, order);
  const payment = yield ctx.run(processPayment, order);
  yield ctx.run(fulfillOrder, order, payment);
  yield ctx.run(notifyCustomer, order);
  return { orderId, status: "fulfilled" };
}
```

**Use when:** Lifecycle management, state machines, long-lived domain objects.

---

## The Hard Problems

### Versioning

You deploy new code. Old executions are mid-flight. The replay now hits different code paths than what was recorded.

**Solutions:** Version-tag your workflows. Drain in-flight executions before deploying breaking changes. Or design steps to be additive (new steps at the end, never remove or reorder existing ones).

### Side Effects at the Boundary

You sent an email in step 3. Step 4 crashes. On replay, step 3 returns the stored result — but the email was already sent.

**Solutions:**
- **Outbox pattern** — write to a durable outbox; a separate process delivers exactly once
- **Idempotency keys** — pass a deterministic key to external APIs so duplicates are no-ops
- **Accept-and-compensate** — accept that duplicates happen; send a correction if needed

### Testing Durability

How do you test that replay actually works? That your workflow survives a crash at every possible step?

**Approaches:**
- **Kill-and-resume test** — start workflow, kill the process mid-step, restart, verify it completes correctly
- **Replay unit test** — capture an execution log, replay against new code, assert same result
- **Transition tests** — enumerate all valid state transitions, verify each one produces correct output
- **Deterministic simulation testing (DST)** — inject controlled randomness across thousands of runs, verify invariants hold

**Full testing guide:** See `references/TESTING.md`.

### Observability

You cannot `console.log` your way through replay. You need to see: what step am I on, what's pending, what failed, what's the state of each promise.

**What to monitor:** Execution list (ID, status, duration), step timeline/waterfall, promise state graph, worker health, queue depth, error rates, retry storms.

---

## Efficiency — Why This Matters

Durable execution has a reputation for being expensive and complex. It doesn't have to be.

| Approach | Monthly cost at 1M tasks/day | Infrastructure |
|----------|------------------------------|----------------|
| **Baked in** (your DB) | $0 incremental | Your existing database |
| **Resonate** (self-hosted) | ~$5 (small VPS) to ~$170 (dedicated) | Single binary + SQLite |
| **Temporal Cloud** | ~$520+ | Managed cluster |
| **AWS Step Functions** | ~$250 (standard) | AWS-locked |

**Why Resonate is cheap:**
- Single binary, zero external dependencies — no Redis, no Kafka, no Kubernetes
- SQLite for storage — no database server to run or pay for
- Tiny SDK (~1300 lines, zero deps) — minimal memory footprint
- Runs on serverless (Lambda, Cloud Functions, Edge Functions) — pay only for execution time
- No cluster, no operator, no dedicated infra team

**Why baked-in is free:**
- Uses your existing database (Postgres, SQLite, MySQL)
- No additional processes, services, or infrastructure
- The "framework" is 50-80 lines of code in your app

The cheapest durable execution is the one that runs on what you already have.

---

## Anti-Patterns

- **Making everything durable** — Not every function needs crash recovery. Only durabilize the critical path. The rest can use simple retries.
- **Ignoring replay semantics** — Code that works on first run but breaks on replay: random IDs, `Date.now()` in control flow, external reads that return different values. Wrap non-deterministic operations as durable steps.
- **Treating it as a queue** — Durable execution is not a task queue. If you just need "retry this job," use a queue. Durable execution is for multi-step workflows with state.
- **Skipping idempotency on external calls** — Durable execution guarantees at-least-once execution of each step. Without idempotency keys on external APIs, you get duplicate charges, emails, and API calls.
- **Over-abstracting early** — Pick one pattern (usually saga or checkpoint), prove it works in your codebase, then expand. Don't build a generic workflow engine before you have a concrete use case.

---

## Quick Reference — Resonate Context API

| Method | Purpose | Example |
|--------|---------|---------|
| `yield ctx.run(fn, ...args)` | Local durable step (checkpoint) | `yield ctx.run(chargeCard, order)` |
| `yield ctx.rpc(name, args, opts)` | Remote durable step (cross-service) | `yield ctx.rpc("process", [item], { target: "poll://any@workers" })` |
| `yield ctx.sleep(ms)` | Durable timer (survives crashes) | `yield ctx.sleep(86_400_000)` |
| `yield ctx.promise(id, opts)` | Suspend until external resolution | `yield ctx.promise("approval-123", { timeoutAt: ... })` |

## Asset Templates

| Template | Use when... |
|----------|-------------|
| `assets/baked-in-checkpoint.ts` | Adding framework-free durability to an existing app |
| `assets/baked-in-outbox.ts` | Exactly-once side effects without a framework |
| `assets/resonate-worker.ts` | Starting a new Resonate worker from scratch |
| `assets/resonate-gateway.ts` | Building an HTTP gateway that dispatches durable workflows |
| `assets/resonate-hitl-worker.ts` | Workflow that suspends for human approval |
| `assets/resonate-saga-worker.ts` | Multi-step transaction with compensation on failure |

## References

| Reference | Load when... |
|-----------|-------------|
| `references/BAKED-IN.md` | Implementing framework-free durability with just your DB |
| `references/RESONATE-QUICKSTART.md` | Setting up Resonate from scratch |
| `references/RESONATE-PATTERNS.md` | Implementing a specific pattern with Resonate |
| `references/RESONATE-SDK.md` | Needing SDK API details, configuration, or wire protocol info |
| `references/TESTING.md` | Verifying that durability actually works under failure |
| `references/DEPLOYMENT.md` | Deploying to production (VPS, serverless, Docker) |
| `references/TROUBLESHOOTING.md` | Debugging workflow hangs, failures, or unexpected behavior |
