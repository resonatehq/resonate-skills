# Resonate Quickstart — Zero to Running in 5 Minutes

Resonate = single-binary server (Go + SQLite) + tiny SDK (TypeScript, zero deps). This gets you from nothing to a running durable workflow.

---

## Prerequisites

- [Go](https://go.dev) 1.21+ or download the pre-built binary from GitHub releases
- That's it. No Docker, no Postgres, no Redis.

---

## 1. Start the Server

```bash
git clone https://github.com/resonatehq/resonate.git
cd resonate
go build -o resonate-server .
./resonate-server
```

Server starts on port 8001. SQLite database created at `./resonate.db`. Zero configuration needed.

Verify it's running:
```bash
curl -s http://localhost:9090/metrics | head -5
```

---

## 2. Write Your First Worker

Create a new project:

```bash
mkdir my-worker && cd my-worker
bun init -y
```

Install the SDK:
```bash
npm install @resonatehq/sdk
```

Create `src/index.ts`:

```typescript
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001" });

// Register a durable function — each yield is a crash-safe checkpoint
resonate.register("greet", function* (ctx: Context, name: string) {
  const greeting = yield ctx.run(function* () {
    return `Hello, ${name}!`;
  });

  yield ctx.sleep(1000);  // Durable sleep — survives crashes

  const farewell = yield ctx.run(function* () {
    return `Goodbye, ${name}!`;
  });

  return { greeting, farewell };
});

// Start the worker and dispatch a workflow
await resonate.start();
console.log("Worker running...");

const result = await resonate.invoke("greet", ["World"], { id: "greet-demo" });
console.log("Result:", result);

await resonate.stop();
```

Run it:
```bash
bun run src/index.ts
```

Expected output:
```
Worker running...
Result: { greeting: "Hello, World!", farewell: "Goodbye, World!" }
```

---

## 3. What Just Happened

1. **Worker registered** "greet" function with the server
2. **`resonate.invoke()`** created a promise (`greet-demo`) and a task on the server
3. **Server dispatched** the task to the worker via SSE poll
4. **Worker executed** the generator function, checkpointing at each `yield`:
   - `yield ctx.run(...)` — checkpoint a local step
   - `yield ctx.sleep(1000)` — checkpoint a durable timer
5. **Server stored** the final result. The promise settled as `resolved`.

If the worker had crashed between steps, the server would re-dispatch the task. A new worker would replay from the top — but completed steps return their cached results instantly. Execution resumes from the first incomplete step.

---

## 4. Key Concepts

### Everything is a Promise

Every workflow, every step, every timer creates a **durable promise** on the server. Promises have:
- An **ID** (deterministic, derived from workflow ID + step sequence)
- A **state** (`pending` → `resolved` or `rejected`)
- A **value** (the result or error, persisted in SQLite)

### Generators Are the Magic

Your workflow is a generator function (`function*`). Each `yield` yields a **descriptor** to the engine. The engine:
- On first run: executes the step, stores the result as a promise
- On replay: checks if the promise exists, returns the cached result, skips the step

This is why generators — they can be re-entered at any point.

### Worker Groups

Workers in the same `group` share the workload:
```typescript
const resonate = new Resonate({ url: "http://localhost:8001", group: "order-workers" });
```
Start 3 workers in the same group → the server load-balances tasks across them. One dies → the others pick up its work.

---

## 5. Add More Patterns

### Human-in-the-Loop

```typescript
resonate.register("approval", function* (ctx: Context, orderId: string) {
  const approvalId = `approval-${orderId}`;

  // Send notification (durable step)
  yield ctx.run(function* () {
    console.log(`Approve at: http://localhost:5001/approve?promise_id=${approvalId}`);
  });

  // Suspend until human resolves the promise — no process stays alive
  const decision = yield ctx.promise(approvalId, {
    timeoutAt: Date.now() + 5 * 60 * 1000,
  });

  console.log(`Decision: ${decision}`);
  return decision;
});
```

Resolve externally (from an HTTP handler, webhook, or script):
```typescript
await fetch("http://localhost:8001", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    kind: "promise.settle",
    head: { corrId: crypto.randomUUID(), version: "" },
    data: {
      id: "approval-order-42",
      state: "resolved",
      value: { headers: {}, data: btoa(JSON.stringify("approved")) },
    },
  }),
});
```

### Durable Sleep

```typescript
resonate.register("reminder", function* (ctx: Context, userId: string) {
  yield ctx.run(sendEmail, userId, "Your trial starts now!");
  yield ctx.sleep(7 * 24 * 60 * 60 * 1000);  // 7 days — process can die, timer still fires
  yield ctx.run(sendEmail, userId, "Your trial is ending soon.");
});
```

### Cross-Service RPC

```typescript
resonate.register("orchestrator", function* (ctx: Context, orderId: string) {
  // Dispatch to a different worker group — suspends until that worker completes
  const score = yield ctx.rpc("computeScore", [orderId], {
    target: "poll://any@scoring-workers",
  });
  return { orderId, score };
});
```

---

## 6. Server Configuration

All via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `RESONATE_PORT` | `8001` | Protocol endpoint |
| `RESONATE_HOST` | `0.0.0.0` | Bind address |
| `RESONATE_DB_PATH` | `./resonate.db` | SQLite file path |
| `RESONATE_METRICS_PORT` | `9090` | Prometheus metrics |
| `RESONATE_AUTH_PUBLIC_KEY` | _(none)_ | RS256 public key for JWT auth |
| `RESONATE_TICK_INTERVAL_MS` | `1000` | Background processing interval |

---

## 7. Next Steps

- **All patterns with full code:** See `RESONATE-PATTERNS.md`
- **SDK API reference:** See `RESONATE-SDK.md`
- **Deploy to production:** See `DEPLOYMENT.md`
- **Test that durability works:** See `TESTING.md`
- **Starter templates:** Copy from `assets/resonate-worker.ts`, `assets/resonate-gateway.ts`, etc.
