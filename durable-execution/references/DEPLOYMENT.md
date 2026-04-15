# Deploying Resonate Server to Production

The Resonate server is a single-binary Go + SQLite server. No Postgres, no Redis, no Kubernetes. Deploy it like any other Go application.

---

## What You Need

| Resource | Minimum | Production |
|----------|---------|------------|
| CPU | 1 vCPU | 2 vCPU |
| Memory | 256 MB | 512 MB - 1 GB |
| Disk | 100 MB SSD | 1 GB+ SSD |
| Network | 1 port (default 8001) | + port 9090 for metrics |
| Runtime | Go 1.21+ | Go 1.21+ |

That's it. No database server, no message broker, no cluster.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RESONATE_PORT` | `8001` | Protocol endpoint |
| `RESONATE_HOST` | `0.0.0.0` | Bind address |
| `RESONATE_DB_PATH` | `./resonate.db` | SQLite file path — **use an absolute path in production** |
| `RESONATE_METRICS_PORT` | `9090` | Prometheus metrics endpoint |
| `RESONATE_TICK_INTERVAL` | `100` | Background processing interval in ms (100-500ms for production) |
| `RESONATE_TASK_LEASE_TIMEOUT` | `30000` | Task lease TTL in ms |
| `RESONATE_TASK_RETRY_TIMEOUT` | `5000` | Pending task retry interval in ms |
| `RESONATE_AUTH_PUBLIC_KEY` | _(disabled)_ | Path to RS256 PEM public key for JWT auth |
| `RESONATE_LOG_LEVEL` | `info` | `info` or `debug` |

---

## Option 1: VPS with systemd (~$5/mo)

The cheapest production deployment. A $5/mo VPS (Hetzner, DigitalOcean, Linode) handles substantial workloads.

```bash
# Clone and build
git clone https://github.com/resonatehq/resonate.git /opt/resonate
cd /opt/resonate && go build -o resonate-server .

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
Environment=RESONATE_DB_PATH=/data/resonate/resonate.db
Environment=RESONATE_PORT=8001
Environment=RESONATE_HOST=0.0.0.0
ExecStart=/opt/resonate/resonate-server
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
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=1 go build -o resonate-server .

FROM debian:bookworm-slim
COPY --from=builder /app/resonate-server /usr/local/bin/
ENV RESONATE_DB_PATH=/data/resonate.db
ENV RESONATE_HOST=0.0.0.0
EXPOSE 8001 9090
CMD ["resonate-server"]
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
#     RESONATE_DB_PATH = "/data/resonate.db"

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

# Unrestricted access (empty prefix)
jwt encode --secret @private_key.pem -A RS256 '{"prefix":""}'

# Scoped to a prefix (only promises starting with "my-app/")
jwt encode --secret @private_key.pem -A RS256 '{"prefix":"my-app/"}'

# With expiration
jwt encode --secret @private_key.pem -A RS256 --exp='+90 days' '{"prefix":""}'
```

| Payload | Access |
|---------|--------|
| `{}` | **DENIED** — no prefix claim |
| `{"prefix": ""}` | All promises (unrestricted) |
| `{"prefix": "my-app/"}` | Only `my-app/*` promises |

### 3. Configure Server

```bash
# Add to systemd service:
Environment=RESONATE_AUTH_PUBLIC_KEY=/etc/resonate/public_key.pem

# Or Docker:
docker run -v /path/to/public_key.pem:/etc/resonate/public_key.pem:ro \
  -e RESONATE_AUTH_PUBLIC_KEY=/etc/resonate/public_key.pem \
  resonate-server
```

### 4. Configure Workers

```typescript
const resonate = new Resonate({
  url: "https://resonate.example.com",
  auth: { token: process.env.RESONATE_AUTH_TOKEN! },
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
curl -sf http://localhost:9090/metrics && echo "healthy" || echo "unhealthy"
```

For container orchestrators:

```yaml
livenessProbe:
  httpGet:
    path: /metrics
    port: 9090
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Scaling

The Resonate server is a **single-process, single-writer** system. SQLite supports one writer at a time.

- **Vertical scaling:** More CPU + RAM + fast SSD. Handles substantial throughput.
- **Shard by prefix:** Multiple server instances, each with its own database, partitioned by promise ID prefix.
- **Graduate:** If you outgrow embedded SQLite, migrate to a PostgreSQL-backed Resonate server implementation.

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

Resonate is 3-100x cheaper than managed alternatives because it has zero external dependencies. No Redis, no Kafka, no Kubernetes operator.

---

## Production Checklist

- [ ] `RESONATE_DB_PATH` set to absolute path on persistent storage
- [ ] systemd/Docker configured with auto-restart
- [ ] JWT auth enabled if server is network-accessible
- [ ] Reverse proxy with SSL configured
- [ ] Metrics endpoint accessible to monitoring
- [ ] Database backup strategy in place
- [ ] Workers configured with correct server URL and auth token
- [ ] Firewall: only ports 443 (proxy) and 9090 (metrics) exposed
