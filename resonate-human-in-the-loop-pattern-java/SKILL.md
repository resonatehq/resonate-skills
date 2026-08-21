---
name: resonate-human-in-the-loop-pattern-java
description: Implement human-in-the-loop workflows in Java with the Resonate SDK — durable functions that park on ctx.promise() until an external actor settles the latent promise. The Java SDK ships an r.promises sub-client, so external resolution is a clean r.promises.resolve(id, new Value(...)) call (or the CLI / server HTTP API) with no manual base64 encoding — the Go SDK's 0.1.0 tag now has the same Promises() sub-client, with manual base64 encoding only on its low-level Sender().PromiseSettle fallback. Use for approval gates, webhook callbacks, and operator unblock steps. Verified against develop/java.mdx (docs PR #230) and the resonate-sdk-java source at io.resonatehq:resonate-sdk-java:0.1.1.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern — Java

> **Prerelease note.** `resonate-sdk-java` is published on Maven Central — pin `io.resonatehq:resonate-sdk-java:0.1.1`. The API mirrors the Python SDK and may change before a stable `1.0`. **Requires Java 21+** (virtual threads, a feature generally available in Java 21). There is no Java HITL example repo yet, so every code block here uses only documented SDK surface, compile-verified against `0.1.1` and cross-checked against `develop/java.mdx` (docs PR #230) and the SDK source.

## Overview

For the language-agnostic mental model, start with `resonate-human-in-the-loop-pattern-typescript`. The idea is identical: create a latent durable promise, hand its ID to the external actor who will settle it, and await — the workflow parks until settlement arrives, surviving any number of crashes or restarts.

**Resolution from Java is direct.** The Java SDK ships a top-level `r.promises` sub-client (it mirrors Python), so settling a promise from outside the workflow is `r.promises.resolve(id, new Value(...))`. There is **no manual base64 encoding** — that extra step is a Go-specific workaround for a sub-client Go lacks. Don't copy it into Java.

## When to use

- Approval gates (budget, deploy, content moderation)
- Third-party webhook callbacks (Stripe, DocuSign, Twilio)
- Operator unblock steps in runbooks
- Any step where the decision or data originates outside the Resonate worker set

## Basic shape

### Workflow side — `ctx.promise()` → `promise.id()` → publish → `await`

```java
import io.resonatehq.resonate.Context;
import io.resonatehq.resonate.Context.ResonateFuture;
import java.time.Duration;

public final class Approval {
    private Approval() {}

    public record ReviewRequest(String item, String requester) {}

    /**
     * Parks until an external actor settles the latent promise. promise.id() returns the id once
     * the promise has been created (instantaneous in local mode, a network round-trip against a
     * real server); await suspends the workflow (durably) until it is settled from outside.
     */
    public static String approvalWorkflow(Context ctx, ReviewRequest req) {
        // A latent durable promise — no registered function backs it; it settles only when an
        // external caller resolves it. 24-hour timeout, capped at the workflow deadline.
        ResonateFuture<Object> promise = ctx.promise(Duration.ofHours(24));
        String approvalId = promise.id();

        // Publish the promise ID inside a ctx.run so the side effect is checkpointed and does
        // not re-run on replay. In production: write to a DB, push to a notification queue, etc.
        ctx.run(Approval::publishApprovalId, req.item(), approvalId).await();

        // Suspend until the promise is settled externally; decode the decision.
        Object decision = promise.await();
        return "item " + req.item() + " decided: " + decision;
    }

    /** Checkpointed publication of the promise ID. Replace the println with a real notification. */
    public static String publishApprovalId(Context ctx, String item, String approvalId) {
        System.out.printf("  [workflow] awaiting approval for %s — promise id: %s%n", item, approvalId);
        return "published";
    }
}
```

Key points:
- `ctx.promise()` takes an optional `Duration` timeout; `ctx.promise()` with no arg uses a 1-day default capped at the parent's remaining deadline.
- `promise.id()` returns the promise's id once it has been created — instantaneous in local mode, blocking on the create round-trip against a real server.
- Publish the ID **inside a `ctx.run`** so the publication is itself durable. A bare side effect above `promise.await()` re-runs on every replay pass.
- `promise.await()` with no decode is fine when you only need to know the promise settled; here we read the decision `Object`.

## Resolving from outside

The Java SDK gives you three mechanisms. Prefer the sub-client when resolving from Java.

### 1. The `promises` sub-client (preferred when resolving from Java)

```java
import io.resonatehq.resonate.Types.Value;

// approve — the Value's data is decoded into what the workflow's await expects
r.promises.resolve(approvalId, new Value(null, "approved")).join();

// reject
r.promises.reject(approvalId, new Value(null, "rejected: budget exceeded")).join();

// cancel
r.promises.cancel(approvalId, new Value(null, "withdrawn")).join();
```

`new Value(headers, data)` — pass `null` headers and the decision as `data`. Each call returns a `CompletableFuture<PromiseRecord>`, so `.join()` (or compose with `thenApply`) to wait for it. `r.promises` also exposes `get`, `create`, and `search`. **No base64 encoding is required** — the SDK codec handles the durability boundary for you.

### 2. CLI (simplest — ops gates, manual approval)

```shell
resonate promise resolve <approvalId> --data '"approved"'
resonate promise reject  <approvalId> --data '"rejected: budget exceeded"'
```

The `--data` value is the JSON payload decoded into the type the workflow's `await` expects. For a `String`-shaped decision, a quoted JSON string like `'"approved"'` is correct. See `resonate-cli` for the full flag reference.

### 3. Server HTTP API (any language / tool)

POST to the server's promise-settle endpoint with the promise ID and a JSON body — useful from a webhook handler, an admin script, or a serverless function in any language. Going straight to the server bypasses the SDK codec, so the `data` field must be the **base64 encoding of the JSON-serialized payload** (the wire format the SDK stores — `value → JSON → base64`, `Codec.java`). `"approved"` as JSON, base64-encoded, is `ImFwcHJvdmVkIg==`:

```shell
curl -s -X POST "http://localhost:8001/promises/${APPROVAL_ID}/resolve" \
     -H "Content-Type: application/json" \
     -d '{"value":{"data":"ImFwcHJvdmVkIg=="}}'
```

This base64 step is exactly what `r.promises.resolve` and the CLI do for you — prefer mechanism 1 or 2 from Java/ops, and reach for the raw endpoint only from a non-Java caller. The `ImFwcHJvdmVkIg==` value is correct for the default `NoopEncryptor`; if you configured an `Encryptor` on the builder, the payload is encrypted before base64, so a raw caller would have to replicate that encryption too — another reason to use the sub-client or CLI.

## Hand-off across processes

A realistic deployment has the worker that creates the promise and the process that resolves it in different programs. The pattern:

1. The worker workflow creates the promise and persists `promise.id()` somewhere the resolver can read it (a DB row, a notification, a ticket).
2. The external actor (an HTTP webhook handler, an operator running the CLI, an admin service) settles it by ID via one of the three mechanisms above.
3. The worker's `promise.await()` wakes and returns the decoded decision — even if the worker crashed and restarted in between.

Because resolution is by promise ID, the resolver needs nothing but the ID and a connection to the same Resonate server.

## Known gaps

- **No hand-chosen promise IDs.** `ctx.promise()` generates the ID internally; you cannot pass an application-layer ID like `"approval/order-42"`. Fetch the ID via `promise.id()` after creation and publish it explicitly. This may change as the API settles toward `1.0`.

## Distinct Java idioms

- **`r.promises.resolve(id, new Value(null, data))`** — the clean external-resolution path. No base64 layer (that's a Go-only workaround). `Value(headers, data)` takes `null` headers and the payload as `data`.
- **`CompletableFuture` from the sub-client** — `r.promises.resolve(...)` returns a `CompletableFuture<PromiseRecord>`; call `.join()` to wait or `thenApply` to compose.
- **Publish the ID inside `ctx.run`.** Java durable functions are not generators (no `yield`); wrap every observable side effect — DB write, notification, the hand-off that publishes the promise ID — in a `ctx.run` so it is checkpointed and not repeated on replay.
- **`ctx.promise(Duration)` vs `ctx.promise()`** — the `Duration` overload sets an explicit timeout; the no-arg form uses the 1-day default capped at the parent deadline.
- **`promise.await()` returns `Object`** — `ctx.promise()` is untyped; cast or `.toString()` the decoded decision as needed.

## Avoid

- **Copying the Go base64 encoding dance.** The Java SDK has `r.promises.resolve` — use `new Value(null, data)` directly. Manual `JSON → base64 → quoted` encoding is a Go-only workaround for a sub-client that Go lacks.
- **Polling via `ctx.sleep` + a status check.** Defeats park-and-resume; burns checkpoints and wall-clock time. Use `ctx.promise()` + `promise.await()`.
- **Publishing the promise ID outside a `ctx.run`.** A bare write above `promise.await()` re-runs on every replay. Checkpoint the publication.
- **Settling before publishing the ID.** A narrow race: if the resolver runs before the `ctx.run` that publishes the ID is checkpointed, a crash between those lines can lose the ID. Checkpoint the publication first, then await.
- **Long-blocking work inside `ctx.run`.** `ctx.run` functions must return promptly; a blocking call holds the task lease open until TTL expires. Long waits belong in `ctx.promise` (latent, externally settled).

## Related skills

- `resonate-basic-durable-world-usage-java` — `ctx.promise`, `ResonateFuture.await`, the replay model
- `resonate-basic-ephemeral-world-usage-java` — the `r.promises` sub-client surface (`resolve` / `reject` / `cancel` / `get` / `create` / `search`)
- `resonate-cli` — `resonate promise resolve / reject / cancel` flag reference
- `durable-execution` — foundational replay semantics; why `await` survives crashes
- `resonate-human-in-the-loop-pattern-typescript` — language-agnostic mental model
- `resonate-human-in-the-loop-pattern-python` — the closest sibling; the Java `promises` sub-client mirrors Python
