# Resonate SDK — API Reference

TypeScript SDK for the Resonate wire protocol. Zero external dependencies.

**Package:** `@resonatehq/sdk`
**Exports:** `Resonate`, `Context`, `Descriptor` (types)

---

## Resonate Class

### Constructor

```typescript
import { Resonate, type Context, type Descriptor } from "@resonatehq/sdk";

const resonate = new Resonate({
  url: "http://localhost:8001",       // Server URL (required, or RESONATE_URL env)
  processId: "worker-1",              // Unique worker identity (default: random UUID)
  group: "default",                   // Consumer group for SSE routing
  ttl: 30000,                         // Task lease TTL in ms
  heartbeatInterval: 5000,            // Heartbeat frequency in ms
  auth: { token: "eyJhbG..." },       // JWT auth token
});
```

### Configuration Reference

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `url` | `RESONATE_URL` | **required** | Server base URL |
| `processId` | `RESONATE_PROCESS_ID` | `crypto.randomUUID()` | Unique worker identity |
| `group` | `RESONATE_GROUP` | `"default"` | Consumer group for SSE routing |
| `ttl` | `RESONATE_TTL` | `30000` | Task lease TTL in ms |
| `heartbeatInterval` | `RESONATE_HEARTBEAT_INTERVAL` | `5000` | Heartbeat interval in ms |
| `auth` | `RESONATE_TOKEN` | none | Pre-encoded JWT token |

Constructor options override env vars. Env vars override defaults.

### Methods

**`register(name, fn, options?)`** — Register a durable function. Must be called before `start()`.

```typescript
resonate.register("processOrder", function* (ctx: Context, orderId: string) {
  // workflow body
}, { version: 2 });  // optional version (default: 1)
```

**`start()`** — Start the worker. Opens SSE connection (`GET /poll/{group}/{pid}`), starts heartbeat loop. Returns a promise that resolves when connected.

**`stop()`** — Stop the worker. Aborts SSE, sends final empty heartbeat. Graceful shutdown.

**`invoke(name, args, options?)`** — Dispatch a workflow and wait for the result.

```typescript
const result = await resonate.invoke("processOrder", ["order-123"], {
  id: "order-123",                    // Promise ID (default: random UUID)
  timeoutAt: Date.now() + 3600000,    // Deadline (default: 1 hour)
  tags: { source: "api" },            // Metadata tags
});
```

Dispatching with the same `id` returns the cached result — the function body does NOT re-execute. This is idempotency.

---

## Context API

Every durable function receives a `Context` as its first parameter. Context methods return `Descriptor` objects that must be `yield`-ed.

### `yield ctx.run(fn, ...args)` — Local Step

Execute a function locally within the same task. Two forms:

```typescript
// By registered name:
const result = yield ctx.run("addOne", 5);

// Inline generator:
const result = yield ctx.run(function* (_ctx: Context) {
  return 42;
});

// With arguments:
const result = yield ctx.run(function* (_ctx: Context, x: number, y: number) {
  return x + y;
}, 10, 20);
```

**Behavior:**
- Runs synchronously within the current task (no new task created)
- Child promise ID: `{rootId}.{seq}` (deterministic, auto-incremented)
- On replay: if child already settled, uses cached value — function body does NOT re-execute
- Nested generators share the parent's preload cache

### `yield ctx.rpc(name, args?, opts?)` — Remote Call

Dispatch work to another worker group:

```typescript
const result = yield ctx.rpc("chargeCard", [orderId, amount], {
  target: "poll://any@payment-workers",  // Target worker group
  timeoutAt: Date.now() + 30000,         // Deadline (optional)
  version: 2,                            // Version (default: 1)
});
```

**Behavior:**
- Creates a child promise with `resonate:target` tag
- **Suspends** the current task until the RPC completes
- Server dispatches the child to the target worker group
- On resume: server sends a new `execute` message, engine replays from top with updated preload

**Target format:** `poll://any@{group}` — any worker in the specified group.

### `yield ctx.sleep(ms)` — Durable Sleep

```typescript
yield ctx.sleep(5000);           // 5 seconds
yield ctx.sleep(86_400_000);     // 1 day — survives crashes
```

**Behavior:**
- Creates a child promise with `resonate:timer` tag
- Server auto-resolves when timer fires
- **Suspends** the task until resolved
- Returns `undefined`

### `yield ctx.promise(id, opts?)` — External Promise

Wait for a promise that gets settled externally (webhook, approval, manual action):

```typescript
const approval = yield ctx.promise("approval-order-42", {
  timeoutAt: Date.now() + 86_400_000,  // 24 hours
  tags: { approver: "admin" },
});
```

**Behavior:**
- Creates the promise on the server if it doesn't exist (idempotent)
- **Suspends** until someone settles it externally via `promise.settle`
- Returns the decoded value from the settlement

**External settlement** (from HTTP handler, webhook, or script):

```typescript
await fetch("http://localhost:8001", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    kind: "promise.settle",
    head: { corrId: crypto.randomUUID(), version: "" },
    data: {
      id: "approval-order-42",
      state: "resolved",     // or "rejected"
      value: { headers: {}, data: encode("approved") },
    },
  }),
});

function encode(value: unknown): string {
  return btoa(unescape(encodeURIComponent(JSON.stringify(value))));
}
```

---

## Durable Function Rules

Every workflow is a generator function:

```typescript
function* myWorkflow(ctx: Context, input: string): Generator<Descriptor, string, unknown> {
  const step1 = yield ctx.run(transform, input);
  const step2 = yield ctx.rpc("validate", [step1]);
  yield ctx.sleep(1000);
  return step2;
}
```

**Must follow:**

| Rule | Why |
|------|-----|
| Use `function*` (generator), not `async function` | The engine drives execution via `yield` |
| First param is always `Context` | Engine injects it |
| `yield` each step | Without `yield`, the step never executes |
| `return` the final value | Promise resolves with it |
| `throw` to reject | Promise rejects with the error |
| No `await` inside generators | Bypasses the durable execution engine |
| No non-deterministic logic between yields | `Math.random()`, `Date.now()` differ on replay |
| Side effects inside `yield ctx.run()` | Bare side effects re-execute on replay |
| Register before `start()` | Functions received via SSE must already be registered |

---

## Deterministic Child IDs

Every `yield` produces a child promise with a deterministic ID:

```
Root ID: "order-123"
  yield ctx.run(...)    → "order-123.0"
  yield ctx.run(...)    → "order-123.1"
  yield ctx.sleep(...)  → "order-123.2"
  yield ctx.rpc(...)    → "order-123.3"
```

The sequence counter increments for each `yield` and resets to 0 on each fresh execution. This is how replay works: the engine knows child `.0` was already settled and skips re-running it.

---

## Preload Cache

When a task is acquired (or re-acquired after suspend), the server sends a `preload` array of already-settled child promises. The engine caches these and returns cached values for settled steps — this is what makes replay fast.

```
First run:  step0 (execute) → step1 (execute) → step2 (crash)
Replay:     step0 (preload) → step1 (preload) → step2 (execute) → step3 (execute)
```

On replay, steps 0 and 1 return their cached results instantly. The function body does NOT re-execute. Execution resumes from step 2.

---

## Wire Protocol Essentials

All communication uses a single `POST /` endpoint with a JSON envelope:

```json
{
  "kind": "promise.create",
  "head": { "corrId": "uuid", "version": "" },
  "data": { "id": "my-promise", "timeoutAt": 9999999999999, "tags": {} }
}
```

**Encoding:** Values are encoded as `base64(utf8(JSON))`:

```typescript
function encode(value: unknown): string {
  return btoa(unescape(encodeURIComponent(JSON.stringify(value))));
}
function decode(data: string): unknown {
  return JSON.parse(decodeURIComponent(escape(atob(data))));
}
```

**Promise states:** `pending`, `resolved`, `rejected`, `rejected_canceled`, `rejected_timedout` (all lowercase).

**SSE transport:** Workers connect via `GET /poll/{group}/{processId}`. Server pushes `execute` messages when tasks are ready.

**Key operations:**

| Operation | Purpose |
|-----------|---------|
| `promise.create` | Create a durable promise (with optional task dispatch) |
| `promise.get` | Get promise state and value |
| `promise.settle` | Resolve or reject a promise externally |
| `task.acquire` | Claim a task for processing |
| `task.fulfill` | Complete a task with result |
| `task.heartbeat` | Extend task lease |

---

## Error Handling

Errors propagate through the generator chain:

```typescript
resonate.register("resilient", function* (ctx: Context) {
  try {
    const result = yield ctx.run(riskyStep);
    return result;
  } catch (err) {
    // riskyStep threw — err is an Error with the message
    return "fallback-value";
  }
});
```

- Child throws → parent catches (or rejects if uncaught)
- `FenceLostError` (lease lost) → execution aborts silently, server re-dispatches
- Promise timeout → throws at the `yield` point, catchable with try/catch

---

## Known Limitations

| Feature | Status | Workaround |
|---------|--------|------------|
| Fan-out parallelism (`ctx.beginRun`) | Not available | `ctx.rpc` to parallel worker groups |
| Async I/O inside generators | Not available | Sync I/O only; or dedicated RPC worker |
| External settlement API (`resonate.promises.resolve`) | Not available | Direct `promise.settle` wire protocol call |
| Cross-group dispatch (`resonate.rpc`) | Not available | Direct `promise.create` with `resonate:target` tag |
| Auto-generated promise IDs | Not available | Derive from workflow ID: `approval-${orderId}` |
| `ctx.date.now()`, `ctx.math.random()` | Not available | Pass deterministic inputs as function arguments |

These are SDK limitations, not protocol limitations. The Resonate server supports all the underlying operations.

---

## Migration from Old SDK

| Old SDK | Current SDK (`@resonatehq/sdk`) | Notes |
|---|---|---|
| `yield*` | `yield` | No delegation star |
| `resonate.register(fn)` | `resonate.register("name", fn)` | Explicit name required |
| `ctx.run(asyncFn)` | `ctx.run(function* gen)` | Generators only, no async |
| `ctx.beginRun()` | `ctx.rpc()` | Fan-out via parallel workers |
| `ctx.promise({})` auto-ID | `ctx.promise("explicit-id")` | Explicit IDs required |
| `resonate.promises.resolve()` | `resonate.promises.settle(id, "resolved", { data })` | Single method with `state` discriminator (`resolved` / `rejected` / `rejected_canceled`) in `@resonatehq/sdk` v0.10.0 |
| `resonate.rpc()` | `resonate.invoke()` or direct `promise.create` | Different dispatch API |
| `UPPERCASE` states | `lowercase` states | `RESOLVED` → `resolved` |
| `timeout`, `createdOn` | `timeoutAt`, `createdAt` | Field renames |

**Mental model is the same:** Generator-based durable functions, deterministic child IDs, replay via preload cache. The transport changed (HTTP push → SSE pull) but is invisible to workflow authors.

---

## Checklist

When building a new durable workflow:

- [ ] Is the function a `function*` generator?
- [ ] Does it receive `Context` as first parameter?
- [ ] Are all steps `yield`-ed? (`yield ctx.run(...)`, not just `ctx.run(...)`)
- [ ] Are side effects inside `yield ctx.run(...)` blocks?
- [ ] Is the function registered before `start()` is called?
- [ ] For external promises: is the ID deterministic and known to the settling system?
- [ ] For RPC: is the target worker registered and running with the right group?
- [ ] Is the Resonate server running?
