---
name: resonate-http-service-design-typescript
description: Design HTTP services that use Resonate durable functions behind route handlers, including routing patterns, workflow boundaries, and RPC calls to other service workers (e.g., database service). Use when building or refactoring HTTP APIs that trigger durable workflows in TypeScript.
license: Apache-2.0
---

# Resonate HTTP Service Design

## Overview

Use this skill to design HTTP services where route handlers start or await durable workflows. The HTTP server is the entrypoint; durable functions do the work and coordinate via Resonate. Downstream services (like a database service) expose their own durable functions and are invoked via RPC.

## Architecture Model

- HTTP service: Express/Fastify server handling routes and mapping requests to durable workflows.
- Worker service: Resonate worker group that runs durable functions for business logic.
- DB service: Separate worker group exposing durable DB functions via Resonate RPC.
- Resonate Server: Durable promise store and coordination hub.

```
client -> HTTP routes -> Resonate Client (beginRpc/run)
         -> worker group (durable workflow)
             -> ctx.rpc -> db worker group (durable db functions)
```

## Rules

- Route handlers are ephemeral: use Resonate Client APIs, not Context APIs.
- Durable functions are generator functions: `function*` with `yield*`.
- All external effects occur in durable steps and are awaited or explicitly detached.
- Use stable promise IDs for idempotency and replay safety.

## Route Design Patterns

### 1) Submit and poll (async HTTP)

- `POST /jobs` starts a workflow and returns `jobId`.
- `GET /jobs/:id` returns status or result.

### 2) Submit and callback

- `POST /jobs` starts a workflow and returns `jobId`.
- External system resolves a promise when done.

### 3) Webhook gate

- `POST /webhooks/service` resolves a durable promise tied to a workflow.

## Code Examples

### HTTP server entrypoints (Express)

```ts
import express from "express";
import { Resonate } from "@resonatehq/sdk";
import crypto from "node:crypto";

const app = express();
app.use(express.json());

const resonate = new Resonate({
  url: "http://localhost:8001",
  group: "api",
});

// Start workflow
app.post("/jobs", async (req, res) => {
  const jobId = `job/${crypto.randomUUID()}`;

  await resonate.beginRpc(
    jobId,
    "process-job",
    req.body,
    resonate.options({ target: "poll://any@workers" })
  );

  res.status(202).json({ id: jobId });
});

// Poll status
app.get("/jobs/:id", async (req, res) => {
  const handle = await resonate.get(req.params.id);
  const result = await handle.result();
  res.json({ id: req.params.id, result });
});

// Webhook: external system resolves promise
app.post("/webhooks/approval", async (req, res) => {
  const { promiseId, approved } = req.body;
  await resonate.promises.resolve(promiseId, approved);
  res.status(204).end();
});
```

### Worker service (durable workflow)

```ts
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001", group: "workers" });

function* processJob(ctx: Context, payload: { accountId: string }) {
  const account = yield* ctx.rpc(
    "db.getAccount",
    payload.accountId,
    ctx.options({ target: "poll://any@db" })
  );

  const result = yield* ctx.run(processAccount, account);

  const approval = yield* ctx.promise({ id: `approve/${ctx.id}` });
  const ok = yield* approval as boolean;
  if (!ok) {
    throw new Error("rejected");
  }

  yield* ctx.rpc(
    "db.saveResult",
    { id: ctx.id, result },
    ctx.options({ target: "poll://any@db" })
  );

  return { status: "done", result };
}

resonate.register("process-job", processJob);
```

### DB service (durable database functions)

```ts
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001", group: "db" });

function* getAccount(_: Context, id: string) {
  return await db.loadAccount(id);
}

function* saveResult(_: Context, record: { id: string; result: unknown }) {
  await db.save(record);
  return { ok: true };
}

resonate.register("db.getAccount", getAccount);
resonate.register("db.saveResult", saveResult);
```

## Determinism Notes

- Use `ctx.date.now()` and `ctx.math.random()` inside durable functions.
- Wrap side effects in durable steps (`ctx.run`, `ctx.rpc`).
- Ensure returned objects are serializable.

## Structured Concurrency Example

```ts
function* aggregate(ctx: Context, ids: string[]) {
  const futures = ids.map((id) =>
    ctx.beginRpc("db.getAccount", id, ctx.options({ target: "poll://any@db" }))
  );

  const results = [];
  for (const f of futures) {
    results.push(yield* f);
  }

  return results;
}
```

## Error Handling

- Throw errors for normal failures; Resonate retries by default.
- Use `ctx.options({ timeout: ... })` to bound retries.
- Treat 40900 (promise exists) as idempotency, not failure.

## Promise ID Strategy

- Use request-derived IDs for idempotency, or generate UUIDs for fire-and-forget.
- Keep IDs readable: `job/<uuid>`, `approve/<jobId>`, `db/op/<jobId>`.

## Route Checklist

- Inputs validated and serialized before RPC.
- Durable workflow entrypoint registered and reachable.
- Target group matches worker group name.
- Promise ID uniqueness guaranteed.
- Routes return 202 + ID for async workflows.
