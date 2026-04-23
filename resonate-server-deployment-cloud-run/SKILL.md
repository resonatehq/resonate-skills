---
name: resonate-server-deployment-cloud-run
description: Deploy the Resonate server to Google Cloud Run with Cloud SQL Postgres storage. Covers Dockerfile reuse, the Cloud SQL Auth Proxy Unix socket, mandatory --server-url, JWT authentication via Secret Manager, and known limitations.
license: Apache-2.0
---

# Resonate Server Deployment on Cloud Run

## Overview

Deploy the Resonate server (`resonatehq/resonate`, Rust) as a Google Cloud Run service backed by Cloud SQL Postgres. This is a distinct deployment target from Linux + systemd — see [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) for the VM pattern.

Companion skill: workers that talk to this server deploy via [`resonate-gcp-deployments-typescript`](../resonate-gcp-deployments-typescript/SKILL.md).

## When to use this skill

- You want the Resonate server on GCP without managing a VM.
- You want durable storage in Cloud SQL Postgres instead of on-disk SQLite.
- Your workers run on Cloud Functions, Cloud Run, or any other environment that can reach a Cloud Run service over HTTPS.

## Assumptions & Inputs

You (the agent) should obtain or be given:

- `GCP_PROJECT_ID` — GCP project
- `GCP_REGION` — region (e.g. `us-central1`)
- `CLOUD_SQL_INSTANCE` — Cloud SQL Postgres instance name (e.g. `resonate-db`)
- `PG_USER`, `PG_PASSWORD`, `PG_DATABASE` — Postgres credentials
- `SERVICE_NAME` — Cloud Run service name (e.g. `resonate-server`)

Prerequisites:

- `gcloud` authenticated to `GCP_PROJECT_ID`
- Cloud Run, Cloud SQL Admin, and Secret Manager APIs enabled
- A Cloud SQL Postgres 16+ instance, with the `PG_DATABASE` created
- Local clone of `resonatehq/resonate` (used as the build source — the repo ships its own Dockerfile)

## High-level flow

1. **Clone the server repo** — the upstream Dockerfile builds the Rust server binary.
2. **Deploy to Cloud Run** — source-based deploy, pointing at the clone.
3. **Connect to Cloud SQL** — via the Cloud SQL Auth Proxy Unix socket.
4. **Set `--server-url`** — mandatory (the default of `http://localhost:8001` silently breaks workers).
5. **(Recommended) Wire up JWT auth** via Secret Manager.
6. **(Recommended) Move the Postgres URL into Secret Manager** via a wrapper entrypoint.

## Step 1 — Clone the server repo

The upstream repo ships a Dockerfile that builds the `resonate-server` binary. Re-use it as-is rather than writing your own.

```bash
git clone --depth 1 https://github.com/resonatehq/resonate.git /tmp/resonate-deploy
```

## Step 2 — Deploy the Cloud Run service (SQLite first, to verify the pipe works)

Before adding Cloud SQL, deploy with in-memory SQLite to confirm the service starts and is reachable. This shortens the debug loop.

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=/tmp/resonate-deploy \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --allow-unauthenticated \
  --port=8001 \
  --min-instances=1 \
  --memory=512Mi \
  --args="serve,--server-url=https://PLACEHOLDER"
```

Capture the service URL from the deploy output, then redeploy with `--server-url` pointing at that URL. Cloud Run URLs are deterministic per service/project/region, so the second deploy is a one-time fix-up.

`--min-instances=1` is important: with cold starts, the first worker poll after a quiet period will race the server coming up and can look like a connection failure.

## Step 3 — Attach Cloud SQL and switch to Postgres

Postgres connections run through the Cloud SQL Auth Proxy, which exposes a Unix socket at `/cloudsql/<connection-name>` inside the container. The runtime used by Resonate (`sqlx`) only accepts Unix sockets when the socket path is URL-encoded as the authority of the connection string — not via a `?host=` query param.

**Wrong** (gives a misleading "empty host" error):

```
postgres://PG_USER:PG_PASSWORD@/PG_DATABASE?host=/cloudsql/PROJECT:REGION:INSTANCE
```

**Right** (URL-encode `/cloudsql/PROJECT:REGION:INSTANCE` into the authority):

```
postgres://PG_USER:PG_PASSWORD@%2Fcloudsql%2FPROJECT%3AREGION%3AINSTANCE/PG_DATABASE
```

Redeploy with Cloud SQL attached and the Postgres URL passed in:

```bash
CONNECTION_NAME="$GCP_PROJECT_ID:$GCP_REGION:$CLOUD_SQL_INSTANCE"
PG_URL="postgres://$PG_USER:$PG_PASSWORD@$(python3 -c "import urllib.parse; print(urllib.parse.quote('/cloudsql/$CONNECTION_NAME', safe=''))")/$PG_DATABASE"

gcloud run deploy "$SERVICE_NAME" \
  --source=/tmp/resonate-deploy \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --allow-unauthenticated \
  --port=8001 \
  --min-instances=1 \
  --memory=512Mi \
  --add-cloudsql-instances="$CONNECTION_NAME" \
  --args="serve,--server-url=https://<service-url>,--aio-store-postgres-url=$PG_URL"
```

> **Known limitation — Postgres password is visible in `--args`.** The Rust server currently has no clap `env =` fallback on `--aio-store-postgres-url`, so the URL must be passed as a flag and ends up in Cloud Run's revision config, visible to anyone with `roles/run.viewer` on the project. Mitigation below. Do not copy the raw-`--args` form into production.

## Step 4 — Move the Postgres URL into Secret Manager (recommended)

Until the upstream clap `env =` fix lands, the simplest mitigation is a tiny wrapper entrypoint that reads the URL from an env var (populated from Secret Manager) and `exec`s the server with the flag.

Create `entrypoint.sh` alongside a thin `Dockerfile.wrapper` that extends the upstream image:

```dockerfile
# Dockerfile.wrapper
FROM ghcr.io/resonatehq/resonate:latest
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh
set -e
exec resonate-server serve \
  --server-url="$RESONATE_SERVER_URL" \
  --aio-store-postgres-url="$RESONATE_PG_URL" \
  "$@"
```

Store the URL:

```bash
echo -n "$PG_URL" | gcloud secrets create resonate-pg-url --data-file=-
```

Deploy with the secret mounted as an env var:

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=. \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --allow-unauthenticated \
  --port=8001 \
  --min-instances=1 \
  --add-cloudsql-instances="$CONNECTION_NAME" \
  --set-env-vars=RESONATE_SERVER_URL=https://<service-url> \
  --set-secrets=RESONATE_PG_URL=resonate-pg-url:latest
```

The password no longer appears in revision config, only in Secret Manager (where IAM controls access).

## Step 5 — JWT authentication

Cloud Run URLs are effectively public, so if you're going anywhere near production, enable JWT auth. The mechanics are identical to [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) — the only difference is key delivery.

Generate the key pair locally:

```bash
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

Upload the public key to Secret Manager:

```bash
gcloud secrets create resonate-public-key --data-file=public_key.pem
```

Mount it as a file in the Cloud Run revision and point the server at it:

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source=. \
  --region="$GCP_REGION" \
  --project="$GCP_PROJECT_ID" \
  --port=8001 \
  --min-instances=1 \
  --add-cloudsql-instances="$CONNECTION_NAME" \
  --set-env-vars=RESONATE_SERVER_URL=https://<service-url> \
  --set-secrets=RESONATE_PG_URL=resonate-pg-url:latest,/etc/resonate/public_key.pem=resonate-public-key:latest
```

Extend `entrypoint.sh` to pass the key path:

```bash
exec resonate-server serve \
  --server-url="$RESONATE_SERVER_URL" \
  --aio-store-postgres-url="$RESONATE_PG_URL" \
  --api-auth-public-key=/etc/resonate/public_key.pem \
  "$@"
```

Token generation and `prefix` claim semantics are covered in [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md#jwt-authentication-setup). Drop `--allow-unauthenticated` from the Cloud Run deploy flags once JWT is the gatekeeper — workers still need HTTPS reachability, which they get; the auth check happens at the Resonate API layer.

## Step 6 — Verify

```bash
SERVER_URL=$(gcloud run services describe "$SERVICE_NAME" --region="$GCP_REGION" --project="$GCP_PROJECT_ID" --format="value(status.url)")
curl -s "$SERVER_URL/promises" | head -c 200
```

With auth enabled, expect a 401 without a token.

## Common pitfalls

- **`--server-url` missing.** Server returns `http://localhost:8001` in task URLs. Worker logs show `fetch failed` / `connection_error`. Always set it.
- **`--args` quoting.** Comma-separated, no spaces. Cloud Run `--args` replaces the image's `CMD`; the upstream Dockerfile's `ENTRYPOINT` is still `resonate-server`, so the first arg is `serve`.
- **Cloud SQL socket URL format.** See Step 3 — `?host=` fails silently.
- **First request after deploy 5xx.** `--min-instances=1` keeps one warm instance so worker polls don't race cold start.
- **Forgetting the Cloud SQL Admin API.** Enable it once per project: `gcloud services enable sqladmin.googleapis.com`.

## Architecture

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │   Cloud Run    │  (--min-instances=1, --allow-unauthenticated + JWT)
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
- (If Step 5) JWT-authenticated API surface.
