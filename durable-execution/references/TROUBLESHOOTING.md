# Troubleshooting Durable Workflows

Five-step triage for when things go wrong.

---

## The 5-Step Debug Protocol

1. **Is the server running?** — `curl -sf http://localhost:9090/metrics && echo "up" || echo "down"`
2. **Is the worker connected?** — Check worker logs for "Worker running" and SSE connection
3. **What state is the promise in?** — Query with `promise.get`
4. **What state is the task in?** — Query with `task.get` or `debug.snap`
5. **Is the target group correct?** — Compare `resonate:target` tag with worker's group

---

## Workflow Hangs (Never Completes)

The most common issue. The workflow dispatched but nothing happens.

### Cause 1: Worker Not Registered for the Function

The server dispatched a task for function `processOrder`, but no worker has registered it.

**Diagnosis:**
```bash
# Check worker logs for registration messages
# Look for: "register: processOrder"
```

**Fix:** Ensure `resonate.register("processOrder", fn)` is called before `resonate.start()`.

### Cause 2: Target Group Mismatch

The promise was created with `"resonate:target": "poll://any@workers"` but the worker connected with `group: "worker"` (singular).

**Diagnosis:**
```bash
# Check the promise's target tag
curl -s -X POST http://localhost:8001 \
  -H "Content-Type: application/json" \
  -d '{"kind":"promise.get","head":{"corrId":"d","version":""},"data":{"id":"YOUR_ID"}}' | jq '.data.promise.tags["resonate:target"]'
```

**Fix:** Make sure the `group` in `new Resonate({ group: "..." })` matches the target in the dispatch call.

### Cause 3: Missing `yield`

```typescript
// WRONG — ctx.run() returns a Descriptor, step never executes
const result = ctx.run(myStep, data);

// CORRECT — yield the Descriptor
const result = yield ctx.run(myStep, data);
```

**Fix:** Add `yield` before every `ctx.run()`, `ctx.rpc()`, `ctx.sleep()`, and `ctx.promise()`.

### Cause 4: Task Stuck in `acquired` State

A worker acquired the task but crashed before fulfilling it. The task remains `acquired` until the lease expires (default: 30 seconds).

**Diagnosis:**
```bash
curl -s -X POST http://localhost:8001 \
  -H "Content-Type: application/json" \
  -d '{"kind":"debug.snap","head":{"corrId":"d","version":""},"data":{}}' | jq '.data.taskTimeouts'
```

**Fix:** Wait for the lease timeout. The server will transition the task back to `pending` and re-dispatch it. If you need faster recovery, reduce `RESONATE_TASK_LEASE_TIMEOUT`.

### Cause 5: `async function` Instead of `function*`

```typescript
// WRONG — async functions bypass the durable execution engine
resonate.register("broken", async function (ctx, data) { ... });

// CORRECT — must be a generator function
resonate.register("working", function* (ctx, data) { ... });
```

---

## Promise States and What They Mean

| State | Meaning | Action |
|-------|---------|--------|
| `pending` | Workflow is in progress or hasn't started | Wait, or check if a worker is processing it |
| `resolved` | Workflow completed successfully | Read the value for the result |
| `rejected` | Workflow failed with an error | Read the value for the error message |
| `rejected_canceled` | Workflow was explicitly canceled | Check who/what canceled it |
| `rejected_timedout` | Promise exceeded its `timeoutAt` deadline | Increase timeout or investigate slow steps |

Query a promise:

```bash
curl -s -X POST http://localhost:8001 \
  -H "Content-Type: application/json" \
  -d '{"kind":"promise.get","head":{"corrId":"q","version":""},"data":{"id":"YOUR_PROMISE_ID"}}' | jq '.data.promise'
```

---

## Server Won't Start

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "Port already in use" | Another process on port 8001 | `lsof -i :8001` then kill it, or change `RESONATE_PORT` |
| Exits immediately | Database path not writable | `mkdir -p $(dirname $RESONATE_DB_PATH)` |
| Auth key error | Invalid PEM file | Regenerate: `openssl genrsa -out priv.pem 2048 && openssl rsa -in priv.pem -pubout -out pub.pem` |

---

## Authentication Errors

### 401 Unauthorized

- **No token:** Add `auth: { token: "..." }` to Resonate constructor
- **Expired token:** Regenerate with `jwt encode --secret @private_key.pem -A RS256 --exp='+90 days' '{"prefix":""}'`
- **Wrong key:** Token was signed with a different private key than the server's public key

### 403 Forbidden

- **Prefix mismatch:** Token's `prefix` claim doesn't match the promise ID prefix
- **Empty payload `{}`:** An empty JWT payload denies all access. Must include `"prefix": ""` for unrestricted access

---

## Unexpected Replays

Side effects (logging, API calls) re-execute on replay. This is expected behavior.

```typescript
resonate.register("myWorkflow", function* (ctx: Context) {
  // WARNING: This console.log runs on EVERY replay
  console.log("Starting workflow");

  // This only runs once — result is cached
  const result = yield ctx.run(function* () {
    console.log("This also replays, but the RESULT is cached");
    return 42;
  });

  return result;
});
```

**Fix:** Move side effects inside `yield ctx.run()` blocks. Use the outbox pattern for external calls (emails, webhooks). Accept that `console.log` inside generators will re-execute on replay.

---

## `await` Inside Generator (Silent Failure)

```typescript
// WRONG — await bypasses the durable execution engine
resonate.register("broken", function* (ctx: Context) {
  const data = await fetch("https://api.example.com/data");  // NOT durable!
  return data;
});
```

The `await` works on first run but is NOT replayed on crash recovery. The fetch call is invisible to the engine.

**Fix:** Resonate SDK generators are synchronous. For async I/O, use `ctx.rpc()` to dispatch to a dedicated worker, or accept that the operation is not replay-safe.

---

## Non-Deterministic Replay Errors

If you use `Math.random()`, `Date.now()`, or `crypto.randomUUID()` between yields, replay produces different child IDs than the original execution. The engine can't match cached results to the correct steps.

```typescript
// WRONG — different ID on replay
resonate.register("broken", function* (ctx: Context) {
  const id = crypto.randomUUID();  // Different on replay!
  yield ctx.run(processWithId, id);
});

// CORRECT — derive ID from stable inputs
resonate.register("fixed", function* (ctx: Context, orderId: string) {
  yield ctx.run(processWithId, `${orderId}-step-1`);
});
```

---

## Server Health Checks

```bash
# Is the server responding?
curl -sf http://localhost:9090/metrics | head -3

# How many requests has it processed?
curl -s http://localhost:9090/metrics | grep resonate_requests_total

# Snapshot of all internal state (promises, tasks, messages, timeouts)
curl -s -X POST http://localhost:8001 \
  -H "Content-Type: application/json" \
  -d '{"kind":"debug.snap","head":{"corrId":"snap","version":""},"data":{}}' | jq '.data | keys'
```

---

## Quick Fixes

| Problem | Quick Fix |
|---------|-----------|
| "Same ID returns old result" | Expected — idempotency. Use a new ID for a new execution |
| Worker keeps restarting | Check `journalctl -u resonate -f` for crash logs |
| Tasks re-dispatching repeatedly | Worker not calling `task.fulfill`. Check for unhandled errors |
| HITL promise never resolves | Verify the `promise.settle` call uses the correct promise ID and encoding |
| Sleep timer doesn't fire | Check `RESONATE_TICK_INTERVAL` — timers have tick-interval granularity |
| Database locked errors | Only one server process per database file. Kill duplicates |
