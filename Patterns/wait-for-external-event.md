# Wait for External Event

<!--
## TODO

- [ ] Show raw HTTP API (e.g., curl) for resolving promises so ANY app can resolve, not just JS/TS
- [ ] Timeouts are mandatory: every external promise MUST have a timeout, otherwise the app hangs forever. There is no generic default — ask the user for the appropriate timeout based on use case
- [x] Handle both resolve and reject: reject manifests as an exception in the awaiting function
- [ ] Timeout is also a reject with a specific exception type. For many use cases timeout and rejection are equivalent, but if a use case must distinguish, show how
- [ ] Show end-to-end pattern, not just hello world variant — include database updates, UI integration, error handling
-->

## Alternative Names

- Wait for External Signal
- Wait for External System
- Wait for External Service
- Wait for Webhook
- Wait for Human in the Loop

## Problem

A durable function needs to pause execution and wait for an event that originates outside the call graph. The event may come from:

- A human action (approval, review, decision)
- An external system (webhook, callback)
- A third-party service (payment confirmation, identity verification)

The function cannot proceed until the external event occurs, but it should not block resources while waiting. The wait may last seconds, hours, or days.

## Solution

Use `context.promise()` to create a durable promise that can be resolved externally.

The pattern has three steps:

1. **Create** Create the durable promise
2. **Share** Share the durable promise ID (e.g. store in database, publish on channel)
3. **Await** Await the durable promise

```typescript
function* foo(context: Context, orderId: string) {
  // 1. Create durable promise (does not await)
  const promise = yield* context.promise();

  // 2. Share the durable promise ID
  yield* context.run(share, promise.id);

  // 3. Await the durable promise
  try {
    const result = yield* promise;
    // The durable promise was resolved
    return result;
  } catch (error) {
    // The durable promise was rejected (or timedout)
    return;
  }
}
```

### Share the Durable Promise ID

The Durable Promise ID must be shared with an external actor so the actor can resolve or reject the promise. Common approaches:

**Store in database:**

```typescript
function* approvalWorkflow(context: Context, orderId: string) {
  const promise = yield* context.promise();

  // Store promise ID in database, linked to the order
  yield* context.run(async () => {
    const supabase = context.getDependency("supabase");
    await supabase.from("pending_approvals").insert({
      order_id: orderId,
      promise_id: promise.id,
      status: "pending",
      created_at: new Date().toISOString()
    });
  });

  // Wait for approval
  const approval = yield* promise;

  return approval;
}
```

**Send to external system:**

```typescript
function* paymentWorkflow(context: Context, orderId: string, amount: number) {
  const promise = yield* context.promise();

  // Send promise ID to payment provider as callback reference
  yield* context.run(async () => {
    await paymentProvider.createPayment({
      amount,
      callbackRef: promise.id,  // Provider sends this back in webhook
      webhookUrl: `https://example.com/webhooks/payment`
    });
  });

  // Wait for payment confirmation
  const confirmation = yield* promise;

  return confirmation;
}
```

### Resolving the Promise Externally

The external actor resolves the promise using the Resonate API:

```typescript
// In an API route, webhook handler, or UI action handler
await resonate.promises.resolve(promiseId, {
  approved: true,
  approvedBy: "user@example.com"
});
```

Or to reject:

```typescript
await resonate.promises.reject(promiseId, {
  reason: "Declined by manager"
});
```

## Variants

### With Timeout

Combine with `context.sleep()` to add a timeout. Use `beginRun` to race the promise against a timer:

```typescript
function* approvalWithTimeout(context: Context, orderId: string) {
  const promise = yield* context.promise();

  yield* context.run(sendApprovalRequest, orderId, promise.id);

  // Race: approval vs 24-hour timeout
  const approvalFuture = promise;
  const timeoutFuture = yield* context.beginRun(function* (ctx) {
    yield* ctx.sleep(24 * 60 * 60 * 1000);
    return { timedOut: true };
  });

  // TODO: Resonate may provide a race() primitive
  // For now, handle timeout logic in the external system
  const result = yield* approvalFuture;

  return result;
}
```

### Multiple Approvers

Wait for multiple external events before proceeding:

```typescript
function* multiApprovalWorkflow(context: Context, orderId: string) {
  // Create multiple promises
  const managerPromise = yield* context.promise();
  const financePromise = yield* context.promise();

  // Externalize both
  yield* context.run(sendApprovalRequests, orderId, {
    manager: managerPromise.id,
    finance: financePromise.id
  });

  // Wait for both approvals
  const managerApproval = yield* managerPromise;
  const financeApproval = yield* financePromise;

  if (managerApproval.approved && financeApproval.approved) {
    yield* context.run(processOrder, orderId);
  }

  return { managerApproval, financeApproval };
}
```

## Related Patterns

- **Scheduled Delay**: Use `context.sleep()` for time-based waits instead of event-based waits
- **Saga / Compensation**: Combine with external events for approval gates in multi-step transactions
