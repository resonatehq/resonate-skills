---
name: resonate-state-bus-pattern-typescript
description: Stream durable workflow state to browsers (or any live subscriber) via an external realtime bus. Use this pattern when the worker is short-lived or scales to zero — keeping the worker stateless, making progress observable, and decoupling clients from the workflow's lifecycle.
license: Apache-2.0
---

# Resonate State Bus Pattern (TypeScript)

## Overview

A long-running Resonate workflow often needs to surface progress to a live UI. If the worker is short-lived (Cloud Function, Lambda) or scales to zero, holding an SSE or WebSocket connection for the workflow's lifetime isn't an option — the connection outlives the worker process.

The state bus pattern decouples the two sides:

1. The workflow writes its current state to an external realtime bus via `ctx.run`.
2. The browser (or any subscriber) subscribes to that bus directly.

The worker stays stateless, progress becomes durable (`ctx.run` guarantees it), and the UI becomes a pure read view — the workflow can run with or without anyone watching.

## Mental Model

```
Worker (short-lived)             Bus                    Browser (long-lived)
   │                              │                              │
   ├─ ctx.run(writeState, s1) ──▶ │                              │
   │                              ├─ notify subscribers ───────▶ render(s1)
   │  [worker returns, scales to  │                              │
   │   zero — bus persists state] │                              │
   │                              │                              │
   ├─ ctx.run(writeState, s2) ──▶ │                              │
   │                              ├─ notify subscribers ───────▶ render(s2)
```

The workflow never talks to a client. The client never talks to the workflow. Both sides talk to the bus.

## When to use this pattern

- Your worker is serverless (Cloud Functions, Lambda, Edge Functions) and can't hold a persistent connection.
- The workflow must run whether or not anyone is subscribed (e.g. a shared game, a batch job whose output is watched by an ops dashboard).
- Multiple clients need the same live view of the same workflow.
- You want the workflow's progress to survive worker restarts and cold starts with no reconnection logic in the UI.

## When NOT to use this pattern

- The worker already runs in the browser (see [`example-browser-worker-ts`](https://github.com/resonatehq-examples/example-browser-worker-ts)) — then the client *is* the worker, and in-tab state works.
- The workflow is short enough to fit inside a single request/response — just return the result.
- The consumer is another service that can long-poll the Resonate server directly.

## Core pattern

### Step 1 — Pick a bus

Any durable realtime system with a pub/sub shape works. The only requirement is that the worker can write and the subscriber can listen.

Common options:

| Bus | When it's the right pick |
|---|---|
| **Firestore** (`onSnapshot`) | GCP-native. One managed service, generous free tier, trivial client SDK. Lightest setup when you're already on GCP. |
| **Supabase Realtime** | You're already on Supabase; the same table you're writing to can be subscribed to. |
| **Pub/Sub + Firestore sink** | Higher throughput, fan-out to multiple downstreams. |
| **Any DB with change feeds** (Mongo, Postgres LISTEN/NOTIFY via a gateway) | Existing infra dictates it. |

The rest of this skill uses Firestore for concreteness. The pattern is identical across buses — only the client libraries change.

### Step 2 — Write state from the worker

Instantiate the bus client at module scope so it's reused across warm invocations. Putting it inside the generator would reconnect on every replay and slow cold paths.

```ts
import { Firestore } from "@google-cloud/firestore";
import type { Context } from "@resonatehq/sdk";

const firestore = new Firestore();
const liveDoc = firestore.collection("myapp").doc("live");

async function writeState(_ctx: Context, state: Record<string, unknown>) {
  await liveDoc.set(state);
}

function* workflow(ctx: Context) {
  // ... compute some state ...
  yield* ctx.run(writeState, { step: "ready", t: Date.now() });
  // ... more work, more writes ...
}
```

Grant the worker's runtime service account `roles/datastore.user` on the project (Firestore Native uses Datastore IAM).

### Step 3 — Subscribe from the browser

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

Firestore rules should be read-only for anonymous users on the subscriber collection — no client should ever write through to it. Only the worker's service account does.

## Design notes

**One document per workflow, not per event.** Overwrite the document on each state write. Subscribers want the current state, not a log. If you need history, write a log *alongside* the live doc and render it separately.

**Idempotent writes.** `ctx.run` replays on failure. The `writeState` activity must be safe to re-execute with the same input. `set()` (vs `update()`) is the simple choice — replays overwrite with identical data.

**Shape the state for the UI, not the workflow.** The workflow's internal state may be richer than what the UI needs. Project to a narrow DTO in `writeState` so the subscriber payload stays small and the schema stays stable even as the workflow evolves.

**Don't stream high-frequency updates.** Firestore (and most buses) bill per write. If your workflow produces 1000 updates/second, batch them — write every N events or every M milliseconds, driven from the workflow.

## Contrast with "browser as worker"

[`example-countdown-web-ts`](https://github.com/resonatehq-examples/example-countdown-web-ts) and [`example-browser-worker-ts`](https://github.com/resonatehq-examples/example-browser-worker-ts) show a different model: the Resonate worker itself runs inside the browser tab. That's the right shape for a per-tab durable workflow (e.g. a client-side task that must survive a page reload). The state bus pattern is the opposite shape — the worker is server-side, possibly serverless, and the browser is a read-only subscriber. Use the bus pattern when the workflow is shared across viewers or must run when no one is watching.

## Reference example

[`example-chess-hero-gcp-ts`](https://github.com/resonatehq-examples/example-chess-hero-gcp-ts) — perpetual chess game as a single Resonate workflow on Cloud Functions Gen 2; move-by-move state written to Firestore; any number of browser tabs subscribe with `onSnapshot`. Deploy flow covered in [`resonate-gcp-deployments-typescript`](../resonate-gcp-deployments-typescript/SKILL.md) (worker) and [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md) (server).
