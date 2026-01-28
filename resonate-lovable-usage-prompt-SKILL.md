---
name: resonate-lovable-usage-prompt
description: Specialized guidance for building Resonate applications within Lovable.dev. Use when working with Lovable's AI-assisted development environment to scaffold, iterate, and deploy Resonate workflows.
---

# Resonate Lovable Usage Prompt

## Overview

This skill provides specialized guidance for building Resonate durable execution applications within [Lovable.dev](https://lovable.dev), an AI-assisted full-stack development environment. Lovable enables rapid prototyping and deployment of React + Node.js applications with built-in hosting.

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

**Use Resonate's Supabase shim within Lovable:**

```typescript
// supabase/functions/workflow/index.ts
import { Resonate } from "@resonatehq/supabase";

export default async (req: Request) => {
  const resonate = new Resonate({
    url: Deno.env.get("RESONATE_URL")!
  });

  // Handle workflow invocation
  // ...
};
```

**Limitation:** Supabase Edge Functions have 30-second timeout, so workflows must complete quickly or use async patterns.

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
