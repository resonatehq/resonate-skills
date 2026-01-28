---
name: resonate-durable-sleep-scheduled-work-typescript
description: Implement durable sleep and scheduled work patterns for long-running timers, countdowns, scheduled notifications, and delayed execution. Use ctx.sleep() for delays that survive crashes and span hours, days, or weeks.
---

# Resonate Durable Sleep & Scheduled Work (TypeScript)

## Overview

Resonate's `ctx.sleep()` creates suspension points where workflows pause without consuming resources. Unlike regular `setTimeout()` or `sleep()`, durable sleep survives crashes, restarts, and redeployments. The workflow suspends its state, and Resonate resumes it exactly where it left off after the delay expires.

**Core principle:** `yield* ctx.sleep(milliseconds)` suspends execution durably. Workflows can sleep for seconds, hours, days, or weeks without holding threads, connections, or memory.

## Mental Model

```
Regular Sleep (Ephemeral):
  Process → sleep(10h) → [BLOCKS THREAD] → Resume
  Crash during sleep? → LOST, never resumes

Durable Sleep (Resonate):
  Workflow → yield* ctx.sleep(10h) → [SUSPENDS STATE] → Resume
  Crash during sleep? → Automatically resumes after delay
  Restart process? → Resumes from checkpoint after delay

No resources held during sleep - workflow exists only as durable state.
```

## Basic Pattern: Simple Countdown

```typescript
import { Context } from "@resonatehq/sdk";

function* countdown(
  ctx: Context,
  count: number,
  delayMinutes: number,
  notificationUrl: string
) {
  for (let i = count; i > 0; i--) {
    // Send notification
    yield* ctx.run(sendNotification, notificationUrl, `Countdown: ${i}`);

    // Durable sleep - workflow suspends here
    yield* ctx.sleep(delayMinutes * 60 * 1000);
  }

  // Send final notification
  yield* ctx.run(sendNotification, notificationUrl, "Done!");
}

async function sendNotification(
  _ctx: Context,
  url: string,
  message: string
) {
  await fetch(url, {
    method: "POST",
    body: message,
    headers: { "Content-Type": "text/plain" }
  });
}
```

**Usage:**
```typescript
// Count down from 5, waiting 1 minute between each count
await resonate.run(
  "countdown-1",
  countdown,
  5,              // count
  1,              // delay in minutes
  "https://ntfy.sh/mychannel"
);
```

**What happens:**
1. Sends "Countdown: 5"
2. Sleeps 1 minute (workflow suspends, no resources held)
3. Resumes, sends "Countdown: 4"
4. Sleeps 1 minute
5. Repeats until "Done!"

## Pattern: Scheduled Reminder

```typescript
function* scheduleReminder(
  ctx: Context,
  userId: string,
  message: string,
  delayMs: number
) {
  // Sleep until reminder time
  yield* ctx.sleep(delayMs);

  // Send reminder
  yield* ctx.run(sendEmail, userId, "Reminder", message);

  return { sent: true, timestamp: Date.now() };
}

// Schedule reminder 24 hours from now
await resonate.run(
  `reminder/${userId}/${Date.now()}`,
  scheduleReminder,
  "user-123",
  "Don't forget to review the document!",
  24 * 60 * 60 * 1000  // 24 hours
);
```

## Pattern: Recurring Notification

```typescript
function* recurringNotification(
  ctx: Context,
  userId: string,
  intervalHours: number,
  totalOccurrences: number
) {
  for (let i = 1; i <= totalOccurrences; i++) {
    // Send notification
    yield* ctx.run(async () => {
      await emailService.send({
        to: userId,
        subject: `Daily Update #${i}`,
        body: `This is occurrence ${i} of ${totalOccurrences}`
      });
    });

    // Sleep until next occurrence (unless this was the last one)
    if (i < totalOccurrences) {
      yield* ctx.sleep(intervalHours * 60 * 60 * 1000);
    }
  }

  return { completed: true, sent: totalOccurrences };
}

// Send daily notifications for 7 days
await resonate.run(
  `daily-updates/${userId}`,
  recurringNotification,
  "user@example.com",
  24,  // every 24 hours
  7    // 7 times total
);
```

## Pattern: Exponential Backoff with Sleep

```typescript
function* retryWithBackoff(
  ctx: Context,
  operation: string,
  maxAttempts: number = 5
) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      // Try the operation
      const result = yield* ctx.run(performOperation, operation);
      return { success: true, attempt, result };
    } catch (error) {
      if (attempt === maxAttempts) {
        // Final attempt failed
        return { success: false, attempts: maxAttempts, error };
      }

      // Exponential backoff: 2^attempt seconds
      const delayMs = Math.pow(2, attempt) * 1000;
      console.log(`Attempt ${attempt} failed, retrying in ${delayMs}ms`);

      // Durable sleep before retry
      yield* ctx.sleep(delayMs);
    }
  }
}
```

## Pattern: Rate Limiting with Sleep

```typescript
function* processWithRateLimit(
  ctx: Context,
  items: string[],
  itemsPerMinute: number
) {
  const delayMs = (60 * 1000) / itemsPerMinute;
  const results = [];

  for (let i = 0; i < items.length; i++) {
    // Process item
    const result = yield* ctx.run(processItem, items[i]);
    results.push(result);

    // Sleep to enforce rate limit (except after last item)
    if (i < items.length - 1) {
      yield* ctx.sleep(delayMs);
    }
  }

  return { processed: results.length, results };
}

// Process 10 items per minute
await resonate.run(
  "batch-process-1",
  processWithRateLimit,
  items,
  10  // 10 items per minute
);
```

## Pattern: Multi-Stage Delayed Workflow

```typescript
function* onboardingFlow(
  ctx: Context,
  userId: string,
  email: string
) {
  // Day 0: Welcome email
  yield* ctx.run(sendEmail, email, "Welcome!", welcomeTemplate);

  // Day 1: Sleep 24 hours, send tips
  yield* ctx.sleep(24 * 60 * 60 * 1000);
  yield* ctx.run(sendEmail, email, "Getting Started Tips", tipsTemplate);

  // Day 3: Sleep 48 hours, send feature highlight
  yield* ctx.sleep(48 * 60 * 60 * 1000);
  yield* ctx.run(sendEmail, email, "Feature Highlight", featureTemplate);

  // Day 7: Sleep 96 hours, send survey
  yield* ctx.sleep(96 * 60 * 60 * 1000);
  yield* ctx.run(sendEmail, email, "How are we doing?", surveyTemplate);

  return { completed: true, userId };
}
```

## Pattern: Scheduled Cleanup Job

```typescript
function* scheduledCleanup(
  ctx: Context,
  intervalDays: number
) {
  while (true) {
    // Perform cleanup
    const deleted = yield* ctx.run(cleanupOldRecords);

    console.log(`Deleted ${deleted} records`);

    // Sleep until next cleanup
    yield* ctx.sleep(intervalDays * 24 * 60 * 60 * 1000);
  }
}

async function cleanupOldRecords(_ctx: Context): Promise<number> {
  // Delete records older than 90 days
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - 90);

  const result = await db.delete({
    table: "logs",
    where: { created_at: { lt: cutoffDate } }
  });

  return result.deletedCount;
}

// Run cleanup every 7 days
await resonate.run(
  "cleanup-job",
  scheduledCleanup,
  7  // every 7 days
);
```

## Pattern: Timeout with Sleep

```typescript
function* operationWithTimeout(
  ctx: Context,
  taskId: string,
  timeoutMinutes: number
) {
  // Start the operation
  const handle = yield* ctx.beginRpc(longRunningOperation, taskId);

  // Start a timeout timer
  const timeoutHandle = yield* ctx.beginRpc(
    timeoutTimer,
    timeoutMinutes
  );

  // Race between operation and timeout
  // Note: Resonate doesn't have built-in race(), so we check completion
  try {
    const result = yield* handle;
    return { success: true, result };
  } catch (error) {
    return { success: false, timedOut: true };
  }
}

function* timeoutTimer(ctx: Context, minutes: number) {
  yield* ctx.sleep(minutes * 60 * 1000);
  throw new Error("Operation timed out");
}
```

## Pattern: Scheduled Batch Processing

```typescript
function* batchProcessingScheduler(
  ctx: Context,
  batchSize: number,
  intervalHours: number
) {
  while (true) {
    // Fetch batch of items to process
    const items = yield* ctx.run(fetchPendingItems, batchSize);

    if (items.length === 0) {
      console.log("No items to process");
    } else {
      // Process each item
      for (const item of items) {
        yield* ctx.run(processItem, item);
      }

      console.log(`Processed ${items.length} items`);
    }

    // Sleep until next batch
    yield* ctx.sleep(intervalHours * 60 * 60 * 1000);
  }
}

// Process batches of 100 items every 6 hours
await resonate.run(
  "batch-processor",
  batchProcessingScheduler,
  100,  // batch size
  6     // every 6 hours
);
```

## Real-World Example: Trial Expiration Workflow

```typescript
function* trialExpirationWorkflow(
  ctx: Context,
  userId: string,
  trialDays: number
) {
  const trialEndDate = new Date();
  trialEndDate.setDate(trialEndDate.getDate() + trialDays);

  // Send welcome email immediately
  yield* ctx.run(sendEmail, userId, "Trial Started", welcomeEmail);

  // Day 7: Reminder
  yield* ctx.sleep(7 * 24 * 60 * 60 * 1000);
  yield* ctx.run(sendEmail, userId, "Trial Reminder", reminderEmail);

  // Day 13: Upgrade prompt
  yield* ctx.sleep(6 * 24 * 60 * 60 * 1000);
  yield* ctx.run(sendEmail, userId, "Upgrade Now", upgradeEmail);

  // Day 14: Trial expires
  yield* ctx.sleep(1 * 24 * 60 * 60 * 1000);

  // Check if user upgraded
  const user = yield* ctx.run(getUser, userId);

  if (!user.isPaid) {
    // Downgrade to free tier
    yield* ctx.run(downgradeAccount, userId);
    yield* ctx.run(sendEmail, userId, "Trial Ended", trialEndedEmail);
  }

  return { userId, upgraded: user.isPaid };
}
```

## Crash Recovery Behavior

**Scenario:** Countdown crashes during sleep

```typescript
Initial execution:
  ✓ Send "Countdown: 5"
  ✓ ctx.sleep(60000) - Creates durable timer
  ✗ CRASH

Resume (after worker restarts):
  → Resonate replays from beginning
  → Send "Countdown: 5" - Idempotent (or checkpointed)
  → ctx.sleep(60000) - Resonate knows this already completed, skips
  → After sleep expires, workflow automatically resumes
  → Continues with "Countdown: 4"
```

**Key insight:** Crashes during sleep don't lose the timer. Resonate tracks the sleep's expiration time durably.

## Sleep Duration Limits

**Practical limits:**
- **Milliseconds:** Minimum sleep duration
- **Days/Weeks:** Common for scheduled workflows
- **Months:** Possible but consider alternatives for very long delays
- **Maximum:** No hard limit, but long sleeps may require server configuration

**Recommendation:** For delays > 90 days, consider using a cron-style scheduler or external job queue.

## Common Pitfalls

### 1. Using setTimeout Instead of ctx.sleep

```typescript
// ❌ WRONG - Not durable, lost on crash
function* badCountdown(ctx: Context, count: number) {
  for (let i = count; i > 0; i--) {
    yield* ctx.run(notify, `Count: ${i}`);
    await new Promise(resolve => setTimeout(resolve, 60000));  // LOST ON CRASH
  }
}

// ✅ CORRECT - Durable sleep
function* goodCountdown(ctx: Context, count: number) {
  for (let i = count; i > 0; i--) {
    yield* ctx.run(notify, `Count: ${i}`);
    yield* ctx.sleep(60000);  // SURVIVES CRASHES
  }
}
```

### 2. Sleeping Outside Generator Context

```typescript
// ❌ WRONG - Can't sleep outside ctx
async function badDelay(ctx: Context) {
  await ctx.run(async () => {
    await new Promise(resolve => setTimeout(resolve, 60000));  // NOT DURABLE
  });
}

// ✅ CORRECT - Sleep in generator
function* goodDelay(ctx: Context) {
  yield* ctx.sleep(60000);  // DURABLE
}
```

### 3. Not Handling Idempotency

```typescript
// ❌ WRONG - Sends duplicate notifications on retry
function* badReminder(ctx: Context, userId: string) {
  await sendEmail(userId, "Reminder");  // NOT IDEMPOTENT
  yield* ctx.sleep(24 * 60 * 60 * 1000);
}

// ✅ CORRECT - Wrapped in ctx.run for idempotency
function* goodReminder(ctx: Context, userId: string) {
  yield* ctx.run(sendEmail, userId, "Reminder");  // IDEMPOTENT
  yield* ctx.sleep(24 * 60 * 60 * 1000);
}
```

## Time Precision

**Resonate's sleep precision:**
- Sleep duration specified in milliseconds
- Actual resume time may vary by seconds (depends on server load, polling intervals)
- **Not suitable for sub-second precision requirements**
- **Suitable for minutes, hours, days**

**Example:**
```typescript
yield* ctx.sleep(60000);  // Sleeps ~60 seconds (±few seconds variance)
yield* ctx.sleep(100);    // Not reliable for 100ms precision
```

## Decision Tree

**Use ctx.sleep() when:**
- Delays span minutes, hours, or days
- Workflow must survive crashes during delay
- Scheduled notifications, reminders, timeouts
- Rate limiting, backoff, periodic tasks

**Don't use ctx.sleep() when:**
- Sub-second precision required (use regular timers)
- Delay is ephemeral (use setTimeout in ctx.run)
- Need to cancel/update sleep dynamically (use promises with timeout)

## Summary

Durable sleep enables workflows to:
- Pause execution for hours, days, or weeks without consuming resources
- Survive crashes, restarts, and redeployments during sleep
- Implement countdowns, scheduled reminders, recurring tasks
- Handle timeouts, rate limiting, exponential backoff
- Create multi-stage delayed workflows

**Core recipe:** `yield* ctx.sleep(milliseconds)` → Workflow suspends → Resonate resumes after delay → Continue execution
