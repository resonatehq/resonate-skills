---
name: resonate-server-deployment
description: Deploy and configure the Resonate server on Linux systems with systemd. Covers installation, public URL configuration, JWT authentication, and troubleshooting.
---

# Resonate Server Deployment

## Overview

Deploy the Resonate server on Linux systems using systemd for process management. This skill covers installation, configuration of public URLs, JWT authentication setup, and common troubleshooting scenarios.

## Prerequisites

- Linux system with systemd
- Root/sudo access
- curl installed
- (Optional) openssl for generating JWT keys
- (Optional) jwt-cli for generating tokens

## Quick Start

### Basic Deployment (No Auth)

```bash
# Download and run
curl -L -o deploy-resonate.sh https://example.com/deploy-resonate.sh
chmod +x deploy-resonate.sh
sudo ./deploy-resonate.sh
```

### Deployment with Public URL and JWT Auth

```bash
# Generate RSA key pair first
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem

# Deploy with configuration
sudo RESONATE_API_URL=https://resonate.example.com \
     RESONATE_AUTH_ENABLED=true \
     RESONATE_PUBLIC_KEY=/path/to/public_key.pem \
     ./deploy-resonate.sh
```

## Configuration Options

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `RESONATE_VERSION` | v0.8.2 | Server version to install |
| `RESONATE_PORT` | 8001 | HTTP API port |
| `RESONATE_API_URL` | (none) | Public URL for the server (e.g., `https://resonate.example.com`) |
| `RESONATE_AUTH_ENABLED` | false | Enable JWT authentication |
| `RESONATE_PUBLIC_KEY` | (none) | Path to JWT public key file (required if auth enabled) |

## Server Flags Reference

The Resonate server binary accepts these flags:

```bash
resonate serve [flags]

Flags:
  --api-url string              Public URL for the server
  --api-auth-public-key string  Path to JWT public key for authentication
  --api-http-port int           HTTP API port (default 8001)
  --aio-store-sqlite-path       SQLite database path
  --aio-store-postgres-*        PostgreSQL connection options
```

## Architecture

```
                    Internet
                       │
                       ▼
              ┌────────────────┐
              │  Nginx/Caddy   │  (SSL termination)
              │  Port 443      │
              └────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │ Resonate Server│  (systemd service)
              │ Port 8001      │
              └────────────────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
       ┌──────────┐      ┌──────────┐
       │ Worker 1 │      │ Worker 2 │
       └──────────┘      └──────────┘
```

## Systemd Service Configuration

### Basic Service File

```ini
# /etc/systemd/system/resonate.service
[Unit]
Description=Resonate Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/lib/resonate
ExecStart=/usr/local/bin/resonate serve
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=resonate
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Service File with Public URL and Auth

```ini
# /etc/systemd/system/resonate.service
[Unit]
Description=Resonate Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/lib/resonate
ExecStart=/usr/local/bin/resonate serve \
  --api-url https://resonate.example.com \
  --api-auth-public-key /etc/resonate/public_key.pem
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=resonate
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## JWT Authentication Setup

### 1. Generate RSA Key Pair

```bash
# Generate private key (keep secret!)
openssl genrsa -out private_key.pem 2048

# Extract public key (deploy with server)
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

### 2. Install JWT CLI (for generating tokens)

```bash
# macOS
brew install mike-engel/jwt-cli/jwt-cli

# Linux (download binary)
curl -L -o jwt https://github.com/mike-engel/jwt-cli/releases/latest/download/jwt-linux
chmod +x jwt
sudo mv jwt /usr/local/bin/
```

### 3. Generate Client Tokens

```bash
# Basic token (no prefix restriction)
jwt encode --secret @private_key.pem -A RS256 '{}'

# Token with prefix (restricts access to promises with this prefix)
jwt encode --secret @private_key.pem -A RS256 '{"prefix":"my-app"}'

# Token with expiration
jwt encode --secret @private_key.pem -A RS256 --exp='+30 days' '{}'
```

### 4. Configure Server

```bash
# Copy public key to server
scp public_key.pem root@server:/etc/resonate/

# Update service
sudo RESONATE_AUTH_ENABLED=true \
     RESONATE_PUBLIC_KEY=/etc/resonate/public_key.pem \
     ./deploy-resonate.sh --update-service
```

### 5. Configure Clients

```typescript
// SDK client
const resonate = new Resonate({
  url: "https://resonate.example.com",
  token: process.env.RESONATE_AUTH_TOKEN  // JWT token
});

// Direct HTTP calls
const headers = {
  "Content-Type": "application/json",
  "Authorization": `Bearer ${token}`
};
```

## Why API URL Matters

The `--api-url` flag tells the Resonate server its public address. This is critical when:

1. **Workers poll for tasks**: The server returns URLs that workers use for callbacks
2. **Clients connect from different networks**: The server advertises its address
3. **Behind a reverse proxy**: Server needs to know the external URL, not localhost

**Without `--api-url`:**
- Server returns `http://localhost:8001` in responses
- External workers can't reach callback URLs
- Polling may fail with connection errors

**With `--api-url`:**
- Server returns `https://resonate.example.com` in responses
- Workers and clients use the correct public URL

## Directory Structure

```
/usr/local/bin/
└── resonate              # Server binary

/etc/resonate/
├── public_key.pem        # JWT public key (if auth enabled)
└── resonate.yml          # Optional config file

/var/lib/resonate/
└── resonate.db           # SQLite database (default)

/etc/systemd/system/
└── resonate.service      # Systemd service file
```

## Common Operations

### View Logs

```bash
# Follow logs
journalctl -u resonate -f

# Last 100 lines
journalctl -u resonate -n 100

# Logs since boot
journalctl -u resonate -b

# Logs from specific time
journalctl -u resonate --since "2024-01-28 10:00:00"
```

### Service Management

```bash
# Status
systemctl status resonate

# Start/Stop/Restart
systemctl start resonate
systemctl stop resonate
systemctl restart resonate

# Enable/Disable on boot
systemctl enable resonate
systemctl disable resonate

# Reload service file after changes
systemctl daemon-reload
```

### Check Configuration

```bash
# View current service configuration
systemctl cat resonate

# Check what command is running
ps aux | grep resonate

# Test server is responding
curl http://localhost:8001/promises
```

## Troubleshooting

### 401 Unauthorized Errors

**Symptoms:**
- Clients get 401 errors
- "missing authorization header" in logs
- Workers can't poll for tasks

**Causes & Fixes:**

1. **Auth enabled on server but client has no token**
   ```typescript
   // Add token to client
   const resonate = new Resonate({
     url: "https://resonate.example.com",
     token: process.env.RESONATE_AUTH_TOKEN
   });
   ```

2. **Token is expired**
   ```bash
   # Generate new token
   jwt encode --secret @private_key.pem -A RS256 --exp='+30 days' '{}'
   ```

3. **Wrong public key on server**
   ```bash
   # Verify key matches
   openssl rsa -in private_key.pem -pubout | diff - public_key.pem
   ```

4. **Token signed with wrong private key**
   ```bash
   # Decode and verify token
   jwt decode $TOKEN
   ```

### Server Not Responding

**Symptoms:**
- `curl http://localhost:8001` hangs or refuses connection
- Service shows as active but port not open

**Causes & Fixes:**

1. **Service crashed on startup**
   ```bash
   journalctl -u resonate -n 50
   # Look for error messages
   ```

2. **Port already in use**
   ```bash
   netstat -tlnp | grep 8001
   # Kill conflicting process or change port
   ```

3. **Firewall blocking port**
   ```bash
   # Check UFW (Ubuntu)
   ufw status
   # Allow internal access only (recommended)
   # Don't expose 8001 directly - use reverse proxy
   ```

### Workers Can't Connect

**Symptoms:**
- Workers start but never receive tasks
- "connection refused" errors

**Causes & Fixes:**

1. **Wrong RESONATE_URL in worker**
   ```bash
   # Should be public URL if server is remote
   RESONATE_URL=https://resonate.example.com
   # NOT http://localhost:8001 (unless worker is on same machine)
   ```

2. **Missing --api-url on server**
   ```bash
   # Server doesn't know its public URL
   # Update service with --api-url flag
   ```

3. **SSL/TLS issues**
   ```bash
   # Test with curl
   curl -v https://resonate.example.com/promises
   # Check certificate is valid
   ```

### Database Issues

**Symptoms:**
- "database is locked" errors
- Data not persisting across restarts

**Causes & Fixes:**

1. **Multiple processes accessing SQLite**
   ```bash
   # Only one server should access the DB
   ps aux | grep resonate
   # Kill duplicates
   ```

2. **Wrong working directory**
   ```bash
   # Check where DB is being created
   find / -name "resonate.db" 2>/dev/null
   # Ensure WorkingDirectory is set in service file
   ```

3. **Disk full**
   ```bash
   df -h
   # Clean up or expand disk
   ```

## Production Checklist

- [ ] Server binary installed and versioned
- [ ] Systemd service file created and enabled
- [ ] `--api-url` set to public URL
- [ ] JWT authentication enabled (if needed)
- [ ] Public key deployed to `/etc/resonate/`
- [ ] Reverse proxy configured (Nginx/Caddy)
- [ ] SSL certificate installed
- [ ] Firewall configured (only 443 exposed)
- [ ] Logs rotating (journald handles this)
- [ ] Monitoring configured (port 9090 metrics)
- [ ] Backup strategy for database

## Reverse Proxy Configuration

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
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # For long-polling
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
}
```

### Caddy

```caddyfile
resonate.example.com {
    reverse_proxy localhost:8001
}
```

## Summary

Key deployment steps:
1. Install binary from GitHub releases
2. Create systemd service with appropriate flags
3. Configure `--api-url` for public accessibility
4. Enable JWT auth with `--api-auth-public-key` if needed
5. Set up reverse proxy with SSL
6. Configure clients with correct URL and token

**Critical flags:**
- `--api-url`: Server's public URL (required for external access)
- `--api-auth-public-key`: JWT public key path (required for auth)
