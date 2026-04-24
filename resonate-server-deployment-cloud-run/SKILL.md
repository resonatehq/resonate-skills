---
name: resonate-server-deployment-cloud-run
description: Deploy the Resonate server to Google Cloud Run with Cloud SQL Postgres storage. Covers Dockerfile reuse, the Cloud SQL Auth Proxy Unix socket, mandatory server URL, JWT authentication via Secret Manager, and configuration through `RESONATE_` environment variables.
license: Apache-2.0
---

# Resonate Server Deployment on Cloud Run

## Overview

Deploy the Resonate server ([`resonatehq/resonate`](https://github.com/resonatehq/resonate), Rust) as a Google Cloud Run service backed by Cloud SQL Postgres. Distinct deployment target from Linux + systemd — see [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) for the VM pattern.

Companion skill: workers that talk to this server deploy via [`resonate-gcp-deployments-typescript`](../resonate-gcp-deployments-typescript/SKILL.md).

## When to use this skill

- You want the Resonate server on GCP without managing a VM.
- You want durable storage in Cloud SQL Postgres instead of on-disk SQLite.
- Your workers run on Cloud Functions, Cloud Run, or any other environment that can reach a Cloud Run service over HTTPS.

## When NOT to use this skill

- You manage the host yourself (VM, bare metal, homelab) — use [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) for the systemd pattern.
- You're deploying TypeScript **workers** (not the server itself) to GCP — use [`resonate-gcp-deployments-typescript`](../resonate-gcp-deployments-typescript/SKILL.md).

## Assumptions & Inputs

You (the agent) should obtain or be given:

- `GCP_PROJECT_ID` – GCP project
- `GCP_REGION` – region (e.g. `us-central1`)
- `CLOUD_SQL_INSTANCE` – Cloud SQL Postgres instance name (e.g. `resonate-db`)
- `PG_USER`, `PG_PASSWORD`, `PG_DATABASE` – Postgres credentials
- `SERVICE_NAME` – Cloud Run service name (e.g. `resonate-server`)

Prerequisites:

- `gcloud` authenticated to `GCP_PROJECT_ID`
- Cloud Run, Cloud SQL Admin, and Secret Manager APIs enabled
- A Cloud SQL Postgres 16+ instance, with `PG_DATABASE` created
- Local clone of `resonatehq/resonate` (used as build source — the repo ships its own Dockerfile)

## Configuration model

The Resonate server reads all settings from `RESONATE_`-prefixed environment variables (figment, `__` for nesting). There is no need for `--args` overrides or a custom Dockerfile wrapper — pass config through `gcloud run deploy --set-env-vars` and mount secrets with `--set-secrets`.

| Env var | Equivalent flag | Purpose |
|---|---|---|
| `RESONATE_SERVER__URL` | `--server-url` | Mandatory public URL; the default `http://localhost:8001` silently breaks workers |
| `RESONATE_STORAGE__TYPE` | `--storage-type` | `sqlite` (default) or `postgres` |
| `RESONATE_STORAGE__POSTGRES__URL` | `--storage-postgres-url` | Postgres connection string |
| `RESONATE_AUTH__PUBLICKEY` | `--auth-publickey` | Path (inside the container) to the mounted JWT public key |

## High-level flow

1. **Clone the server repo** – the upstream Dockerfile builds the Rust server binary.
2. **Smoke-test deploy (SQLite)** – confirm the service starts and is reachable before wiring storage.
3. **Attach Cloud SQL and switch to Postgres** – connection URL lives in Secret Manager.
4. **Wire up JWT auth** – public key mounted from Secret Manager.
5. **Verify.**

## Step 1 – Clone the server repo

The upstream repo ships a Dockerfile that builds the `resonate-server` binary. Reuse it as-is.

```bash
git clone --depth 1 https://github.com/resonatehq/resonate.git /tmp/resonate-deploy
```

## Step 2 – Smoke-test deploy (SQLite)

Deploy first with in-memory SQLite to confirm the service starts and is reachable, then come back and switch storage. A shorter feedback loop when the pipe breaks.

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=/tmp/resonate-deploy \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --allow-unauthenticated \
  --port=8001 \
  --min-instances=1 \
  --memory=512Mi \
  --set-env-vars=RESONATE_SERVER__URL=https://PLACEHOLDER
```

Capture the service URL from the deploy output, then redeploy with `RESONATE_SERVER__URL` pointing at that URL. Cloud Run URLs are deterministic per service/project/region, so the second deploy is a one-time fix-up.

**Why `--min-instances=1`:** with cold starts, the first worker poll after a quiet period races the server coming up and surfaces as a connection failure. One warm instance avoids this.

## Step 3 – Attach Cloud SQL and switch to Postgres

Postgres connections run through the Cloud SQL Auth Proxy, which exposes a Unix socket at `/cloudsql/<connection-name>` inside the container.

**Connection string format.** sqlx (the driver used by the Resonate server) accepts Unix sockets via the `?host=...` query parameter in general, but when the connection string has an empty authority — `postgres://user:pw@/db?host=/cloudsql/...` — it surfaces a misleading "empty host" error. URL-encoding the socket path into the authority avoids that failure mode and is the form to use here:

```
postgres://PG_USER:PG_PASSWORD@%2Fcloudsql%2FPROJECT%3AREGION%3AINSTANCE/PG_DATABASE
```

Store the URL in Secret Manager so the password never lands in the Cloud Run revision config:

```bash
CONNECTION_NAME="$GCP_PROJECT_ID:$GCP_REGION:$CLOUD_SQL_INSTANCE"
PG_URL="postgres://$PG_USER:$PG_PASSWORD@$(python3 -c "import urllib.parse; print(urllib.parse.quote('/cloudsql/$CONNECTION_NAME', safe=''))")/$PG_DATABASE"
echo -n "$PG_URL" | gcloud secrets create resonate-pg-url --data-file=-
```

Redeploy with Cloud SQL attached and the secret mounted as an env var:

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=/tmp/resonate-deploy \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --allow-unauthenticated \
  --port=8001 \
  --min-instances=1 \
  --memory=512Mi \
  --add-cloudsql-instances="$CONNECTION_NAME" \
  --set-env-vars=RESONATE_SERVER__URL=https://<service-url>,RESONATE_STORAGE__TYPE=postgres \
  --set-secrets=RESONATE_STORAGE__POSTGRES__URL=resonate-pg-url:latest
```

## Step 4 – JWT authentication

Cloud Run URLs are reachable from the public internet; enable JWT auth before production traffic. The mechanics match the systemd variant — only the key delivery changes.

Generate the key pair locally (commands in [`resonate-server-deployment#jwt-authentication-setup`](../resonate-server-deployment/SKILL.md#jwt-authentication-setup)), then upload the public key to Secret Manager:

```bash
gcloud secrets create resonate-public-key --data-file=public_key.pem
```

Mount the public key as a file in the revision and point the server at it via env var:

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=/tmp/resonate-deploy \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --port=8001 \
  --min-instances=1 \
  --memory=512Mi \
  --add-cloudsql-instances="$CONNECTION_NAME" \
  --set-env-vars=RESONATE_SERVER__URL=https://<service-url>,RESONATE_STORAGE__TYPE=postgres,RESONATE_AUTH__PUBLICKEY=/etc/resonate/public_key.pem \
  --set-secrets=RESONATE_STORAGE__POSTGRES__URL=resonate-pg-url:latest,/etc/resonate/public_key.pem=resonate-public-key:latest
```

Drop `--allow-unauthenticated` once JWT is the gatekeeper. The auth check happens at the Resonate API layer, so workers still reach the service over HTTPS.

Token generation and `prefix` claim semantics are covered in [`resonate-server-deployment#jwt-authentication-setup`](../resonate-server-deployment/SKILL.md#jwt-authentication-setup).

## Step 5 – Verify

```bash
SERVER_URL=$(gcloud run services describe "$SERVICE_NAME" --region="$GCP_REGION" --project="$GCP_PROJECT_ID" --format="value(status.url)")
curl -s "$SERVER_URL/promises" | head -c 200
```

With auth enabled, expect a 401 without a token.

## Common pitfalls

- **`RESONATE_SERVER__URL` missing.** Server returns `http://localhost:8001` in task URLs. Worker logs show `fetch failed` / `connection_error`. Always set it.
- **Empty-authority Postgres URL.** `postgres://user:pw@/db?host=/cloudsql/...` surfaces as "empty host" in sqlx. Encode the socket path into the authority instead (Step 3).
- **First request after deploy 5xx.** `--min-instances=1` keeps one warm instance so worker polls don't race cold start.
- **Cloud SQL Admin API not enabled.** One-time per project: `gcloud services enable sqladmin.googleapis.com`.
- **Secret not mounted.** If `RESONATE_STORAGE__POSTGRES__URL` is blank at boot, the server falls back to SQLite silently. Confirm `--set-secrets` is present and the revision actually received the env var.

## Architecture

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │   Cloud Run    │  (--min-instances=1, JWT-gated)
              │ resonate-server│
              │    :8001       │
              └────────┬───────┘
                       │ Cloud SQL Auth Proxy
                       │ (Unix socket)
                       ▼
              ┌────────────────┐
              │  Cloud SQL     │
              │  Postgres      │
              └────────────────┘
                       ▲
              Workers  │
              ┌────────┴───────┐
              │ Cloud Function │  (RESONATE_URL = server URL)
              └────────────────┘
```

## Outputs

- A Cloud Run service serving the Resonate HTTP API, backed by Cloud SQL Postgres.
- A public HTTPS URL suitable as `RESONATE_URL` for workers and `--server` for the Resonate CLI.
- (If Step 4) JWT-authenticated API surface.

## Reference example

[`example-chess-hero-gcp-ts`](https://github.com/resonatehq-examples/example-chess-hero-gcp-ts) – end-to-end: worker on Cloud Functions Gen 2, server on Cloud Run, output streamed to the browser via the state bus pattern ([`resonate-state-bus-pattern-typescript`](../resonate-state-bus-pattern-typescript/SKILL.md)).
