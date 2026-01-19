---
name: resonate-operate-deploy
description: Operate and deploy Resonate servers and workers across environments (local, VM, container, Kubernetes, Cloud Run, serverless shims). Use when configuring server settings, choosing transports, or deploying workers with TypeScript.
---

# Resonate Operate and Deploy

## Overview

Use this skill to deploy and operate Resonate in specific environments. Separate concerns: the Resonate Server (durable promises, coordination) and workers (TypeScript SDK processes).

## Architecture Basics

- Server: stores promises, coordinates invocations, exposes API and poller endpoints.
- Workers: connect to the server, register functions, and execute durable work.
- Transport: default is HTTP long polling; Kafka and SQS are optional via plugins.

## Environment Selection

- Local dev: `resonate dev` (in-memory SQLite, resets on restart).
- Single VM: run server binary with `resonate serve --config resonate.yml` and a process manager.
- Container/Docker: run `resonate` image and mount a config file.
- Kubernetes: deploy the server with a Deployment/Service and configure a durable store.
- Cloud Run / Cloud Functions: run server as Cloud Run service; workers as HTTP-triggered functions.
- Serverless workers: use shim packages (`@resonatehq/aws`, `@resonatehq/gcp`, `@resonatehq/cloudflare`, `@resonatehq/supabase`).

## Server Configuration Essentials

- Set `system.url` to the public base URL of the server.
- Configure API and poller addresses; expose required ports (default 8001 API, 8002 poller).
- Choose a durable store: SQLite for dev, PostgreSQL for production.
- Configure HTTP Basic Auth when exposing the server publicly.
- Keep `metricsAddr` enabled for metrics scraping.

## Worker Configuration Essentials

- Set `RESONATE_URL` or pass `url` in the client constructor.
- Set a `group` for each worker process to enable routing and load balancing.
- For serverless workers, export the shim handler and set `RESONATE_URL` via env vars.

## Transport Options

- Default: HTTP long polling (works everywhere with minimal setup).
- Kafka: enable server plugin and use the TS Kafka transport plugin.
- SQS: enable server plugin and configure workers accordingly.

## Deployment Checklist

- Verify server API reachability with a simple request to `/promises`.
- Ensure `system.url` matches the public URL clients use.
- Run at least one worker in the target group before invoking.
- Use a stable promise ID scheme to avoid accidental id reuse.
- For production, configure persistent storage and authentication.

## Code Examples

### Local dev (server + worker)

```bash
resonate dev
```

```ts
import { Resonate } from "@resonatehq/sdk";

const resonate = new Resonate({ url: "http://localhost:8001", group: "worker" });
```

### Minimal resonate.yml

```yaml
system:
  url: "https://your-domain:8001"

api:
  subsystems:
    http:
      config:
        addr: ":8001"
        auth:
          user: pass

    grpc:
      config:
        addr: ":50051"

aio:
  subsystems:
    sender:
      config:
        plugins:
          poll:
            config:
              addr: ":8002"
              auth:
                user: pass

metricsAddr: ":9090"
```

### Docker run

```bash
docker run -p 8001:8001 -p 8002:8002 -p 9090:9090 \
  resonatehqio/resonate serve
```

### Kubernetes (server)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resonate
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resonate
  template:
    metadata:
      labels:
        app: resonate
    spec:
      containers:
        - name: resonate
          image: ghcr.io/resonatehq/resonate:v0.5.0
          args:
            - "serve"
            - "--aio-store=postgres"
```

### Cloud Run server

```bash
gcloud run deploy resonate-server \
  --image=resonatehqio/resonate \
  --region=<region> \
  --allow-unauthenticated \
  --port=8001 \
  --args="serve"
```

### Cloud Function worker

```bash
gcloud functions deploy countdown-workflow \
  --gen2 \
  --runtime=nodejs22 \
  --trigger-http \
  --allow-unauthenticated \
  --set-env-vars=RESONATE_URL=<server-url>
```

### Serverless shim (AWS example)

```ts
import { Resonate } from "@resonatehq/aws";
import type { Context } from "@resonatehq/sdk";

const resonate = new Resonate();

function* task(_: Context, id: string) {
  return id;
}

resonate.register("task", task);

export const handler = resonate.httpHandler();
```

## Operations

- Upgrade server with package manager or by replacing the binary.
- Monitor metrics on the configured metrics port and watch server logs.
- Use the CLI for smoke tests: `resonate invoke`, `resonate promises get`, `resonate tree`.
