---
name: resonate-token-authentication-typescript
description: Secure Resonate applications with JWT-based authentication and prefix-based authorization. Use for multi-tenant systems, role-based access control, and isolating workers or services from accessing each other's promises.
license: Apache-2.0
---

# Resonate Token Authentication & Authorization (TypeScript)

## Overview

Resonate supports JWT-based authentication and prefix-based authorization to secure access to the Resonate server and control which promises clients can access. This enables multi-tenant systems, role-based access control, and isolation between different workers or services.

**Core principle:** Generate JWT tokens signed with a private key, configure the Resonate server with the public key, and clients present tokens to authenticate. Optionally use the `prefix` claim for fine-grained authorization.

## Mental Model

```
Without Auth:
  Client → Resonate Server → All Promises Accessible

With Token Auth:
  Client + Valid JWT → Resonate Server → All Promises Accessible
  Client + Invalid JWT → Resonate Server → REJECTED (401 Unauthorized)

With Prefix Auth (Multi-Tenant):
  Client + JWT(prefix="tenant-1") → Access only "tenant-1:*" promises
  Client + JWT(prefix="tenant-2") → Access only "tenant-2:*" promises
  Client + JWT(prefix="worker-a") → Access only "worker-a:*" promises
```

## Setup: Generate Keys and Tokens

### 1. Generate RSA Key Pair

```bash
# Generate private key (keep this secret!)
openssl genrsa -out private_key.pem 2048

# Extract public key (share with Resonate server)
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

**Security:**
- Store `private_key.pem` securely (never commit to git, use secrets manager)
- `public_key.pem` can be deployed with server configuration

### 2. Install JWT CLI Tool

```bash
# macOS
brew install mike-engel/jwt-cli/jwt-cli

# Other platforms
# https://github.com/mike-engel/jwt-cli#installation
```

### 3. Generate JWT Tokens

**Basic authentication token (no prefix):**
```bash
jwt encode --secret @private_key.pem -A RS256 '{}'
```

**Token with prefix claim (for authorization):**
```bash
jwt encode --secret @private_key.pem -A RS256 '{"prefix":"tenant-1"}'
```

**Store token in environment variable:**
```bash
export MY_TOKEN=$(jwt encode --secret @private_key.pem -A RS256 '{"prefix":"worker-1"}')
```

## Server Configuration

### Start Resonate Server with Authentication

```bash
# Enable JWT authentication
resonate dev --auth-publickey public_key.pem
```

**What this does:**
- Enables JWT authentication on all HTTP API endpoints
- Validates JWT signatures using the provided public key
- Rejects requests with invalid or missing tokens (401 Unauthorized)
- Enforces prefix-based authorization if `prefix` claim is present

### Production Deployment

```bash
# Using Docker
docker run -v $(pwd)/public_key.pem:/keys/public_key.pem \
  resonatehq/resonate \
  serve --auth-publickey /keys/public_key.pem

# Using systemd service (see `resonate-server-deployment`)
resonate serve \
  --auth-publickey /etc/resonate/public_key.pem \
  --storage-type postgres \
  --storage-postgres-url postgres://user:pass@db.example.com/resonate
```

For fully worked deploys of the server with auth wired in, see [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) (Linux/systemd) or [`resonate-server-deployment-cloud-run`](../resonate-server-deployment-cloud-run/SKILL.md) (GCP Cloud Run + Cloud SQL).

## Pattern 1: Basic Token Authentication

**Use when:** You want to restrict access to trusted clients only, but all authenticated clients have equal access.

```typescript
import { Resonate, Context } from "@resonatehq/sdk";

function* helloAuth(ctx: Context, greeting: string) {
  const result = yield* ctx.run((ctx: Context) => {
    return `${greeting} world!`;
  });

  return result;
}

// Authenticated client
const resonate = new Resonate({
  url: "http://localhost:8001",
  token: process.env.MY_TOKEN  // JWT token from environment
});

// Register and run workflows
const workflow = resonate.register("workflow", helloAuth);
const result = await workflow.run("workflow.id", "hello");

console.log(result);  // "hello world!"

resonate.stop();
```

**Without token:**
```typescript
// ❌ This will fail with "ResonateError: The request is unauthorized"
const resonateNoAuth = new Resonate({
  url: "http://localhost:8001"
  // No token provided
});

const workflow = resonateNoAuth.register("workflow", helloAuth);
await workflow.run("workflow.id", "hello");  // THROWS 401 ERROR
```

## Pattern 2: Prefix-Based Authorization (Multi-Tenant)

**Use when:** You need to isolate promises between tenants, workers, or services.

### Server-Side Prefix Enforcement

When a client connects with a token containing a `prefix` claim, the Resonate server enforces that the client can only:
- Create promises with IDs starting with that prefix
- Access promises with IDs starting with that prefix

**Generate tenant-specific tokens:**
```bash
# Token for Tenant 1
export TENANT1_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{"prefix":"tenant-1"}')

# Token for Tenant 2
export TENANT2_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{"prefix":"tenant-2"}')

# Token for Worker A
export WORKER_A_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{"prefix":"worker-a"}')
```

### Client with Automatic Prefix

```typescript
// Tenant 1 client
const tenant1 = new Resonate({
  url: "http://localhost:8001",
  token: process.env.TENANT1_TOKEN,
  prefix: "tenant-1"  // SDK automatically prefixes all promise IDs
});

const workflow1 = tenant1.register("workflow", processOrder);

// Creates promise with ID "tenant-1:order-123"
await workflow1.run("order-123", orderData);

// Tenant 2 client (isolated from Tenant 1)
const tenant2 = new Resonate({
  url: "http://localhost:8001",
  token: process.env.TENANT2_TOKEN,
  prefix: "tenant-2"
});

const workflow2 = tenant2.register("workflow", processOrder);

// Creates promise with ID "tenant-2:order-123"
await workflow2.run("order-123", orderData);

// Tenant 1 cannot access Tenant 2's promises (and vice versa)
```

### Client with Manual Prefix

```typescript
const resonate = new Resonate({
  url: "http://localhost:8001",
  token: process.env.TENANT1_TOKEN
  // No prefix set in SDK
});

const workflow = resonate.register("workflow", processOrder);

// Manually prefix the promise ID
await workflow.run("tenant-1:order-123", orderData);
```

**Why manual prefixing?**
- More control over promise ID structure
- Can use different prefixes for different workflows
- Useful for migration or complex ID schemes

## Pattern 3: Multi-Worker Isolation

**Use when:** Multiple worker groups should not interfere with each other's promises.

```typescript
// Worker Group A
const workerA = new Resonate({
  url: "http://localhost:8001",
  token: process.env.WORKER_A_TOKEN,
  prefix: "worker-a",
  group: "workers-a"
});

workerA.register("processTask", processTask);

// Worker Group B (isolated from A)
const workerB = new Resonate({
  url: "http://localhost:8001",
  token: process.env.WORKER_B_TOKEN,
  prefix: "worker-b",
  group: "workers-b"
});

workerB.register("processTask", processTask);

// Worker A can only claim tasks prefixed with "worker-a:"
// Worker B can only claim tasks prefixed with "worker-b:"
```

## Pattern 4: Environment Variable Configuration

**Use when:** You want to avoid hardcoding credentials in source code.

```typescript
// Set environment variables
// RESONATE_TOKEN=<jwt-token>
// RESONATE_PREFIX=tenant-1

// SDK automatically reads from environment
const resonate = new Resonate({
  url: "http://localhost:8001"
  // Token and prefix read from RESONATE_TOKEN and RESONATE_PREFIX
});

const workflow = resonate.register("workflow", myWorkflow);
await workflow.run("order-123", data);  // Creates "tenant-1:order-123"
```

**Environment variables:**
- `RESONATE_TOKEN`: JWT token
- `RESONATE_PREFIX`: Promise ID prefix

## Pattern 5: Role-Based Access Control (RBAC)

**Use when:** Different roles need different levels of access.

```bash
# Admin token (access all promises)
export ADMIN_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{}')

# Service token (access only service-specific promises)
export SERVICE_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{"prefix":"service-api"}')

# User token (access only user-specific promises)
export USER_TOKEN=$(jwt encode -S @private_key.pem -A RS256 '{"prefix":"user-${userId}"}')
```

**Client code:**
```typescript
// Admin client (no prefix restriction)
const admin = new Resonate({
  url: "http://localhost:8001",
  token: process.env.ADMIN_TOKEN
});

// Can access any promise ID
await admin.promises.get("tenant-1:order-123");
await admin.promises.get("tenant-2:order-456");

// Service client (restricted to service-api:* promises)
const service = new Resonate({
  url: "http://localhost:8001",
  token: process.env.SERVICE_TOKEN,
  prefix: "service-api"
});

// Can only access "service-api:*" promises
await service.promises.get("request-123");  // OK (becomes "service-api:request-123")
```

## Token Claims and Validation

**Required claims:**
- None (empty JWT `{}` is valid for basic authentication)

**Optional claims:**
- `prefix` (string): Restricts promise access to IDs starting with this prefix
- `exp` (number): Token expiration timestamp (Unix epoch)
- `iat` (number): Token issued-at timestamp

**Token validation:**
```bash
# Generate token with expiration (1 hour)
jwt encode -S @private_key.pem -A RS256 \
  --exp='+1 hour' \
  '{"prefix":"tenant-1"}'

# Decode and verify token
jwt decode -S @public_key.pem $MY_TOKEN
```

## Security Best Practices

### 1. Protect Private Keys

```bash
# ❌ WRONG - Committing keys to git
git add private_key.pem  # NEVER DO THIS

# ✅ CORRECT - Use secrets manager
aws secretsmanager create-secret \
  --name resonate-jwt-private-key \
  --secret-string file://private_key.pem

# ✅ CORRECT - Add to .gitignore
echo "*.pem" >> .gitignore
echo "*.key" >> .gitignore
```

### 2. Rotate Keys Periodically

```bash
# Generate new key pair
openssl genrsa -out private_key_v2.pem 2048
openssl rsa -in private_key_v2.pem -pubout -out public_key_v2.pem

# Update server configuration
resonate serve --auth-publickey public_key_v2.pem

# Issue new tokens with new private key
jwt encode -S @private_key_v2.pem -A RS256 '{"prefix":"tenant-1"}'
```

### 3. Use Token Expiration

```bash
# Generate short-lived tokens (1 hour)
jwt encode -S @private_key.pem -A RS256 --exp='+1 hour' '{}'

# Generate long-lived tokens (30 days)
jwt encode -S @private_key.pem -A RS256 --exp='+30 days' '{}'
```

### 4. Least Privilege Prefixes

```typescript
// ✅ GOOD - Narrow prefix
const client = new Resonate({
  token: token,
  prefix: "user-123"  // Can only access user-123's promises
});

// ❌ BAD - Wide prefix
const client = new Resonate({
  token: token,
  prefix: "user"  // Can access all users' promises (user-*, user-123, etc.)
});
```

## Production Recommendations

### 1. Use External Identity Provider

For production systems, integrate with identity providers like:
- **Keycloak**: Open-source identity and access management
- **Auth0**: Managed authentication service
- **Okta**: Enterprise identity platform
- **AWS Cognito**: AWS-managed identity service

### 2. Token Refresh Strategy

```typescript
// Implement token refresh before expiration
let token = await getInitialToken();
let resonate = new Resonate({ url: serverUrl, token });

// Refresh token periodically
setInterval(async () => {
  token = await refreshToken(token);
  resonate = new Resonate({ url: serverUrl, token });
}, 30 * 60 * 1000);  // Refresh every 30 minutes
```

### 3. Audit Logging

```typescript
// Log authentication events
resonate.on("error", (error) => {
  if (error.message.includes("unauthorized")) {
    logger.warn("Authentication failed", {
      timestamp: Date.now(),
      error: error.message
    });
  }
});
```

## Common Pitfalls

### 1. Forgetting to Configure Server

```bash
# ❌ WRONG - Server not configured for auth
resonate dev  # No --auth-publickey flag

# Client with token connects but auth is not enforced
```

```bash
# ✅ CORRECT - Server configured for auth
resonate dev --auth-publickey public_key.pem
```

### 2. Mismatched Prefix in Token and SDK

```typescript
// Token has prefix="tenant-1"
const token = generateToken({ prefix: "tenant-1" });

// ❌ WRONG - SDK prefix doesn't match token
const resonate = new Resonate({
  token: token,
  prefix: "tenant-2"  // Server will reject requests
});

// ✅ CORRECT - SDK prefix matches token
const resonate = new Resonate({
  token: token,
  prefix: "tenant-1"  // Matches token claim
});
```

### 3. Exposing Tokens in Logs

```typescript
// ❌ WRONG - Token exposed in logs
console.log(`Token: ${process.env.MY_TOKEN}`);

// ✅ CORRECT - Redact tokens
console.log(`Token: ${process.env.MY_TOKEN?.slice(0, 10)}...`);
```

## Decision Tree

**Use token authentication when:**
- Restricting access to trusted clients only
- Deploying to production environments
- Exposing Resonate server over public networks

**Use prefix-based authorization when:**
- Multi-tenant systems (isolate tenant data)
- Role-based access control
- Isolating worker groups
- Implementing data access policies

**Don't use authentication when:**
- Local development (use `resonate dev` without flags)
- Internal private networks (though still recommended)
- Single-tenant systems with trusted clients

## Summary

Token authentication and prefix-based authorization enable:
- Secure access control with JWT tokens
- Multi-tenant isolation via prefix claims
- Role-based access control
- Fine-grained promise access policies
- Integration with external identity providers

**Core recipe:** Generate RSA key pair → Configure server with public key → Generate JWT tokens with prefix claims → Clients authenticate with tokens → Server enforces prefix-based access control
