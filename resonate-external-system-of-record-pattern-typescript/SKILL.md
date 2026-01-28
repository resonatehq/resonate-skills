---
name: resonate-external-system-of-record-pattern-typescript
description: Maintain consistency across multiple systems without distributed transactions using the "Write Last, Read First" principle. Use when integrating with external systems of record that can't participate in transactions.
---

# Resonate External System of Record Pattern (TypeScript)

## Overview

The External System of Record pattern enables consistent state across multiple systems without distributed transactions. By designating one system as the "system of record" (source of truth) and treating others as "systems of reference" (cached/staged data), you can maintain consistency through careful ordering of writes and reads.

**Core principle:** Write to the system of record last. Read from the system of record first. Use Resonate's durable execution to guarantee eventual completion and automatic recovery.

## Mental Model

```
WITHOUT TRANSACTIONS:

  System of Reference         System of Record
  (Staging/Cache)            (Source of Truth)
         │                          │
         │   Write Last →           │
         │   Read First ←           │
         │                          │
   [May be stale]           [Determines existence]

Resonate guarantees eventual completion:
- Crash after reference write? Resumes, completes record write
- Crash after record write? Resumes, skips completed steps
```

## System Roles

### System of Record
- **The champion:** If it exists here, it exists in reality
- **Determines existence:** The authoritative source of truth
- **Examples:** TigerBeetle ledger, payment processor, blockchain

### System of Reference
- **The supporter:** Staging area or cache
- **Does NOT determine existence:** A record here without the system of record means nothing
- **Examples:** PostgreSQL, Redis, application database

## Core Pattern: Write Last, Read First

### Writing (Creating Records)

```typescript
function* createEntity(ctx: Context, id: string, data: any) {
  // 1. Write to system of REFERENCE first (staging)
  yield* ctx.run(writeToReference, id, data);

  // 2. Write to system of RECORD last (commits existence)
  yield* ctx.run(writeToRecord, id, data);

  // Only NOW does the entity truly exist
  return { id, status: "created" };
}
```

**Why this order?**
- Reference write is "safe" - doesn't commit anything
- Only when record write succeeds does the entity spring into existence
- If crash happens between writes, Resonate will resume and complete the record write

### Reading (Checking Existence)

```typescript
function* getEntity(ctx: Context, id: string) {
  // 1. Read from system of RECORD first (check existence)
  const exists = yield* ctx.run(checkRecord, id);

  if (!exists) {
    return null;  // Doesn't exist, regardless of reference
  }

  // 2. Read from system of REFERENCE (get cached/detailed data)
  const data = yield* ctx.run(readFromReference, id);

  return data;
}
```

**Why this order?**
- Record system determines existence
- Reference system may be stale or missing data
- Reading reference first could return data for non-existent entity

## Real-World Example: Account Creation with TigerBeetle

```typescript
import { Context } from "@resonatehq/sdk";
import { Client, type Account } from "tigerbeetle-node";
import type Database from "better-sqlite3";

type Result =
  | { type: "created" }
  | { type: "exists_same" }
  | { type: "exists_diff" };

/**
 * Create account in SQLite (System of Reference)
 *
 * A record here is staged but does NOT determine existence
 */
function sqCreateAccount(
  ctx: Context,
  uuid: string,
  guid: string
): Result {
  const db = ctx.getDependency<Database.Database>("sqClient");

  try {
    db.prepare("INSERT INTO accounts (uuid, guid) VALUES (?, ?)")
      .run(uuid, guid);

    return { type: "created" };
  } catch (error: any) {
    // Handle unique constraint violations
    if (error.message?.includes("UNIQUE constraint failed")) {
      const existing = db
        .prepare("SELECT guid FROM accounts WHERE uuid = ?")
        .get(uuid) as { guid: string } | undefined;

      if (existing && existing.guid === guid) {
        return { type: "exists_same" };  // Idempotent retry
      } else {
        return { type: "exists_diff" };  // Conflict!
      }
    }

    throw new Error(`Failed to create account in SQLite: ${error.message}`);
  }
}

/**
 * Create account in TigerBeetle (System of Record)
 *
 * A record here IS COMMITTED and DOES determine existence
 */
async function tbCreateAccount(
  ctx: Context,
  guid: string
): Promise<Result> {
  const client = ctx.getDependency<Client>("tbClient");

  const account: Account = {
    id: BigInt(guid),
    debits_pending: 0n,
    debits_posted: 0n,
    credits_pending: 0n,
    credits_posted: 0n,
    ledger: 1,
    code: 1,
    flags: 0,
    user_data_128: 0n,
    user_data_64: 0n,
    user_data_32: 0,
    reserved: 0,
    timestamp: 0n
  };

  const errors = await client.createAccounts([account]);

  if (errors.length === 0) {
    return { type: "created" };
  }

  const error = errors[0];

  if (error.result === "exists") {
    return { type: "exists_same" };  // Idempotent retry
  }

  if (error.result.includes("exists_with_different")) {
    return { type: "exists_diff" };  // Conflict!
  }

  throw new Error(`Failed to create account: ${JSON.stringify(error)}`);
}

/**
 * Create account using "Write Last, Read First" principle
 *
 * Resonate guarantees:
 * - Eventual completion via checkpointing
 * - Reliable resumption after crashes
 * - Idempotent retries
 */
function* createAccount(
  ctx: Context,
  uuid: string
): Generator<any, { uuid: string; guid: string }, any> {
  // 1. Generate ID
  const guid = yield* ctx.run((ctx: Context) => {
    return generateId().toString();
  });

  // 2. Write to REFERENCE (staging)
  const sqResult = yield* ctx.run(sqCreateAccount, uuid, guid);

  // 3. Panic if reference shows conflicting data
  yield* ctx.panic(sqResult.type === "exists_diff");

  // 4. Write to RECORD (commits existence)
  const tbResult = yield* ctx.run(tbCreateAccount, guid);

  // 5. Panic if record shows conflicting data
  yield* ctx.panic(tbResult.type === "exists_diff");

  // 6. Panic if ordering was violated
  // (reference says "created" but record says "exists_same")
  yield* ctx.panic(
    sqResult.type === "created" && tbResult.type === "exists_same"
  );

  return { uuid, guid };
}

function generateId(): bigint {
  // Generate unique ID for TigerBeetle
  return BigInt(Date.now()) * 1000000n + BigInt(Math.floor(Math.random() * 1000000));
}
```

## Using ctx.panic() for Invariant Violations

`ctx.panic()` signals unrecoverable violations that require operator intervention:

```typescript
// Panic if data conflicts
yield* ctx.panic(result.type === "exists_diff");

// Panic if ordering was violated
yield* ctx.panic(
  referenceCreated && recordExisted
);
```

**When panic occurs:**
- Workflow permanently enters panic state
- Operator receives alert
- Manual investigation required
- Cannot auto-recover

**When to panic:**
- Data inconsistency detected
- Invariant violation
- Corruption or system integrity issues

## Pattern Variations

### Pattern 1: Write Last, Read First with Rollback

```typescript
function* createWithRollback(
  ctx: Context,
  id: string,
  data: any
) {
  // Write to reference
  yield* ctx.run(writeToReference, id, data);

  try {
    // Write to record (may fail)
    yield* ctx.run(writeToRecord, id, data);
  } catch (error) {
    // Rollback reference write
    yield* ctx.run(deleteFromReference, id);
    throw error;
  }

  return { id, status: "created" };
}
```

### Pattern 2: Two-Phase with Validation

```typescript
function* createWithValidation(
  ctx: Context,
  id: string,
  data: any
) {
  // Phase 1: Validate
  const valid = yield* ctx.run(validateData, data);
  if (!valid) {
    throw new Error("Invalid data");
  }

  // Phase 2: Write to reference
  yield* ctx.run(writeToReference, id, data);

  // Phase 3: Write to record
  yield* ctx.run(writeToRecord, id, data);

  // Phase 4: Verify consistency
  const consistent = yield* ctx.run(verifyConsistency, id);
  yield* ctx.panic(!consistent);

  return { id, status: "created" };
}
```

### Pattern 3: Compensating Actions

```typescript
function* createWithCompensation(
  ctx: Context,
  id: string,
  data: any
) {
  const state = { referenceWritten: false, recordWritten: false };

  try {
    // Write to reference
    yield* ctx.run(writeToReference, id, data);
    state.referenceWritten = true;

    // Write to record
    yield* ctx.run(writeToRecord, id, data);
    state.recordWritten = true;

    return { id, status: "created" };
  } catch (error) {
    // Compensate based on state
    if (state.recordWritten) {
      // Can't undo record write - panic
      yield* ctx.panic(true);
    } else if (state.referenceWritten) {
      // Undo reference write
      yield* ctx.run(deleteFromReference, id);
    }
    throw error;
  }
}
```

## Idempotency

Both systems must handle idempotent retries:

```typescript
// Reference system (SQLite)
try {
  db.prepare("INSERT INTO accounts (uuid, guid) VALUES (?, ?)").run(uuid, guid);
  return { type: "created" };
} catch (error) {
  // Check if this is a retry with same data
  const existing = db.prepare("SELECT guid FROM accounts WHERE uuid = ?").get(uuid);

  if (existing && existing.guid === guid) {
    return { type: "exists_same" };  // Idempotent retry - OK
  } else {
    return { type: "exists_diff" };  // Conflict - PANIC
  }
}

// Record system (TigerBeetle)
const errors = await client.createAccounts([account]);

if (errors.length === 0) {
  return { type: "created" };
}

if (errors[0].result === "exists") {
  return { type: "exists_same" };  // Idempotent retry - OK
}

if (errors[0].result.includes("exists_with_different")) {
  return { type: "exists_diff" };  // Conflict - PANIC
}
```

## Recovery Scenarios

### Scenario 1: Crash After Reference Write

```
Initial execution:
  ✓ Write to reference (SQLite)
  ✗ CRASH

Resume:
  → Resonate replays from beginning
  → Reference write: "exists_same" (idempotent)
  → Record write: Completes successfully
  → Entity now exists
```

### Scenario 2: Crash After Record Write

```
Initial execution:
  ✓ Write to reference
  ✓ Write to record
  ✗ CRASH before return

Resume:
  → Resonate replays from beginning
  → Reference write: "exists_same" (idempotent)
  → Record write: "exists_same" (idempotent)
  → Return success
```

### Scenario 3: Partial Completion Detected

```
Initial execution:
  ✗ Reference write failed

Resume:
  → Different execution path (random ID generated differently)
  → Reference write: Success with NEW guid
  → Record write: "exists_same" (record from previous execution!)
  → Panic: Reference shows "created", Record shows "exists_same"
  → Ordering violation detected - manual investigation required
```

## Multi-System Pattern

```typescript
function* createAcrossMultipleSystems(
  ctx: Context,
  id: string,
  data: any
) {
  // Write to all reference systems first
  yield* ctx.run(writeToCache, id, data);
  yield* ctx.run(writeToApplicationDB, id, data);
  yield* ctx.run(writeToSearchIndex, id, data);

  // Write to system of record LAST
  yield* ctx.run(writeToSystemOfRecord, id, data);

  // Now entity officially exists
  return { id, status: "created" };
}
```

## Reading Pattern

```typescript
function* readEntity(ctx: Context, id: string) {
  // ALWAYS check system of record FIRST
  const existsInRecord = yield* ctx.run(
    checkSystemOfRecord,
    id
  );

  if (!existsInRecord) {
    // Doesn't exist, regardless of reference systems
    return null;
  }

  // Entity exists - now read from reference systems for performance
  const cachedData = yield* ctx.run(readFromCache, id);

  if (cachedData) {
    return cachedData;
  }

  // Cache miss - read from application DB
  const dbData = yield* ctx.run(readFromApplicationDB, id);

  // Populate cache for next time
  yield* ctx.run(writeToCache, id, dbData);

  return dbData;
}
```

## Common Pitfalls

### 1. Writing to Record First

```typescript
// ❌ WRONG - Writes to record first
function* wrongOrder(ctx: Context, id: string, data: any) {
  yield* ctx.run(writeToRecord, id, data);  // Commits existence!
  yield* ctx.run(writeToReference, id, data);  // If this fails, entity exists but reference is missing
}

// ✅ CORRECT - Writes to record last
function* correctOrder(ctx: Context, id: string, data: any) {
  yield* ctx.run(writeToReference, id, data);  // Safe staging
  yield* ctx.run(writeToRecord, id, data);  // Commits existence
}
```

### 2. Reading Reference First

```typescript
// ❌ WRONG - Reads reference first
function* wrongReadOrder(ctx: Context, id: string) {
  const data = yield* ctx.run(readFromReference, id);
  if (data) return data;  // Might return data for non-existent entity!
  return null;
}

// ✅ CORRECT - Reads record first
function* correctReadOrder(ctx: Context, id: string) {
  const exists = yield* ctx.run(checkRecord, id);
  if (!exists) return null;
  return yield* ctx.run(readFromReference, id);
}
```

### 3. Missing Idempotency

```typescript
// ❌ WRONG - Not idempotent
async function writeToReference(ctx: Context, id: string, data: any) {
  await db.insert({ id, data });  // Fails on retry!
}

// ✅ CORRECT - Idempotent
async function writeToReference(ctx: Context, id: string, data: any) {
  try {
    await db.insert({ id, data });
    return { type: "created" };
  } catch (error) {
    // Check if retry with same data
    const existing = await db.findById(id);
    if (existing && existing.data === data) {
      return { type: "exists_same" };
    }
    return { type: "exists_diff" };
  }
}
```

### 4. Not Using Panic for Conflicts

```typescript
// ❌ WRONG - Silently continues with conflict
const result = yield* ctx.run(writeToRecord, id, data);
if (result.type === "exists_diff") {
  console.error("Conflict detected");
  return { status: "error" };
}

// ✅ CORRECT - Panics on conflict
const result = yield* ctx.run(writeToRecord, id, data);
yield* ctx.panic(result.type === "exists_diff");
```

## Decision Tree

**Use this pattern when:**
- Multiple systems must stay consistent
- No distributed transaction support
- One system is clearly the source of truth
- External systems can't participate in transactions

**Don't use this pattern when:**
- Single database with ACID transactions
- All systems support 2PC/distributed transactions
- No clear system of record

## Summary

The External System of Record pattern enables:
- Consistency across multiple systems without distributed transactions
- Clear designation of authoritative source of truth
- Durable execution guarantees eventual completion
- Idempotent retry handling
- Automatic recovery from crashes

**Core recipe:** Write to reference systems first → Write to record system last → Read from record system first → Use `ctx.panic()` for invariant violations
