---
name: resonate-lovable-usage-prompt
description: Specialized guidance for building Resonate applications within Lovable.dev. Use when working with Lovable's AI-assisted development environment to scaffold, iterate, and deploy Resonate workflows.
---

# Resonate Lovable Usage Prompt

## Overview

This skill provides specialized guidance for building Resonate durable execution applications within [Lovable.dev](https://lovable.dev), an AI-assisted full-stack development environment. Lovable enables rapid prototyping and deployment of React + Node.js applications with built-in hosting.

## Core Philosophy: Resonate IS the Database

**Critical insight:** For workflow state, Resonate is your single source of truth. You don't need a separate database to track workflow status, pending approvals, or execution state.

```
Traditional approach (unnecessary complexity):
  Create workflow → Store in DB → Poll DB for status → Sync with Resonate

Resonate approach (simple):
  Create workflow → Query Resonate directly → Done
```

**Why this matters:**
- Query promises by ID prefix to list pending items
- Resolve promises to complete workflows
- No DB sync logic, no stale state, no race conditions

**When you DO need a database:**
- Business data that outlives workflows (user profiles, orders, products)
- Analytics and reporting
- Data that needs relational queries

**When you DON'T need a database:**
- Workflow status (use Resonate promises)
- Pending approvals (query by prefix)
- Execution state (Resonate tracks this)

## Lovable Environment Constraints

### What Lovable Provides

- React frontend (Vite + TypeScript)
- Node.js backend (Express + TypeScript)
- PostgreSQL database (Supabase)
- Built-in deployment pipeline
- AI-assisted code generation
- Git integration

### Lovable Limitations for Resonate

**Lovable does NOT provide:**
- Long-running backend processes
- WebSocket servers
- Background workers
- Cron jobs or schedulers

**Implication:** You cannot run a persistent Resonate worker inside Lovable's backend.

## Recommended Architecture for Lovable + Resonate

### Architecture A: External Resonate Server (Recommended)

**Lovable backend acts as HTTP client to external Resonate:**

```
Lovable Frontend (React)
        ↓
Lovable Backend (Express) [HTTP Client Only]
        ↓
External Resonate Server (Cloud Run, Fly.io, Railway)
        ↓
Resonate Workers (separate deployment)
```

**Lovable backend code:**

```typescript
// server/routes/workflows.ts
import { Resonate } from "@resonatehq/sdk";
import express from "express";

const router = express.Router();

// Connect to external Resonate server
const resonate = new Resonate({
  url: process.env.RESONATE_URL,  // e.g., https://resonate.example.com
  token: process.env.RESONATE_AUTH_TOKEN,  // JWT token for auth (if server requires it)
  group: "lovable-client"
});

// Start a workflow (non-blocking)
router.post("/api/workflows/start", async (req, res) => {
  const { workflowType, input } = req.body;
  const workflowId = `workflow-${Date.now()}`;

  await resonate.beginRpc(
    workflowId,
    workflowType,
    input,
    resonate.options({
      target: "poll://any@workers"  // Routes to external workers
    })
  );

  res.json({
    workflowId,
    status: "started",
    pollUrl: `/api/workflows/${workflowId}/status`
  });
});

// Check workflow status
router.get("/api/workflows/:id/status", async (req, res) => {
  try {
    const handle = await resonate.get(req.params.id);
    const result = await handle.result();

    res.json({
      workflowId: req.params.id,
      status: "completed",
      result
    });
  } catch (error) {
    res.json({
      workflowId: req.params.id,
      status: "pending"
    });
  }
});

export default router;
```

**External workers (deployed separately):**

```typescript
// workers/index.ts (deployed to Cloud Run/Fly.io)
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({
  url: process.env.RESONATE_URL,
  token: process.env.RESONATE_AUTH_TOKEN,  // JWT token for auth
  group: "workers"
});

function* processOrder(ctx: Context, input: any) {
  // Durable workflow logic
  const validated = yield* ctx.run(validateOrder, input);
  const payment = yield* ctx.run(processPayment, validated);
  const shipment = yield* ctx.run(createShipment, payment);

  return { status: "success", shipment };
}

resonate.register("processOrder", processOrder);

// Keep worker alive
process.on("SIGTERM", () => process.exit(0));
```

### Architecture B: Supabase Edge Functions (Alternative)

If using Lovable with Supabase Edge Functions, see the **resonate-supabase-deployments-typescript** skill for complete Deno-specific patterns including:
- `@resonatehq/supabase` shim usage
- `start/` and `probe/` endpoint patterns
- Deno.serve() request handling

**Limitation:** Supabase Edge Functions have 30-second timeout, so workflows must complete quickly or use async patterns.

## HTTP API vs SDK: When to Use Which

This is the most important decision when building with Lovable + Resonate.

### Use the SDK (Programming Model) When:

| Action | SDK Method | Example |
|--------|------------|---------|
| Start a workflow | `resonate.run()`, `resonate.rpc()`, `beginRun()`, `beginRpc()` | Starting an order processing workflow |
| Execute durable code | Generator functions with `ctx.run()`, `ctx.sleep()` | The workflow logic itself |
| Register workflow handlers | `resonate.register()` | Setting up workers |

**Key insight:** The SDK is for **executing workflows**. You need a process that can run the workflow code.

### Use the HTTP API Directly When:

| Action | HTTP Method | Example |
|--------|-------------|---------|
| List promises by prefix | `GET /promises?id=prefix-*` | Showing all pending approvals in UI |
| Get promise state | `GET /promises/{id}` | Checking if a workflow completed |
| Resolve a HITL promise | `PATCH /promises/{id}` | User clicking "Approve" button |
| Create a standalone promise | `POST /promises` | External system creating a promise to be resolved later |

**Key insight:** The HTTP API is for **managing promise state** without running workflow code.

### Decision Flowchart

```
Do you need to RUN workflow code (generators, ctx.run, ctx.sleep)?
  │
  ├─ YES → Use SDK: resonate.run(), resonate.rpc(), etc.
  │         (Requires a worker process that can execute the code)
  │
  └─ NO → Are you querying or resolving existing promises?
           │
           ├─ YES → Use HTTP API: GET/PATCH /promises
           │         (Can be done from any HTTP client)
           │
           └─ NO → You probably need the SDK
```

### Lovable-Specific Guidance

Since Lovable **cannot run persistent workers**, your Lovable backend should:

1. **Use SDK** to START workflows on external workers: `resonate.beginRpc()` with `target: "poll://any@workers"`
2. **Use HTTP API** to QUERY workflow state: `GET /promises?id=...`
3. **Use HTTP API or SDK** to RESOLVE promises: `PATCH /promises/{id}` or `resonate.promises.resolve()`

The actual workflow EXECUTION happens on your external workers (Cloud Run, Fly.io, etc.), not in Lovable.

## Lovable-Specific Patterns

### Pattern 1: Async Workflow with Polling

**Frontend (React):**

```tsx
// src/components/WorkflowRunner.tsx
import { useState } from "react";
import { useToast } from "@/components/ui/use-toast";

export function WorkflowRunner() {
  const [workflowId, setWorkflowId] = useState<string | null>(null);
  const [status, setStatus] = useState<"idle" | "running" | "completed">("idle");
  const { toast } = useToast();

  const startWorkflow = async () => {
    setStatus("running");

    const response = await fetch("/api/workflows/start", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        workflowType: "processOrder",
        input: { orderId: "123", items: [...] }
      })
    });

    const { workflowId } = await response.json();
    setWorkflowId(workflowId);

    // Poll for completion
    const pollInterval = setInterval(async () => {
      const statusRes = await fetch(`/api/workflows/${workflowId}/status`);
      const data = await statusRes.json();

      if (data.status === "completed") {
        clearInterval(pollInterval);
        setStatus("completed");
        toast({ title: "Workflow completed!", description: JSON.stringify(data.result) });
      }
    }, 2000);
  };

  return (
    <div>
      <button onClick={startWorkflow} disabled={status === "running"}>
        Start Workflow
      </button>
      {status === "running" && <p>Workflow running... (ID: {workflowId})</p>}
      {status === "completed" && <p>Workflow completed!</p>}
    </div>
  );
}
```

### Pattern 2: Store Workflow State in Supabase

**Track workflow progress in Lovable's built-in PostgreSQL:**

```typescript
// server/routes/workflows.ts
router.post("/api/workflows/start", async (req, res) => {
  const { workflowType, input } = req.body;
  const workflowId = `workflow-${Date.now()}`;

  // Store in database
  await supabase.from("workflows").insert({
    id: workflowId,
    type: workflowType,
    input,
    status: "pending",
    created_at: new Date().toISOString()
  });

  // Start Resonate workflow
  await resonate.beginRpc(workflowId, workflowType, input, resonate.options({
    target: "poll://any@workers"
  }));

  res.json({ workflowId });
});

// Webhook for completion (called by external worker)
router.post("/api/workflows/:id/complete", async (req, res) => {
  const { id } = req.params;
  const { result } = req.body;

  await supabase.from("workflows").update({
    status: "completed",
    result,
    completed_at: new Date().toISOString()
  }).eq("id", id);

  res.json({ success: true });
});
```

### Pattern 3: Human-in-the-Loop with Lovable UI

**Workflow creates approval request, Lovable UI resolves it:**

```typescript
// External worker
function* approvalWorkflow(ctx: Context, orderId: string) {
  const promise = yield* ctx.promise({
    id: `approval-${orderId}`,
    timeout: 24 * 60 * 60 * 1000
  });

  // Notify Lovable backend
  yield* ctx.run(async () => {
    await fetch(`${lovableBackendUrl}/api/approvals/create`, {
      method: "POST",
      body: JSON.stringify({
        orderId,
        promiseId: promise.id
      })
    });
  });

  const decision = yield* promise;
  return decision;
}

// Lovable backend
router.post("/api/approvals/resolve/:promiseId", async (req, res) => {
  const { promiseId } = req.params;
  const { approved } = req.body;

  const data = Buffer.from(JSON.stringify({ approved })).toString('base64');

  await resonate.promises.resolve(promiseId, { data });

  res.json({ success: true });
});

// Lovable frontend
function ApprovalButton({ promiseId }: { promiseId: string }) {
  const handleApprove = async () => {
    await fetch(`/api/approvals/resolve/${promiseId}`, {
      method: "POST",
      body: JSON.stringify({ approved: true })
    });
  };

  return <button onClick={handleApprove}>Approve</button>;
}
```

## Deployment Strategy

### Step 1: Deploy Resonate Server

**Options:**
- Google Cloud Run (recommended for Lovable users)
- Fly.io
- Railway
- Render

**Example (Cloud Run):**

```bash
# Deploy Resonate server
docker pull resonatehq/resonate:latest
gcloud run deploy resonate-server \
  --image resonatehq/resonate:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

### Step 2: Deploy Resonate Workers

**Same platforms, separate service:**

```bash
# Build and deploy worker
docker build -t workers .
gcloud run deploy resonate-workers \
  --image gcr.io/PROJECT/workers \
  --platform managed \
  --set-env-vars RESONATE_URL=https://resonate-server-xxx.run.app
```

### Step 3: Configure Lovable Backend

**Set environment variable in Lovable:**

```bash
RESONATE_URL=https://resonate-server-xxx.run.app
```

## Resonate HTTP API Reference (Direct Calls)

When using Lovable with an external Resonate server, you'll make direct HTTP calls. Here's the complete API reference.

### Server Configuration

```typescript
const RESONATE_URL = "https://resonate-server.example.com";  // Your Resonate server
const RESONATE_AUTH_TOKEN = process.env.RESONATE_AUTH_TOKEN; // Optional auth token

const headers = {
  "Content-Type": "application/json",
  ...(RESONATE_AUTH_TOKEN && { "Authorization": `Bearer ${RESONATE_AUTH_TOKEN}` })
};
```

### Promise States

Resonate promises have these states:

| State | Description |
|-------|-------------|
| `PENDING` | Promise created, waiting for resolution |
| `RESOLVED` | Promise completed successfully |
| `REJECTED` | Promise failed/rejected |
| `REJECTED_TIMEDOUT` | Promise exceeded timeout |
| `REJECTED_CANCELED` | Promise was canceled |

### Create a Promise

**Endpoint:** `POST /promises`

**Request:**
```json
{
  "id": "approval-request-abc123",
  "timeout": 86400000,
  "param": {
    "headers": {},
    "data": "eyJvcmRlcklkIjoiMTIzIn0="
  },
  "tags": {
    "type": "approval",
    "created_by": "user@example.com"
  }
}
```

**Notes:**
- `id`: Use a prefix convention like `approval-request-` for easy querying
- `timeout`: Epoch milliseconds from now (NOT ISO string). Example: `86400000` = 24 hours
- `param.data`: Base64-encoded JSON for initial data (optional)
- `tags`: Key-value metadata for filtering (optional)

**Response:**
```json
{
  "id": "approval-request-abc123",
  "state": "PENDING",
  "timeout": 1706486400000,
  "param": { "headers": {}, "data": "eyJvcmRlcklkIjoiMTIzIn0=" },
  "value": { "headers": {}, "data": null },
  "tags": { "type": "approval" },
  "createdOn": 1706400000000,
  "completedOn": null
}
```

**TypeScript Example:**
```typescript
async function createApprovalRequest(requestId: string, data: any) {
  const response = await fetch(`${RESONATE_URL}/promises`, {
    method: "POST",
    headers,
    body: JSON.stringify({
      id: `approval-request-${requestId}`,
      timeout: Date.now() + (24 * 60 * 60 * 1000), // 24 hours from now
      param: {
        headers: {},
        data: btoa(JSON.stringify(data)) // Base64 encode
      }
    })
  });
  return response.json();
}
```

### Resolve a Promise

**Endpoint:** `PATCH /promises/{id}`

**Request (approve):**
```json
{
  "state": "RESOLVED",
  "value": {
    "data": "eyJhcHByb3ZlZCI6dHJ1ZX0="
  }
}
```

**Request (reject):**
```json
{
  "state": "REJECTED",
  "value": {
    "data": "eyJhcHByb3ZlZCI6ZmFsc2UsInJlYXNvbiI6IkJ1ZGdldCBleGNlZWRlZCJ9"
  }
}
```

**Notes:**
- `value.data`: MUST be Base64-encoded JSON
- Set `state` to `RESOLVED` for approval, `REJECTED` for rejection

**TypeScript Example:**
```typescript
async function resolveApproval(promiseId: string, approved: boolean, reason?: string) {
  const decision = { approved, reason, resolvedAt: Date.now() };
  const encodedData = btoa(JSON.stringify(decision)); // Base64 encode

  const response = await fetch(`${RESONATE_URL}/promises/${promiseId}`, {
    method: "PATCH",
    headers,
    body: JSON.stringify({
      state: approved ? "RESOLVED" : "REJECTED",
      value: { data: encodedData }
    })
  });
  return response.json();
}
```

### Get a Single Promise

**Endpoint:** `GET /promises/{id}`

**Response:**
```json
{
  "id": "approval-request-abc123",
  "state": "PENDING",
  "timeout": 1706486400000,
  "param": { "headers": {}, "data": "eyJvcmRlcklkIjoiMTIzIn0=" },
  "value": { "headers": {}, "data": null },
  "tags": { "type": "approval" },
  "createdOn": 1706400000000,
  "completedOn": null
}
```

**TypeScript Example:**
```typescript
async function getApprovalRequest(promiseId: string) {
  const response = await fetch(`${RESONATE_URL}/promises/${promiseId}`, {
    method: "GET",
    headers
  });
  const promise = await response.json();

  // Decode the data if present
  if (promise.param?.data) {
    promise.decodedParam = JSON.parse(atob(promise.param.data));
  }
  if (promise.value?.data) {
    promise.decodedValue = JSON.parse(atob(promise.value.data));
  }

  return promise;
}
```

### List Promises by Prefix (Query)

**Endpoint:** `GET /promises?id={prefix}*&limit={n}&state={state}`

**Query Parameters:**
- `id`: Prefix pattern with wildcard, e.g., `approval-request-*`
- `limit`: Max results (default 10, max usually 100)
- `state`: Filter by state: `pending`, `resolved`, `rejected`

**Example Queries:**
```
GET /promises?id=approval-request-*&limit=50
GET /promises?id=approval-request-*&state=pending&limit=20
GET /promises?id=countdown-*&state=pending
```

**Response:**
```json
{
  "promises": [
    {
      "id": "approval-request-abc123",
      "state": "PENDING",
      "timeout": 1706486400000,
      ...
    },
    {
      "id": "approval-request-def456",
      "state": "RESOLVED",
      ...
    }
  ],
  "cursor": "next-page-token"
}
```

**TypeScript Example:**
```typescript
async function listPendingApprovals() {
  const response = await fetch(
    `${RESONATE_URL}/promises?id=approval-request-*&state=pending&limit=50`,
    { method: "GET", headers }
  );
  const { promises } = await response.json();

  // Decode all param data
  return promises.map(p => ({
    ...p,
    decodedParam: p.param?.data ? JSON.parse(atob(p.param.data)) : null
  }));
}
```

### Complete Lovable Pattern: No Database Needed

```typescript
// Create approval - Resonate is the source of truth
app.post("/api/approvals", async (req, res) => {
  const requestId = crypto.randomUUID();
  const promise = await createApprovalRequest(requestId, req.body);
  res.json({ requestId, promiseId: promise.id });
});

// List pending approvals - Query Resonate directly
app.get("/api/approvals", async (req, res) => {
  const approvals = await listPendingApprovals();
  res.json(approvals);
});

// Get single approval
app.get("/api/approvals/:id", async (req, res) => {
  const approval = await getApprovalRequest(`approval-request-${req.params.id}`);
  res.json(approval);
});

// Resolve approval
app.post("/api/approvals/:id/resolve", async (req, res) => {
  const { approved, reason } = req.body;
  await resolveApproval(`approval-request-${req.params.id}`, approved, reason);
  res.json({ success: true });
});
```

### Timeout Calculation

**IMPORTANT:** Timeouts are absolute epoch milliseconds, not durations.

```typescript
// ❌ WRONG - This is a duration, not a timestamp
const timeout = 24 * 60 * 60 * 1000;

// ✅ CORRECT - Absolute timestamp (now + duration)
const timeout = Date.now() + (24 * 60 * 60 * 1000);

// Common patterns:
const in1Hour = Date.now() + (60 * 60 * 1000);
const in24Hours = Date.now() + (24 * 60 * 60 * 1000);
const in7Days = Date.now() + (7 * 24 * 60 * 60 * 1000);
```

## Lovable AI Prompt Templates

### Prompt 1: Scaffold Resonate Client

```
Create a Resonate client module in server/lib/resonate.ts that:
1. Connects to process.env.RESONATE_URL
2. Uses group "lovable-client"
3. Exports a configured Resonate instance
4. Includes TypeScript types for workflow inputs/outputs
```

### Prompt 2: Create Workflow API Routes

```
Create Express routes in server/routes/workflows.ts that:
1. POST /api/workflows/start - starts a workflow via resonate.beginRpc
2. GET /api/workflows/:id/status - checks workflow status
3. POST /api/workflows/:id/cancel - cancels a workflow
4. Include error handling and TypeScript types
```

### Prompt 3: Build React UI for Workflows

```
Create a React component WorkflowDashboard that:
1. Lists all workflows from Supabase "workflows" table
2. Shows workflow status (pending/running/completed/failed)
3. Allows starting new workflows
4. Polls for status updates every 3 seconds
5. Uses shadcn/ui components (button, card, badge)
```

## Example: Complete Lovable + Resonate App

**Use case:** Order processing with approval

### 1. External Worker (Cloud Run)

```typescript
// worker/index.ts
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({
  url: process.env.RESONATE_URL,
  token: process.env.RESONATE_AUTH_TOKEN,  // JWT token for auth
  group: "workers"
});

function* processOrder(ctx: Context, order: any) {
  // Validate
  const valid = yield* ctx.run(validateOrder, order);

  // Create approval promise
  const approval = yield* ctx.promise({
    id: `approval-${order.id}`,
    timeout: 24 * 60 * 60 * 1000
  });

  // Notify Lovable
  yield* ctx.run(notifyLovable, order.id, approval.id);

  // Wait for human approval
  const decision = yield* approval;

  if (!decision.approved) {
    return { status: "rejected" };
  }

  // Process payment
  const payment = yield* ctx.run(chargePayment, order);

  return { status: "approved", payment };
}

resonate.register("processOrder", processOrder);
```

### 2. Lovable Backend

```typescript
// server/routes/orders.ts
import { resonate, supabase } from "../lib";

router.post("/api/orders", async (req, res) => {
  const order = req.body;
  const orderId = `order-${Date.now()}`;

  // Store in database
  await supabase.from("orders").insert({
    id: orderId,
    data: order,
    status: "pending"
  });

  // Start workflow
  await resonate.beginRpc(
    orderId,
    "processOrder",
    order,
    resonate.options({ target: "poll://any@workers" })
  );

  res.json({ orderId });
});

router.post("/api/approvals/:promiseId", async (req, res) => {
  const { promiseId } = req.params;
  const { approved } = req.body;

  const data = Buffer.from(JSON.stringify({ approved })).toString('base64');
  await resonate.promises.resolve(promiseId, { data });

  res.json({ success: true });
});
```

### 3. Lovable Frontend

```tsx
// src/pages/Orders.tsx
import { useState, useEffect } from "react";
import { Button } from "@/components/ui/button";

export function Orders() {
  const [orders, setOrders] = useState([]);

  const createOrder = async () => {
    const response = await fetch("/api/orders", {
      method: "POST",
      body: JSON.stringify({
        items: ["item1", "item2"],
        total: 100
      })
    });

    const { orderId } = await response.json();
    // Refresh list
  };

  return (
    <div>
      <Button onClick={createOrder}>Create Order</Button>
      <OrderList orders={orders} />
    </div>
  );
}
```

## Common Lovable + Resonate Patterns

1. **Lovable as orchestration UI** - Resonate for durable execution
2. **Supabase for state** - Resonate for workflows
3. **Polling for updates** - webhooks for completions
4. **External workers** - Lovable as control plane

## Limitations to Explain

When building with Lovable + Resonate, tell users:

1. Lovable backend is stateless HTTP - cannot run persistent workers
2. External Resonate deployment required
3. Polling needed for workflow status (no WebSockets in Lovable)
4. Database updates via Supabase for real-time UI

## Quick Start Checklist

- [ ] Deploy Resonate server (Cloud Run recommended)
- [ ] Deploy Resonate workers (same platform)
- [ ] Add RESONATE_URL to Lovable environment variables
- [ ] Create Resonate client module in Lovable backend
- [ ] Create workflow API routes in Lovable backend
- [ ] Build React UI components for workflow management
- [ ] Add Supabase table for workflow state tracking
- [ ] Test end-to-end workflow execution

## Summary

Lovable + Resonate enables rapid development of durable execution applications by:
- Using Lovable for UI and HTTP API layer
- Using Resonate for durable workflows and coordination
- Deploying workers separately on Cloud Run/Fly.io
- Connecting via HTTP APIs and promise resolution

**Key principle:** Lovable handles the user interface, Resonate handles the durable execution.
