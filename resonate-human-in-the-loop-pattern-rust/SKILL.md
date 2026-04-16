---
name: resonate-human-in-the-loop-pattern-rust
description: Implement human-in-the-loop workflows in Rust with Resonate — durable functions that block on ctx.promise::<T>() until an external actor (webhook, UI, CLI, operator) resolves via resonate.promises.resolve/reject. Use when a Rust workflow step must wait on a decision or data that doesn't come from another worker. v0.1.0 caveat: ctx.promise is in sdk-rs source but not yet in rust.mdx.
license: Apache-2.0
---

# Resonate Human-in-the-Loop Pattern — Rust

> **v0.1.0 caveat.** `ctx.promise::<T>()` is a real `pub fn` in the Rust SDK source (`resonate-sdk-rs:resonate/src/context.rs:352`) with a full `PromiseTask<T>` builder, but it is not yet covered in `docs/develop/rust.mdx` as of April 2026. The API is safe to use; cite source paths when reviewers ask. APIs may shift between v0.x releases.

## Overview

A human-in-the-loop workflow blocks a durable function on a promise that an external actor settles — a reviewer clicking "approve," a webhook firing from a third-party system, an operator running a CLI command. The worker doesn't poll; it awaits the `PromiseTask` future and Resonate parks the execution until the promise settles.

For the language-agnostic mental model, see `resonate-human-in-the-loop-pattern-typescript`. The Rust shape differs in (a) type-parameterized promises `ctx.promise::<T>()`, (b) lazy builder with `.timeout` / `.data` / `.id` / `.create` / `.await`, (c) SDK-generated IDs you fetch via `.id().await?`.

## When to use

- Approval gates (expense, deploy, content moderation)
- Third-party callbacks (Stripe, DocuSign, Twilio)
- Operator unblock steps (break-glass in runbooks)
- Any step where the data or decision comes from outside the Resonate worker set

## Basic shape

```rust
use resonate::prelude::*;
use serde::{Serialize, Deserialize};
use std::time::Duration;

#[derive(Serialize, Deserialize)]
struct Decision {
    approved: bool,
    reviewer: String,
}

#[resonate::function]
async fn expense_approval(ctx: &Context, expense_id: String) -> Result<String> {
    // build a promise with a 24-hour SLA and the expense ID attached as data
    let task = ctx
        .promise::<Decision>()
        .timeout(Duration::from_secs(24 * 60 * 60))
        .data(&serde_json::json!({ "expense_id": expense_id }))?;

    // fetch the SDK-generated promise ID so an external actor can target it
    let promise_id = task.id().await?;

    // stash the ID where the reviewer UI / webhook will see it
    ctx.run(save_approval_id, (expense_id.clone(), promise_id.clone())).await?;

    // block until someone resolves/rejects/cancels the promise
    let decision: Decision = task.await?;

    if decision.approved {
        ctx.run(process_reimbursement, expense_id.clone()).await?;
        Ok(format!("approved by {}", decision.reviewer))
    } else {
        Ok(format!("rejected"))
    }
}

async fn save_approval_id((expense_id, promise_id): (String, String)) -> Result<()> {
    // write (expense_id, promise_id) to a DB or queue a notification
    Ok(())
}

async fn process_reimbursement(expense_id: String) -> Result<()> {
    Ok(())
}
```

From outside the worker — a webhook, CLI, or admin UI — resolve it via the ephemeral-world `Resonate` client using the fetched `promise_id`:

```rust
use serde_json::json;

// in an Axum/Actix handler, or a separate admin CLI
let resonate = Resonate::new(ResonateConfig::default());
resonate
    .promises
    .resolve(&promise_id, json!({ "approved": true, "reviewer": "alice@acme.io" }))
    .await?;

// or reject:
resonate
    .promises
    .reject(&promise_id, json!({ "reviewer": "bob@acme.io" }))
    .await?;

// or cancel (settles as rejected_canceled):
resonate.promises.cancel(&promise_id, json!(null)).await?;
```

The resolved JSON is deserialized into the `T` you type-parameterized the promise with — `Decision` in this example.

## Eagerly create, hand off the future

If the call graph wants to create the promise *now* but await it *later* (e.g., after kicking off downstream work), use `.create()` to get a `RemoteFuture<T>` handle:

```rust
#[resonate::function]
async fn approve_and_also_work(ctx: &Context, job_id: String) -> Result<Decision> {
    // eagerly create the promise record so downstream code can see its ID
    let approval: RemoteFuture<Decision> = ctx
        .promise::<Decision>()
        .timeout(Duration::from_secs(3600))
        .create()
        .await?;

    // do other durable work in parallel
    ctx.run(send_reviewer_notification, job_id.clone()).await?;

    // later, block on the human
    let decision = approval.await?;
    Ok(decision)
}
```

This matches the TS `ctx.promise()` → `yield promise` split into two calls, giving the parent function latitude to do other work between creation and await.

## Webhook-driven resolution (Axum)

A typical third-party-callback route:

```rust
use axum::{extract::Path, Json, routing::post, Router};
use serde_json::Value;

async fn docusign_webhook(
    Path(envelope_id): Path<String>,
    Json(body): Json<Value>,
) -> axum::http::StatusCode {
    let resonate = Resonate::new(ResonateConfig::default());
    let status = body.get("status").and_then(|v| v.as_str()).unwrap_or("");

    let promise_id = format!("docusign:{}", envelope_id);

    match status {
        "completed" => {
            let _ = resonate
                .promises
                .resolve(&promise_id, serde_json::json!({ "signed": true }))
                .await;
        }
        "declined" | "voided" => {
            let _ = resonate
                .promises
                .reject(&promise_id, serde_json::json!({ "reason": status }))
                .await;
        }
        _ => {}
    }

    axum::http::StatusCode::OK
}

pub fn app() -> Router {
    Router::new().route("/webhooks/docusign/:envelope_id", post(docusign_webhook))
}
```

The durable function that awaits the DocuSign promise would have created it with `ctx.promise().timeout(...).create().await?` earlier — once the webhook hits, the worker's `.await?` unblocks and the workflow continues.

## Multi-approver fan-out (all must approve)

```rust
#[resonate::function]
async fn multi_approver(ctx: &Context, request_id: String) -> Result<String> {
    let approvers = vec!["alice", "bob", "carol"];

    // create one promise per approver, fetch IDs in parallel
    let mut tasks = Vec::with_capacity(approvers.len());
    for approver in &approvers {
        let task = ctx
            .promise::<Decision>()
            .timeout(Duration::from_secs(48 * 60 * 60))
            .data(&serde_json::json!({ "approver": approver, "request": request_id }))?;
        let id = task.id().await?;
        ctx.run(notify_approver, (approver.to_string(), id.clone())).await?;
        tasks.push(task);
    }

    // block on every promise in order; any rejection propagates
    let mut approvals = Vec::with_capacity(tasks.len());
    for task in tasks {
        approvals.push(task.await?);
    }

    if approvals.iter().all(|d| d.approved) {
        Ok("approved".into())
    } else {
        Ok("rejected".into())
    }
}

async fn notify_approver((approver, promise_id): (String, String)) -> Result<()> {
    // send email, Slack, or queue a UI task
    Ok(())
}
```

"First-to-respond wins" races aren't directly supported — there's no `select!` equivalent on `PromiseTask<T>` yet. Workaround: spawn a helper workflow per approver and have the first to resolve mark a shared promise via `resonate.promises.resolve` from outside the Resonate execution layer.

## Distinct Rust idioms

- **Typed promises `PromiseTask<T>`** — `ctx.promise::<Decision>()` makes the return type explicit at creation; no turbofish needed when `T` is inferable from downstream use, otherwise `.await` requires `T: DeserializeOwned`
- **`.id().await?` before `.await?`** — the SDK generates IDs; fetch via `.id()` if you need to stash it before blocking. If you don't need the ID, skip this step and go straight to `.await`
- **`serde_json::json!` + `&impl Serialize`** — `.data()` takes any serializable; `json!` macro is the fastest path for ad-hoc payloads. Returns `Result<Self>` because serialization can fail — handle the `?`
- **`RemoteFuture<T>`** — the handle returned by `.create()`. Implements `Future<Output = Result<T>>`; await when ready
- **Axum/Actix webhook handlers** — fire-and-return-200 is the right shape; the SDK's `.resolve()` is async but webhook handlers shouldn't block the incoming request on SDK internals beyond network latency

## Avoid

- **Polling via `ctx.sleep` + a status check** — defeats the durable-await semantics; costs checkpoints + wall-clock time
- **Using the promise ID outside serde-compatible payloads** — if an external actor constructs a custom JSON payload that doesn't deserialize to your typed `T`, `.await?` returns a deserialization error. Match the schema on both sides or use `serde_json::Value` for flexibility
- **Assuming promise IDs are stable across invocations** — the SDK generates them per-invocation within the call graph. A replay reuses the same ID for the same logical promise (idempotency), but you cannot hand-craft IDs like you could in TS/Python's `ctx.promise(id=...)` at v0.1.0

## Related skills

- `resonate-basic-ephemeral-world-usage-rust` — `resonate.promises.create/resolve/reject/cancel` from the webhook side
- `resonate-basic-durable-world-usage-rust` — `ctx.promise::<T>()` builder details
- `resonate-saga-pattern-rust` — sagas with human-gated forward steps
- `resonate-human-in-the-loop-pattern-typescript` / `-python` — sibling SDKs for comparison
- `durable-execution` — foundational replay semantics; promise-await survives crashes
