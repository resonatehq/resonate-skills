---
name: resonate-gcp-deployments-typescript
description: Deploy Resonate TypeScript workers to Google Cloud Functions (Gen 2) using the GCP shim and connect them to a Resonate Server.
license: Apache-2.0
---

# Resonate Google Cloud TypeScript

## Overview

Deploy a TypeScript Resonate worker as an HTTP-triggered Google Cloud Function that talks to a Resonate Server.

## When to use this skill

- You have a Resonate Server running and reachable over HTTPS (local or Cloud Run).
- You want a _serverless_ worker that can resume durable workflows across invocations using the GCP Functions shim for the TypeScript SDK. [[Serverless workers](https://docs.resonatehq.io/operate/serverless-workers); [GCP deploy tutorial](https://docs.resonatehq.io/learn/deployments/google-cloud-run)]

## Assumptions & Inputs

You (the agent) should obtain or be given:

- `GCP_PROJECT_ID` – Google Cloud project ID
- `GCP_REGION` – GCP region (e.g. `us-central1`)
- `FUNCTION_NAME` – Name for the Cloud Function (e.g. `countdown-workflow`)
- `RESONATE_SERVER_URL` – Public URL of the Resonate Server HTTP API (e.g. `https://resonate-server-...a.run.app`) [[Deploy server](https://docs.resonatehq.io/learn/deployments/google-cloud-run#deploy-the-resonate-server)]
- `WORKER_ENTRY` – TS entry file exposing a `handler` function (typically `index.ts`)

Prerequisites in the project environment:

- Node.js ≥ 20
- `gcloud` CLI authenticated for `GCP_PROJECT_ID`
- `npm` or compatible package manager

## High-level flow

1. **Add the GCP worker shim** (`@resonatehq/gcp`) to the worker project. [[Serverless workers](https://docs.resonatehq.io/operate/serverless-workers)]
2. **Implement the worker** as a Resonate registration + HTTP handler export.
3. **Set the `RESONATE_URL` env var** for the function so it can reach the Resonate Server. [[Deploy Cloud Function](https://docs.resonatehq.io/learn/deployments/google-cloud-run#deploy-the-cloud-function)]
4. **Deploy the function** with `gcloud functions deploy` (Gen 2, HTTP trigger).
5. **Use the function URL as a target** when invoking workflows via the Resonate Server (if needed). [[Trigger countdown](https://docs.resonatehq.io/learn/deployments/google-cloud-run#trigger-a-countdown)]
6. **(Optional) Stream workflow output to browsers** — see [`resonate-state-bus-pattern-typescript`](../resonate-state-bus-pattern-typescript/SKILL.md) for the pattern (Firestore `onSnapshot` is the lightest GCP option).

For deploying the Resonate Server itself on GCP (Cloud Run + Cloud SQL), see [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md).

## Step 1 – Add the GCP worker shim

From the worker project root:

```bash
npm install @resonatehq/gcp
```

Use the `Resonate` class from the GCP package instead of the base SDK. [[Serverless workers](https://docs.resonatehq.io/operate/serverless-workers)]

## Step 2 – Implement the worker entry file

Create `index.ts` (or use the given `WORKER_ENTRY`) with this pattern:

```ts
import { Resonate } from "@resonatehq/gcp";
import type { Context } from "@resonatehq/sdk";

// Example durable workflow
function* countdown(ctx: Context, count: number, delayMs: number) {
  for (let i = count; i > 0; i--) {
    // Replace with your real work; this mirrors the standard countdown example
    yield* ctx.run((c: Context, j = i) => {
      console.log(`Countdown: ${j}`);
    });
    yield* ctx.sleep(delayMs);
  }
  console.log("Done!");
}

// Instantiate Resonate using the GCP shim
const resonate = new Resonate();

// Register the durable function (name "countdown" is just an example)
resonate.register("countdown", countdown);

// Export an HTTP handler compatible with Cloud Functions Gen 2
export const handler = resonate.handlerHttp();
```

This pattern is the same as the documented GCP shim usage, just with a concrete example function. [[Serverless workers](https://docs.resonatehq.io/operate/serverless-workers); [Countdown worker structure](https://docs.resonatehq.io/learn/deployments/google-cloud-run#countdown-workflow)]

## Step 3 – Ensure `RESONATE_URL` points at the server

The worker must know where the Resonate Server is. Use the `RESONATE_URL` environment variable in the function deployment, pointing to the server HTTP base URL. [[Cloud Function deploy](https://docs.resonatehq.io/learn/deployments/google-cloud-run#deploy-the-cloud-function)]

Example value:

```text
RESONATE_URL=https://resonate-server-<hash>-<region>.a.run.app
```

## Step 4 – Deploy to Google Cloud Functions (Gen 2)

From the worker project root:

```bash
gcloud functions deploy <FUNCTION_NAME> \
  --gen2 \
  --region=<GCP_REGION> \
  --runtime=nodejs22 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --allow-unauthenticated \
  --set-env-vars=RESONATE_URL=<RESONATE_SERVER_URL>
```

Example (mirrors the docs, with a generic name): [[Cloud Function deploy](https://docs.resonatehq.io/learn/deployments/google-cloud-run#deploy-the-cloud-function)]

```bash
gcloud functions deploy countdown-workflow \
  --gen2 \
  --region=us-central1 \
  --runtime=nodejs22 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --allow-unauthenticated \
  --set-env-vars=RESONATE_URL=https://resonate-server-...a.run.app
```

From the deploy output, capture the **Function URL** (under `serviceConfig.uri` or `url`). [[Cloud Function deploy](https://docs.resonatehq.io/learn/deployments/google-cloud-run#deploy-the-cloud-function)]

## Step 5 – (Optional) Invoke via Resonate CLI with the worker target

If needed, use the worker URL as the `--target` when invoking a durable function through the Resonate Server. [[Trigger countdown](https://docs.resonatehq.io/learn/deployments/google-cloud-run#trigger-a-countdown)]

Example:

```bash
resonate invoke countdown-workflow-1 \
  --func countdown \
  --arg 5 \
  --arg 60000 \
  --server <RESONATE_SERVER_URL> \
  --target <FUNCTION_URL>
```

Where:

- `--server` is the Resonate Server URL (same as `RESONATE_SERVER_URL`).
- `--target` is the Cloud Function URL from the previous step.

**Timeout note:** the default promise timeout is short. For long-running or forever-loop workflows, set `--timeout` explicitly (e.g. `--timeout 720h` for a 30-day horizon). A workflow whose timeout lapses will not be resumed.

**Pitfall:** if worker logs show `fetch failed` / `connection_error`, the server is probably returning task URLs pointing at `http://localhost:8001`. Set `--server-url` on the server side — see [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md) or [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md).

## Step 6 – (Optional) Stream output to browsers

Cloud Functions are short-lived — they can't hold an SSE or WebSocket connection for the life of a durable workflow. The durable pattern is to write workflow state to an external realtime bus (e.g. Firestore) and subscribe from the browser.

This is its own pattern, covered end-to-end in [`resonate-state-bus-pattern-typescript`](../resonate-state-bus-pattern-typescript/SKILL.md). Firestore + `onSnapshot` is the lightest GCP option; the same shape works with Supabase Realtime, Pub/Sub, or any DB with change feeds.

## Outputs

- A deployed Cloud Function Gen 2 worker exposing an HTTP `handler` compatible with Resonate.
- The function can be used as a durable worker target by the Resonate Server, enabling long-running workflows across short-lived Cloud Function invocations.

## Reference example

[`example-chess-hero-gcp-ts`](https://github.com/resonatehq-examples/example-chess-hero-gcp-ts) — end-to-end: worker on Cloud Functions Gen 2, server on Cloud Run ([`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md)), output streamed to a browser via [`resonate-state-bus-pattern-typescript`](../resonate-state-bus-pattern-typescript/SKILL.md).
