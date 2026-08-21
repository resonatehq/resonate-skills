# Deploying Resonate Server to Production

The open-source Resonate server (`resonatehq/resonate`) is a single-binary Rust + SQLite server. No Postgres, no Redis, no Kubernetes. Deploy it like any other small static binary.

> **Language note.** Code examples here are shown in **TypeScript**. The durable-execution concepts are identical across all four Resonate SDKs — only the syntax differs (Python uses bare `yield`, Rust uses `async fn` + `.await`, Go uses ordinary funcs + `Future.Await`). For concrete, idiomatic syntax in your language, see the per-SDK skills: `resonate-basic-durable-world-usage-{typescript,python,rust,go}` (and the matching pattern/debugging skills).

---

## What You Need

| Resource | Minimum | Production |
|----------|---------|------------|
| CPU | 1 vCPU | 2 vCPU |
| Memory | 256 MB | 512 MB - 1 GB |
| Disk | 100 MB SSD | 1 GB+ SSD |
| Network | 1 port (default 8001) | + port 9090 for metrics |
| Runtime | None (prebuilt binary) | None (prebuilt binary); Rust 1.94+ only if building from source |

That's it. No database server, no message broker, no cluster.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RESONATE_SERVER__HOST` | `localhost` | HTTP server host |
| `RESONATE_SERVER__PORT` | `8001` | HTTP API port |
| `RESONATE_SERVER__BIND` | `0.0.0.0` | Bind address |
| `RESONATE_SERVER__URL` | `http://{host}:{port}` | Externally-reachable URL — **required for distributed deployments where workers call back** |
| `RESONATE_LEVEL` | `info` | Log level: `debug`/`info`/`warn`/`error` |
| `RESONATE_STORAGE__TYPE` | `sqlite` | `sqlite` / `postgres` / `mysql` |
| `RESONATE_STORAGE__SQLITE__PATH` | `resonate.db` | SQLite file path — **use an absolute path in production** |
| `RESONATE_STORAGE__POSTGRES__URL` | — | Postgres connection string |
| `RESONATE_AUTH__PUBLICKEY` | _(disabled)_ | Path to RS256 PEM public key for JWT auth |
| `RESONATE_TASKS__LEASE_TIMEOUT` | `15000` | Task lease timeout (ms) |
| `RESONATE_TASKS__RETRY_TIMEOUT` | `30000` | Suspend/wake retry interval (ms); lower to `500` for chained/recursive workflows |
| `RESONATE_OBSERVABILITY__METRICS_PORT` | `9090` | Prometheus metrics port (`0` disables) |

Run `resonate serve --help` for the full, authoritative flag list.

---

## Option 1: VPS with systemd (~$5/mo)

The lowest-cost production deployment. A $5/mo VPS (Hetzner, DigitalOcean, Linode) handles substantial workloads.

```bash
# Download the prebuilt binary from GitHub releases (recommended):
# https://github.com/resonatehq/resonate/releases
# Example for Linux x86_64 (release assets are tarballs, not bare binaries):
curl -L https://github.com/resonatehq/resonate/releases/latest/download/resonate_linux_x86_64.tar.gz \
  | tar -xzO > /usr/local/bin/resonate-server
chmod +x /usr/local/bin/resonate-server

# Alternatively, build from source (requires Rust toolchain):
# git clone https://github.com/resonatehq/resonate.git /opt/resonate
# cd /opt/resonate && cargo build --release
# cp target/release/resonate /usr/local/bin/resonate-server

# Create data directory
mkdir -p /data/resonate
```

### systemd Service

```ini
# /etc/systemd/system/resonate.service
[Unit]
Description=Resonate Server
After=network.target

[Service]
Type=simple
User=resonate
WorkingDirectory=/opt/resonate
Environment=RESONATE_STORAGE__SQLITE__PATH=/data/resonate/resonate.db
Environment=RESONATE_SERVER__PORT=8001
Environment=RESONATE_SERVER__BIND=0.0.0.0
ExecStart=/usr/local/bin/resonate-server serve
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable resonate
sudo systemctl start resonate

# Check status
sudo systemctl status resonate
journalctl -u resonate -f
```

---

## Option 2: Docker

```dockerfile
FROM rust:1.94-slim-bookworm AS builder
RUN apt-get update && apt-get install -y --no-install-recommends pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/resonate /usr/local/bin/resonate-server
EXPOSE 8001 9090
ENTRYPOINT ["resonate-server"]
CMD ["serve"]
```

```bash
docker build -t resonate-server .
docker run -d \
  -p 8001:8001 \
  -p 9090:9090 \
  -v resonate-data:/data \
  --name resonate \
  resonate-server
```

**Important:** Mount a persistent volume at `/data`. Without it, SQLite data is lost when the container restarts.

---

## Option 3: Fly.io

```bash
# In your resonate server directory:
fly launch --no-deploy
fly volumes create resonate_data --size 1 --region ord

# Edit fly.toml:
#   [mounts]
#     source = "resonate_data"
#     destination = "/data"
#   [env]
#     RESONATE_STORAGE__SQLITE__PATH = "/data/resonate.db"

fly deploy
```

Cost: ~$3-5/mo for a shared CPU VM + $0.15/GB/mo for the volume.

---

## JWT Authentication

Enable JWT auth to restrict access to the server.

### 1. Generate Keys

```bash
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

### 2. Generate Tokens

```bash
# Install jwt-cli: brew install mike-engel/jwt-cli/jwt-cli

# The server requires an `exp` claim on every token — always pass --exp,
# or the token fails authentication before the prefix is ever checked.

# Unrestricted access (empty prefix)
jwt encode --secret @private_key.pem -A RS256 --exp='+90 days' '{"prefix":""}'

# Scoped to a prefix (only promises starting with "my-app/")
jwt encode --secret @private_key.pem -A RS256 --exp='+90 days' '{"prefix":"my-app/"}'
```

| Payload | Access |
|---------|--------|
| `{}` | **DENIED** — no prefix claim |
| `{"prefix": ""}` | All promises (unrestricted) |
| `{"prefix": "my-app/"}` | Only `my-app/*` promises |

### 3. Configure Server

```bash
# Add to systemd service:
Environment=RESONATE_AUTH__PUBLICKEY=/etc/resonate/public_key.pem

# Or Docker:
docker run -v /path/to/public_key.pem:/etc/resonate/public_key.pem:ro \
  -e RESONATE_AUTH__PUBLICKEY=/etc/resonate/public_key.pem \
  resonate-server
```

### 4. Configure Workers

```typescript
const resonate = new Resonate({
  url: "https://resonate.example.com",
  token: process.env.RESONATE_TOKEN!,
});
```

---

## Reverse Proxy (SSL)

Don't expose the Resonate server directly. Put it behind a reverse proxy with SSL.

### Caddy (easiest)

```caddyfile
resonate.example.com {
    reverse_proxy localhost:8001
}
```

Caddy auto-provisions SSL certificates from Let's Encrypt.

### Nginx

```nginx
server {
    listen 443 ssl;
    server_name resonate.example.com;
    ssl_certificate /etc/letsencrypt/live/resonate.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/resonate.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 120s;  # For SSE long-poll
    }
}
```

---

## Health Check

```bash
curl -sf http://localhost:8001/health && echo "healthy" || echo "unhealthy"
```

The health route is `/health` on the API port (`8001`), not `/healthz`. (Port `9090` serves Prometheus `/metrics`, not health.)

For container orchestrators:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8001
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Scaling

The Resonate server is a **single-process, single-writer** system. SQLite supports one writer at a time.

- **Vertical scaling:** More CPU + RAM + fast SSD. Handles substantial throughput.
- **Shard by prefix:** Multiple server instances, each with its own database, partitioned by promise ID prefix.
- **Graduate:** If you outgrow embedded SQLite, point the same binary at Postgres with `--storage-type postgres` (or `mysql`) — no separate implementation needed.

For most workloads, a single $5-20/mo VPS with an SSD is plenty.

---

## Backup

SQLite is a single file. Back it up with a copy:

```bash
# Hot backup (while server is running — SQLite WAL handles this safely)
sqlite3 /data/resonate/resonate.db ".backup /data/backups/resonate-$(date +%Y%m%d).db"
```

For streaming backups to S3, consider [Litestream](https://litestream.io/).

---

## Cost Comparison

| Approach | Monthly Cost (1M tasks/day) | Infrastructure |
|----------|----------------------------|----------------|
| **Resonate** | $5 (VPS) to $170 (dedicated) | Single binary + SQLite |
| **Temporal Cloud** | ~$520+ | Managed cluster |
| **AWS Step Functions** | ~$250 | AWS-locked |
| **Baked-in (your DB)** | $0 incremental | Your existing database |

Resonate costs 3-100x less than managed alternatives because it has zero external dependencies. No Redis, no Kafka, no Kubernetes operator.

---

## Production Checklist

- [ ] `RESONATE_STORAGE__SQLITE__PATH` set to absolute path on persistent storage
- [ ] systemd/Docker configured with auto-restart
- [ ] JWT auth enabled if server is network-accessible
- [ ] Reverse proxy with SSL configured
- [ ] Metrics endpoint accessible to monitoring
- [ ] Database backup strategy in place
- [ ] Workers configured with correct server URL and auth token
- [ ] Firewall: only ports 443 (proxy) and 9090 (metrics) exposed
