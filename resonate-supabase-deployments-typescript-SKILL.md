---
name: resonate-supabase-deployments-typescript
description: Build Resonate workflows on Supabase Edge Functions (TypeScript/Deno) using the Supabase shim, start/probe endpoints, and optional DB progress tracking.
---

# Resonate Supabase TypeScript

## Overview

Build durable workflows on Supabase Edge Functions using Resonate, with start/probe endpoints and optional progress tracking in Supabase.

## Summary

- Build durable workflows on Supabase Edge Functions using the Resonate Supabase shim.
- Use when you need long-running, reliable, stepwise workflows triggered by HTTP or external events.
- Outputs a `flows/index.ts` workflow module plus start/probe endpoint usage patterns.

## Preconditions / Assumptions

- Supabase project with `supabase/functions` directory.
- Supabase Edge Functions runtime (Deno) is available.
- Resonate Server URL is known and reachable.
- Assumes a recent stable Resonate Server and `@resonatehq/supabase` shim.

## Inputs

- `RESONATE_URL`: Resonate Server URL (Supabase secret or env var).
- `SUPABASE_URL`: Supabase project URL (for DB access).
- `SUPABASE_SERVICE_ROLE_KEY`: Service key for server-side DB operations.
- Workflow name (function name to register).
- Execution ID scheme (uuid or domain id).

## Outputs

- `supabase/functions/flows/index.ts` with registered workflows and handler.
- HTTP endpoints for `start` and `probe` invocation.
- Optional DB table for progress tracking.

## Core concepts (minimal)

- Supabase Edge Functions are short-lived; Resonate makes them durable via checkpoints.
- The Resonate Supabase shim exposes `Resonate` and `Context` and provides `httpHandler()`.
- `start` and `probe` endpoints manage top-level workflow execution.
- `context.id` is the execution id (passed via `start`), useful for DB correlation.
- Durable steps are all `yield*` calls on `Context`.

## Procedure

### 1) Confirm the Supabase function layout

Intent: keep entrypoints consistent with Resonate shim routing.

Ensure this structure exists:

```
supabase/
|-- config.toml
`-- functions/
    |-- flows/
    |   |-- deno.json
    |   `-- index.ts
    |-- probe/
    |   |-- deno.json
    |   `-- index.ts
    `-- start/
        |-- deno.json
        `-- index.ts
```

Do not modify `start/` and `probe/` if they are provided by your template; only edit `flows/index.ts`.

### 2) Implement workflows in `flows/index.ts`

Intent: register durable functions and expose the Resonate handler.

```ts
// supabase/functions/flows/index.ts
import { Resonate, type Context } from "@resonatehq/supabase";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// Initialize Resonate - reads RESONATE_URL from env by default
// For token auth, pass token explicitly or set RESONATE_TOKEN env var
const resonate = new Resonate({
  url: Deno.env.get("RESONATE_URL")!,
  token: Deno.env.get("RESONATE_AUTH_TOKEN")  // JWT token if auth required
});

const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
);

resonate.setDependency("supabase", supabase);

function* processOrder(ctx: Context, orderId: string) {
  const supabase = ctx.getDependency("supabase");

  // persist progress
  yield* ctx.run(async () => {
    await supabase.from("order_progress").upsert({
      id: ctx.id,
      order_id: orderId,
      status: "started",
      updated_at: new Date().toISOString(),
    });
  });

  const order = yield* ctx.run(loadOrder, orderId);

  yield* ctx.run(async () => {
    await supabase.from("order_progress").update({
      status: "loaded",
      updated_at: new Date().toISOString(),
    }).eq("id", ctx.id);
  });

  const approval = yield* ctx.promise({ id: `approve/${ctx.id}` });
  const ok = yield* approval as boolean;
  if (!ok) throw new Error("rejected");

  return { orderId, status: "approved" };
}

async function loadOrder(_: Context, orderId: string) {
  return { id: orderId };
}

resonate.register("processOrder", processOrder);

resonate.httpHandler();
```

### 3) Trigger a workflow via `start`

Intent: start durable execution from your app or curl.

```ts
const response = await fetch(`${SUPABASE_URL}/functions/v1/start`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${SUPABASE_ANON_KEY}`,
  },
  body: JSON.stringify({
    uuid: `order/${orderId}`,
    func: "processOrder",
    args: [orderId],
  }),
});

const { uuid } = await response.json();
```

### 4) Check status via `probe`

Intent: poll or gate UI with execution state.

```ts
const response = await fetch(`${SUPABASE_URL}/functions/v1/probe`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${SUPABASE_ANON_KEY}`,
  },
  body: JSON.stringify({ uuid: `order/${orderId}` }),
});

const { status, value } = await response.json();
```

### 5) Resolve external promises (HITL or webhook)

Intent: resume a blocked workflow.

```ts
await fetch(`${RESONATE_URL}/promises/${promiseId}`, {
  method: "PATCH",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ state: "RESOLVED", value: { data: "true" } }),
});
```

## Starting Workflows

There are two patterns for starting workflows in Supabase Edge Functions:

### Pattern A: Use the start/probe template (Recommended)

If your template provides `start/` and `probe/` functions, use them:

```ts
// From your frontend or another service
const response = await fetch(`${SUPABASE_URL}/functions/v1/start`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${SUPABASE_ANON_KEY}`,
  },
  body: JSON.stringify({
    uuid: `countdown-${crypto.randomUUID()}`,
    func: "durableCountdown",
    args: [name, durationMinutes, targetTime],
  }),
});
```

The `start/` function internally calls `resonate.run()` for you.

### Pattern B: Custom Deno.serve() with resonate.run()

If you need custom routing or the template doesn't fit your needs:

```ts
// supabase/functions/flows/index.ts
import { Resonate, type Context } from "@resonatehq/supabase";

const resonate = new Resonate({
  url: Deno.env.get("RESONATE_URL")!,
  token: Deno.env.get("RESONATE_AUTH_TOKEN")
});

function* durableCountdown(ctx: Context, name: string, durationMinutes: number) {
  let remaining = durationMinutes;

  while (remaining > 0) {
    yield* ctx.run(sendWebhook, { type: "tick", name, remaining });
    yield* ctx.sleep(60 * 1000);
    remaining--;
  }

  yield* ctx.run(sendWebhook, { type: "complete", name });
  return { status: "completed", name };
}

resonate.register("durableCountdown", durableCountdown);

// Custom request handler
Deno.serve(async (req) => {
  const url = new URL(req.url);

  // Start a new workflow
  if (req.method === "POST" && url.pathname.endsWith("/start")) {
    const { name, durationMinutes } = await req.json();
    const promiseId = `countdown-${crypto.randomUUID()}`;

    // This starts AND executes the workflow
    resonate.run(promiseId, durableCountdown, name, durationMinutes);

    return new Response(JSON.stringify({ promiseId }), {
      headers: { "Content-Type": "application/json" }
    });
  }

  // List active workflows by querying Resonate server directly
  if (req.method === "GET" && url.pathname.endsWith("/list")) {
    const resonateUrl = Deno.env.get("RESONATE_URL")!;
    const token = Deno.env.get("RESONATE_AUTH_TOKEN");

    const response = await fetch(
      `${resonateUrl}/promises?id=countdown-*&state=pending&limit=50`,
      {
        headers: {
          "Content-Type": "application/json",
          ...(token && { "Authorization": `Bearer ${token}` })
        }
      }
    );

    const data = await response.json();
    return new Response(JSON.stringify(data), {
      headers: { "Content-Type": "application/json" }
    });
  }

  // Default: let Resonate shim handle workflow execution callbacks
  return resonate.handler()(req);
});
```

### Key Insight: resonate.run() vs HTTP API

**Do NOT create promises via direct HTTP API to start workflows.**

```ts
// ❌ WRONG - This creates a promise but doesn't execute workflow code
await fetch(`${RESONATE_URL}/promises`, {
  method: "POST",
  body: JSON.stringify({ id: "my-workflow", timeout: 86400000 })
});

// ✅ CORRECT - This creates the promise AND executes the workflow
resonate.run("my-workflow", myWorkflowFunction, arg1, arg2);
```

The HTTP API (`POST /promises`) is for creating standalone promises that will be resolved externally (like human-in-the-loop). To actually RUN workflow code, you must use `resonate.run()`, `resonate.rpc()`, or the `start/` template endpoint.

## Code patterns

### Durable sleep

```ts
function* reminder(ctx: Context, userId: string) {
  yield* ctx.run(sendEmail, userId);
  yield* ctx.sleep(24 * 60 * 60 * 1000);
  yield* ctx.run(sendFollowUp, userId);
}
```

### Structured concurrency (fork-join)

```ts
function* validate(ctx: Context, orderId: string) {
  const a = yield* ctx.beginRun(checkInventory, orderId);
  const b = yield* ctx.beginRun(checkFraud, orderId);
  return { inventory: yield* a, fraud: yield* b };
}
```

### Use `context.id` for DB correlation

```ts
function* track(ctx: Context, userId: string) {
  const supabase = ctx.getDependency("supabase");
  yield* ctx.run(async () => {
    await supabase.from("execution_progress").upsert({
      id: ctx.id,
      status: "started",
      user_id: userId,
    });
  });
}
```

## Verification

1) Start a workflow and receive a UUID:

```bash
curl -X POST "$SUPABASE_URL/functions/v1/start" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY" \
  -d '{"uuid":"order/123","func":"processOrder","args":["123"]}'
```

Expected: JSON with `uuid` equal to `order/123`.

2) Probe the workflow:

```bash
curl -X POST "$SUPABASE_URL/functions/v1/probe" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $SUPABASE_ANON_KEY" \
  -d '{"uuid":"order/123"}'
```

Expected: `status` is `pending`, `resolved`, or `rejected`.

3) Confirm progress record exists (if using DB tracking):

```sql
select * from order_progress where id = 'order/123';
```

Expected: a row with status `started` or later.

## Troubleshooting

### Supabase Issues
- 401/403 calling `/functions/v1/start` -> missing or wrong `SUPABASE_ANON_KEY`.
- 404 for `start`/`probe` -> functions not deployed or wrong function name.

### Resonate Server Issues
- 401 from Resonate server -> `RESONATE_AUTH_TOKEN` not set or invalid JWT.
- 401 on workflow execution -> token may be expired or server public key mismatch.
- Workflow never resumes -> promise not resolved; check `promises` endpoint.

### Auth Token Setup
```ts
// In your edge function
const resonate = new Resonate({
  url: Deno.env.get("RESONATE_URL")!,
  token: Deno.env.get("RESONATE_AUTH_TOKEN")  // Just "token", not "auth: { ... }"
});

// For direct HTTP calls to Resonate server
const headers = {
  "Content-Type": "application/json",
  "Authorization": `Bearer ${Deno.env.get("RESONATE_AUTH_TOKEN")}`
};
```

### Common Errors
- Duplicate side effects -> side effect not wrapped in `ctx.run`.
- Cached result returned -> execution id reused; use a new uuid.
- "Promise not found" -> workflow ID doesn't match or prefix is wrong.

## Pitfalls / Anti-patterns

- Using `async` durable functions instead of `function*`.
- Calling `Date.now()` or `Math.random()` inside durable functions.
- Returning non-serializable objects.
- Performing side effects outside `ctx.run`.
- Interleaving `beginRun` and immediate `yield*` in ways that defeat concurrency.

## Extensions (optional)

- Add Supabase Realtime to stream progress updates from a table.
- Add human-in-the-loop by pairing `ctx.promise()` with a public webhook.
- See `resonate-http-service-design` for HTTP gateway patterns.
