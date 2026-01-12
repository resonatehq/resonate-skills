# Writing Durable Executions with Resonate

Resonate's Durable Executions, Distributed Async Await, is a programming model based on durable functions and durable promises. Resonate enables stateful, durable executions on stateless, ephemeral infrastructure.

<Note>
  - Async await is a concurrent programming model based on functions and promises, executing on one process.
  - Distributed Async Await is a concurrent and distributed programming model based on durable functions and durable promises, executing across processes.
</Note>

## Overview

This section provides an overview of the Resonate programming model. The example below shows a Resonate function that broadcasts a countdown, sleeping between each broadcast. The example illustrates Resonate's ability to map one durable function execution onto multiple ephemeral function executions.

```typescript
// Resonate
import { Resonate, type Context } from "@resonatehq/supabase";
// Supabase
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// Create a Resonate instance
const resonate = new Resonate();

// Create a Supabase instance
const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!, 
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")
);

// Set dependencies
// Dependencies are available from anywhere in the call graph
resonate.setDependency("supabase", supabase)

// Generator function: can suspend and resume
function* countdown(context: Context, count: number, delay: number) {
  for (let i = count; i > 0; i--) {
    // broadcast current count
    yield* context.run(broadcast, i);
    // sleep (rpc semantics)
    yield* context.sleep(delay * 60 * 1000);          // Hard suspend
  }
}

// Traditional function: atomic unit of work
async function broadcast(context: Context, count: number) {
  // fetch dependency
  const supabase = context.getDependency("supabase");
  // broadcast via supabase
  await supabase.channel(context.rootId).send({ count });
}

// Register functions that can be invoked
resonate.register(countdown);

// Start the HTTP handler
resonate.httpHandler();
```

To invoke a Resonate function, send an HTTP POST request to the Resonate API start endpoint. If you do not set a uuid, a uuid will be generated. However, if you fail to capture the uuid, you will not be able to retrieve the result or to retry the invocation.

```
curl -X POST "https://{URL}/functions/v1/start" -H "Content-Type: application/json" -d '{"uuid": "countdown-1", "func": "countdown", "args": [3, 1]}'
```

The request will return the uuid of the invocation:

```
{"uuid": "countdown-1"}
```

After the countdown terminates, the call graph will look like:

```
countdown-1            resolved(param={func: countdown, args: [3, 1]}, value=void, [resonate:invoke=https://{URL}/functions/v1/flow])
├─ countdown-1.1       resolved(param={func: broadcast, args: [3]}, value=void)
├─ countdown-1.2       resolved(param=60000, value=void, [resonate:timeout=true])
├─ countdown-1.3       resolved(param={func: broadcast, args: [2]}, value=void)
├─ countdown-1.4       resolved(param=60000, value=void, [resonate:timeout=true])
├─ countdown-1.5       resolved(param={func: broadcast, args: [1]}, value=void)
└─ countdown-1.6       resolved(param=60000, value=void, [resonate:timeout=true])
```

## Programming Model

Resonate defines a minimal application programming interface (API):

```typescript
function* foo(context: Context) {
  const future = yield* context.beginRun(bar);     // started, not awaited
  const result = yield* future;                    // awaited
  return result;
}

function* bar(context: Context) {
  const future = yield* context.beginRpc(baz);     // started, not awaited
  const result = yield* future;                    // awaited
  return result;
}

function* baz(context: Context) {
  // ...
}
```

- `const future = yield* context.beginRun(func, arg1, arg2, ...)` — invokes a durable function and returns a durable promise
- `const future = yield* context.beginRpc(func, arg1, arg2, ...)` — invokes a durable function and returns a durable promise
- `const result = yield* future` — suspends until the durable promise is resolved

Commonly, a function is invoked and awaited immediately, therefore Resonate defines two shorthands:

``` typescript
// Shorthand
const result = yield* context.run(func, arg1, arg2);
// Equivalent to
const future = yield* context.beginRun(fn, arg1, arg2);
const result = yield* future;
```

``` typescript
// Shorthand
const result = yield* context.rpc(func, arg1, arg2);
// Equivalent to
const future = yield* context.beginRpc(fn, arg1, arg2);
const result = yield* future;
```

### Serialization Rules

**Anything that gets checkpointed must be serializable.**

- All return values must be serializable — promises store results for replay deduplication
- Arguments to `context.rpc()` must be serializable — they cross activation boundaries
- Arguments to `context.run()` do not need to be serializable — passed in-memory

| | Arguments | Return values |
|---|---|---|
| `context.run()` | In-memory (no serialization) | Stored in promise (must serialize) |
| `context.rpc()` | Cross boundary (must serialize) | Stored in promise (must serialize) |

**Serializable:** plain objects, arrays, strings, numbers, booleans, null

**Not serializable:** functions, closures, class instances with methods, circular references, database connections, file handles

### Structured Concurrency

Resonate functions implement structured concurrency. Everything started within a function is implicitly awaited if not explicitly awaited.

```typescript
function* parent(context: Context) {
  yield* context.beginRun(child1);  // started, not awaited
  yield* context.beginRun(child2);  // started, not awaited
  return "done";
}
```

The function does not return "done" until both `child1` and `child2` complete. This is semantically equivalent to:

```typescript
function* parent(context: Context) {
  const future1 = yield* context.beginRun(child1);
  const future2 = yield* context.beginRun(child2);
  yield* future1;  // implicit await before return
  yield* future2;  // implicit await before return
  return "done";
}
```

No orphaned work. The parent cannot complete until all its children complete.

### Futures

To avoid naming collisions with TypeScript's built-in promises, durable promises are represented by type `Future` and often referred to as futures.

## Execution Model

Resonate enables stateful, durable executions on stateless, ephemeral infrastructure: Resonate maps one stateful, durable function execution onto multiple stateless, ephemeral function executions. The term execution refers to a durable function execution, the term activation refers to an ephemeral function execution.

Soft suspend — yields control, resumes in the same activation
Hard suspend — yields control, resumes in a different activation

Colloquially, both suspend, the difference is where foo wakes up.

## When to Use Resonate

Resonate enables stateful executions on stateless infrastructure.

Supabase Edge Functions are stateless — each invocation is isolated and ephemeral, bounded by a ~60 second timeout. Resonate adds a persistence layer that checkpoints execution progress, enabling long-running, multi-step, failure-resilient processes without sacrificing the simplicity and scalability of serverless.

### When to Use Resonate Functions

Use a Resonate function when your process involves **any** of:

- **Duration** — runs longer than the edge function timeout (~60s)
- **Steps with side effects** — multiple operations that shouldn't repeat on retry
- **Waiting** — for time to pass, humans to act, or external events
- **Reliability requirements** — must complete even if infrastructure fails
- **Coordination** — multiple services or actions that must happen in order

Use a regular edge function when:

- It's a simple request/response (fetch data, return it)
- It completes in milliseconds to seconds
- Failure just means "try again" with no consequences
- There's no state to track between steps

## Core Concepts

### Activations and Executions

An **activation** is a single invocation of a Supabase Edge Function — ephemeral, stateless, bounded by timeout.

An **execution** is the full lifecycle of a Resonate function — potentially spanning multiple activations, surviving failures, and maintaining state throughout.

One execution spans one or more activations. The execution is the logical unit (your workflow). The activation is the physical unit (an edge function instance processing a task).

### Local vs Remote Invocation

`context.run()` — executes within the **same** activation. The sub-function runs in the current edge function instance. It's checkpointed (durable), but doesn't leave the process.

`context.rpc()` — executes in a **different** activation. The current activation creates a durable promise and suspends. The Resonate Server dispatches the work to another activation of the Supabase Edge Function. When that completes, the original execution can resume (possibly in yet another activation).

### Internally vs Externally Resolved Promises

**Internally resolved** — resolved by function execution within the call graph:
- `context.run(fn)` — fn executes in same activation, resolves the promise
- `context.rpc(fn)` — fn executes in another activation, resolves the promise

**Externally resolved** — resolved by something outside the call graph:
- `context.sleep(ms)` — resolved when time elapses (Resonate Server timer)
- `context.promise()` — resolved by an external actor (API call, webhook, human)

All of `rpc`, `sleep`, and `promise` terminate the current activation. The execution suspends until the promise is resolved, then resumes in a new activation.

## Syntax Reference

### Generator Functions

[TODO: function* syntax, yield* keyword, Context type]

### Invoking Functions

Shorthand — invoke and await in one step:

```typescript
const result = yield* context.run(fn, arg1, arg2);
const result = yield* context.rpc("fn", arg1, arg2);
```

Expanded form — start now, await later:

```typescript
const future = yield* context.beginRun(fn, arg1, arg2);
const result = yield* future;

const future = yield* context.beginRpc("fn", arg1, arg2);
const result = yield* future;
```

The `begin*` variants are useful when you want to start multiple operations before waiting for results.

### Sleeping

[TODO: context.sleep() duration format, examples]

### External Promises

[TODO: context.promise() usage, how to resolve externally with resonate.promises.resolve()]

### Deterministic Helpers

[TODO: context.date.now(), context.math.random() — why they exist, when to use]

### Options

[TODO: timeouts, tags]

## Setup & Structure

### File Structure

[TODO: start/, flows/, deno.json, config.toml]

### Imports

[TODO: @resonatehq/supabase vs @resonatehq/sdk, Context type]

### Function Registration

[TODO: resonate.register()]

### Entry Point

[TODO: start function creates promise via RPC, flows function executes work]

### Environment Variables

[TODO: RESONATE_URL, SUPABASE_URL]

## Style Guidelines

### Prefer Simple, Sequential Functions

```typescript
function* processOrder(context: Context, orderId: string) {
  const order = yield* context.run(getOrder, orderId);
  const payment = yield* context.run(chargeCard, order);
  const shipment = yield* context.run(createShipment, order);
  yield* context.run(sendConfirmation, order, shipment);
  return { orderId, status: "complete" };
}
```

### Prefer `run` and `rpc` Over `beginRun` and `beginRpc`

Use the shorthand unless you need concurrency. Simpler code, fewer moving parts.

### If You Begin Multiple Sub-invocations, Prefer Fork-Join

```typescript
function* processOrder(context: Context, orderId: string) {
  // Fork: start concurrent work
  const inventoryFuture = yield* context.beginRun(checkInventory, orderId);
  const fraudFuture = yield* context.beginRun(checkFraud, orderId);
  const creditFuture = yield* context.beginRun(checkCredit, orderId);
  
  // Join: await all results
  const inventory = yield* inventoryFuture;
  const fraud = yield* fraudFuture;
  const credit = yield* creditFuture;
  
  // Continue with results
  return { inventory, fraud, credit };
}
```

Avoid interleaved begins and awaits — they're hard to follow.

## Error Handling

[TODO: what happens when a function throws, retries, catching errors from sub-invocations]

## Patterns

### Human-in-the-Loop

[TODO: context.promise() example with external resolution]

### Scheduled Delays

[TODO: context.sleep() example for reminders, follow-ups]

### Fan-out / Fan-in

[TODO: parallel work with beginRun/beginRpc, collecting results]

### Compensation / Rollback

[TODO: handling failures in multi-step processes]

---

<internals>
## How Activations Work

The SDK's core loop is "execute until blocked." An activation runs your function until it encounters an externally resolved promise (`rpc`, `sleep`, or `promise`). At that point:

1. Execution state is checkpointed
2. The activation terminates  
3. The Resonate Server schedules the next activation when the blocking promise resolves

### Example: `context.run()` — Single Activation

```
Activation 1:
├─ foo 1st span
├─ bar (runs to completion)
└─ foo 2nd span
```

### Example: `context.rpc()` — Multiple Activations

```
Activation 1:
├─ foo 1st span
└─ context.rpc(bar) → suspends, activation terminates

Activation 2:
└─ bar (runs to completion, resolves promise)

Activation 3:
└─ foo 2nd span (scheduled after bar's promise resolves)
```

foo doesn't "wait" for bar. There's no blocking, no open connection, no sleeping thread. The activation ends. foo's continuation (2nd span) only exists as pending state in the Resonate Server until bar completes and triggers its scheduling.

Every `context.rpc()` is actually two scheduling events:
1. Schedule bar (the callee)
2. Schedule foo's continuation (when bar resolves)

### Why Return Values Must Serialize

When an execution replays after a crash:
1. It encounters a `context.run(fn)`
2. The promise for that call already exists with a stored result
3. Instead of re-executing `fn`, it returns the stored result

The return value *is* the promise value. Serialization is required for replay deduplication, not just for crossing activation boundaries.
</internals>
