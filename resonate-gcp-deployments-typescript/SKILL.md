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
6. **(Optional) Deliver output to browsers via the state bus pattern** — when the workflow needs to stream progress to a live UI.

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

## Step 6 – State bus pattern: delivering output to browsers

When the worker is a Cloud Function (short-lived, scales to zero), you can't hold an SSE or WebSocket connection for the lifetime of a workflow. The durable pattern is:

1. The workflow `ctx.run`s a `writeState` step that writes the current state to a database or realtime bus.
2. The browser subscribes directly to that bus.

This keeps the worker stateless and the client decoupled. Any change becomes durable (because `ctx.run` is durable) and observable (because the bus is live).

Firestore + the Firebase JS SDK `onSnapshot` is the lightest-weight option on GCP: one managed service, free tier covers most demos, no gateway to run. Other bus options include Pub/Sub (via Firestore-triggered functions), Supabase Realtime, or any database with change feeds.

**Worker side** — extend your existing worker with a Firestore client and a `writeState` activity:

```ts
import { Firestore } from "@google-cloud/firestore";

// Module scope: client is reused across warm invocations; keeping it inside
// the generator would reconnect on every replay and slow cold paths.
const firestore = new Firestore();
const liveDoc = firestore.collection("myapp").doc("live");

async function writeState(_ctx: Context, state: Record<string, unknown>) {
  await liveDoc.set(state);
}

function* workflow(ctx: Context) {
  // … compute some state …
  yield* ctx.run(writeState, { step: "ready", t: Date.now() });
}
```

Grant the Cloud Function's runtime service account `roles/datastore.user` on the project (Firestore Native uses Datastore IAM).

**Browser side** — minimal subscriber:

```ts
import { initializeApp } from "firebase/app";
import { getFirestore, doc, onSnapshot } from "firebase/firestore";

const app = initializeApp({ projectId: "<your-project>" /* + apiKey, etc. */ });
const db = getFirestore(app);

onSnapshot(doc(db, "myapp", "live"), (snap) => {
  const state = snap.data();
  // render state
});
```

Firestore rules should be read-only for anonymous users on the subscriber collection — no client should ever write through to it; only the worker's service account does.

**Contrast with other "browser + Resonate" patterns.** In [`example-countdown-web-ts`](https://github.com/resonatehq-examples/example-countdown-web-ts) and [`example-browser-worker-ts`](https://github.com/resonatehq-examples/example-browser-worker-ts) the worker itself runs in the browser tab. That's a different use case (a durable workflow per tab). The state bus pattern keeps the worker server-side and treats the browser as a read-only subscriber — the right shape when the workflow is shared across all viewers or must run even when no tabs are open.

## Outputs

- A deployed Cloud Function Gen 2 worker exposing an HTTP `handler` compatible with Resonate.
- The function can be used as a durable worker target by the Resonate Server, enabling long-running workflows across short-lived Cloud Function invocations.
- (If Step 6) A live Firestore document the browser can subscribe to, no gateway required.

## Reference example

[`example-chess-hero-gcp-ts`](https://github.com/resonatehq-examples/example-chess-hero-gcp-ts) — end-to-end: worker on Cloud Functions Gen 2, server on Cloud Run ([`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md)), output streamed to a browser via the state bus pattern.
