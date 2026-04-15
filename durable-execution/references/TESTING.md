# Testing Durability

How to verify that your durable workflows actually survive crashes, replay correctly, and maintain consistency. This is the part most tutorials skip.

---

## The Question

You built a durable workflow. You tested it on the happy path. But have you tested:

- What happens when the process crashes between step 2 and step 3?
- Do completed steps return cached results on replay?
- Does the saga compensate correctly when step 4 fails after step 3 succeeds?
- Can your workflow handle a worker dying mid-execution and another worker picking it up?

If you haven't tested these, you don't have durable execution — you have a workflow that works when nothing goes wrong.

---

## Level 1: Kill-and-Resume Test (Manual)

The simplest proof. Start a workflow, kill the worker mid-execution, restart it, verify it resumes from where it left off.

### Setup

```bash
# Terminal 1: Start Resonate server
cd resonate && ./resonate-server

# Terminal 2: Start the worker
cd my-worker && bun run src/index.ts
```

### Test Protocol

1. **Start a multi-step workflow** (at least 5 steps with visible logging):

```typescript
resonate.register("countdown", function* (ctx: Context, total: number) {
  for (let i = 1; i <= total; i++) {
    yield ctx.run(function* () {
      console.log(`Step ${i}/${total} — completed at ${new Date().toISOString()}`);
      return { step: i };
    });
    if (i < total) yield ctx.sleep(2000);  // 2s between steps — time to kill
  }
  return { status: "done", steps: total };
});
```

2. **Dispatch the workflow:**
```bash
curl -X POST "http://localhost:5001/start?id=test-kill-1&func=countdown"
```

3. **Kill the worker mid-execution** (after step 2 or 3 completes):
```bash
# Ctrl+C in Terminal 2, or:
kill -9 $(pgrep -f "bun run src/index.ts" | head -1)
```

4. **Restart the worker:**
```bash
cd my-worker && bun run src/index.ts
```

5. **Verify:**
   - Steps 1 and 2 should NOT log again (they're in the preload cache)
   - Execution should resume from step 3 (the first incomplete step)
   - The workflow should complete successfully

### What to Look For

| Observation | Meaning |
|-------------|---------|
| Steps 1-2 log again on restart | **FAIL** — steps are re-executing instead of replaying from cache |
| Step 3 starts immediately | **PASS** — preload cache is working |
| Workflow completes with correct result | **PASS** — full replay + resume works |
| Workflow hangs after restart | Worker not reconnecting, or task stuck in `acquired` state (wait for lease timeout) |

### Automated Kill-and-Resume

See `scripts/test-durability.sh` for an automated version of this test.

---

## Level 2: Replay Unit Tests

Verify that your workflow produces the same result when replayed from a cached execution log.

### Approach

1. Run the workflow once and record the promise states
2. Run it again with the same ID — it should return the cached result instantly
3. Verify the function bodies did NOT re-execute

```typescript
import { test, expect } from "bun:test";

test("idempotent replay returns cached result", async () => {
  const resonate = new Resonate({ url: "http://localhost:8001" });

  let executionCount = 0;
  resonate.register("idempotentTest", function* (ctx: Context) {
    executionCount++;
    const result = yield ctx.run(function* () {
      return 42;
    });
    return result;
  });

  await resonate.start();

  // First invocation — executes the function body
  const result1 = await resonate.invoke("idempotentTest", [], { id: "replay-test-1" });
  expect(executionCount).toBe(1);

  // Second invocation with same ID — returns cached result
  const result2 = await resonate.invoke("idempotentTest", [], { id: "replay-test-1" });
  // executionCount may or may not increment (depends on SDK internals),
  // but the result should be identical
  expect(result2).toEqual(result1);

  await resonate.stop();
});
```

### Testing Error Paths

```typescript
test("saga compensates on step failure", async () => {
  const resonate = new Resonate({ url: "http://localhost:8001" });
  const compensated: string[] = [];

  resonate.register("failingSaga", function* (ctx: Context) {
    const completed: string[] = [];
    try {
      yield ctx.run(function* () { return "a"; });
      completed.push("step-a");
      yield ctx.run(function* () { return "b"; });
      completed.push("step-b");
      yield ctx.run(function* () { throw new Error("step-c failed"); });
    } catch {
      for (const step of completed.reverse()) {
        yield ctx.run(function* () {
          compensated.push(step);
          return { compensated: step };
        });
      }
      return { status: "rolled-back" };
    }
  });

  await resonate.start();
  const result = await resonate.invoke("failingSaga", [], { id: "saga-fail-1" });

  expect(result).toEqual({ status: "rolled-back" });
  expect(compensated).toEqual(["step-b", "step-a"]);  // Reverse order

  await resonate.stop();
});
```

---

## Level 3: Transition Tests

Enumerate all valid state transitions and verify each one produces the correct output. This is the standard approach for testing state-machine semantics in distributed systems primitives.

### Concept

A transition test defines:
- An **initial state** (e.g., promise is `pending`, task is `acquired`)
- An **operation** (e.g., `promise.settle` with `state: "resolved"`)
- An **expected result** (e.g., promise transitions to `resolved`, status 200)

```
State: promise(pending) + task(acquired)
Operation: task.fulfill { state: "resolved", value: "hello" }
Expected: promise → resolved, task → fulfilled, status 200
```

### Why This Matters

Manual testing covers the happy path. Transition tests cover the edges:
- What happens when you try to settle an already-resolved promise?
- What happens when a task lease expires during fulfillment?
- What happens when two workers try to acquire the same task?

### Building Your Own

For baked-in durability, test the `runStep` checkpoint function:

```typescript
test("runStep returns cached result on replay", async () => {
  const { runStep, cleanup } = createWorkflowRunner(":memory:");

  let callCount = 0;
  const result1 = await runStep("wf-1", "step-a", () => { callCount++; return 42; });
  const result2 = await runStep("wf-1", "step-a", () => { callCount++; return 99; });

  expect(result1).toBe(42);
  expect(result2).toBe(42);  // Cached, not 99
  expect(callCount).toBe(1); // Function only ran once

  cleanup("wf-1");
});

test("runStep executes steps in order on replay", async () => {
  const { runStep, getCompletedSteps, cleanup } = createWorkflowRunner(":memory:");

  await runStep("wf-2", "reserve", () => ({ reserved: true }));
  await runStep("wf-2", "charge", () => ({ charged: true }));
  // Simulate crash here — step "ship" was never reached

  // On restart, completed steps return cached results:
  const steps = getCompletedSteps("wf-2");
  expect(steps).toEqual(["reserve", "charge"]);

  // Next call to runStep("wf-2", "ship", ...) will execute the function
  const shipResult = await runStep("wf-2", "ship", () => ({ shipped: true }));
  expect(shipResult).toEqual({ shipped: true });

  cleanup("wf-2");
});
```

---

## Level 4: Fuzz / Differential Testing

The gold standard. Run thousands of random operation sequences against both your server and a reference model. Any divergence is a bug.

### How `resonate-test-fuzz` Works

```
┌───────────────┐     random ops     ┌──────────────────┐
│  Reference    │◄───────────────────│  Test Generator   │
│  Model        │    (same ops)      │  (random seed)    │
└───────┬───────┘                    └────────┬──────────┘
        │                                     │
   expected state                        actual state
        │                                     │
        └──────────── compare ────────────────┘
                     (deep equal)
```

1. Generate random sequences of valid operations (promise.create, task.acquire, promise.settle, etc.)
2. Execute each sequence against the **reference model** (in-memory state machine)
3. Execute the same sequence against **your server** (HTTP calls)
4. Compare the responses — they must be identical
5. Repeat with different random seeds (1000+ iterations)

### What Fuzz Tests Catch

- Race conditions in concurrent operations
- Edge cases in state transition logic
- Off-by-one errors in version numbers
- Incorrect handling of expired leases
- Ordering bugs in callback/listener delivery

### Running Transition + Fuzz Tests

The general pattern for running a transition/fuzz harness against a Resonate server:

```bash
# Transition tests: enumerate every valid state transition deterministically
#   e.g. ~300 cases covering promise, task, and schedule primitives
npx tsx path/to/tran.ts \
  --client model --client http \
  --config '{"baseUrl":"http://localhost:8001"}'

# Fuzz tests: random operation sequences over many iterations to find
# edge cases the deterministic set didn't cover
npx tsx path/to/fuzz.ts \
  --client model --client http \
  --config '{"baseUrl":"http://localhost:8001"}' \
  --iterations 1000 --steps 250
```

The two clients (`model` — the reference in-memory state machine, and `http` — the real server) are run against the same operation sequence and their outputs compared. Any divergence is a bug in either the model or the server.

---

## Level 5: Deterministic Simulation Testing (DST)

Inject controlled randomness across thousands of runs. Control time, network failures, and crash points. Verify invariants hold under chaos.

### Concept

DST controls all sources of non-determinism:
- **Time** is injected (virtual clock), not read from the system
- **Random decisions** use a seeded PRNG (reproducible with seed)
- **Network failures** are injected at controlled points
- **Crash/restart** is simulated by killing and restarting the client

```
seed=42 → same random decisions → same test sequence → reproducible
seed=43 → different random decisions → different test sequence → more coverage
```

### What DST Validates

| Invariant | What it means |
|-----------|---------------|
| HTTP_STATUS | Every operation returns the expected status code |
| IDEMPOTENT_STATE | Re-submitting the same operation returns the same result |
| PROMISE_STATE_MONOTONICITY | Promises only transition forward (pending → resolved/rejected) |
| TASK_VERSION_MONOTONICITY | Task versions only increase, never decrease |
| OCC_CORRECTNESS | Optimistic concurrency control rejects stale versions |
| SETTLED_AT_PENDING | Settled promises always have a `settledAt` timestamp |

### When to Use Each Level

| Level | Effort | Coverage | When |
|-------|--------|----------|------|
| Kill-and-resume | 5 min | Basic crash recovery | Every new workflow |
| Replay unit tests | 30 min | Idempotency + error paths | Before shipping |
| Transition tests | 2-4 hours | All state transitions | Building a server/framework |
| Fuzz testing | 4-8 hours | Random edge cases | Building a server/framework |
| DST | Days | Full chaos coverage | Building a production server |

For application developers (not framework builders), levels 1-2 are sufficient. Level 3 adds confidence for critical workflows. Levels 4-5 are for server implementors.

---

## Testing Checklist

Before deploying a durable workflow to production:

- [ ] **Happy path works** — workflow completes with correct result
- [ ] **Kill-and-resume works** — kill worker mid-step, restart, verify resume from checkpoint
- [ ] **Idempotent replay works** — same workflow ID returns cached result without re-execution
- [ ] **Error paths work** — saga compensates correctly, errors propagate as expected
- [ ] **Timeout works** — HITL promises time out correctly, durable sleeps fire at the right time
- [ ] **Side effects are guarded** — external calls use idempotency keys or the outbox pattern
- [ ] **No non-determinism** — no `Math.random()`, `Date.now()`, or `crypto.randomUUID()` in control flow between yields
