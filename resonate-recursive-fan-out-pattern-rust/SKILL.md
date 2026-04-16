---
name: resonate-recursive-fan-out-pattern-rust
description: Implement recursive fan-out in Rust with Resonate — spawn N sub-workflows via .spawn(), optionally recurse deeper, collect results with per-future .await. Use when processing a batch, tree, or crawl where each child is independently durable. v0.1.0 caveat: API surface may change between Rust SDK releases.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern — Rust

> **v0.1.0 caveat.** The Rust SDK is in active development. This pattern uses only documented v0.1.0 surface (`ctx.run().spawn()`, `ctx.rpc().spawn()`, `.await?`). Verify against current SDK source before shipping.

## Overview

Recursive fan-out spawns multiple child invocations in parallel, awaits them individually, and (optionally) recurses deeper. The Rust expression uses `.spawn()` on a Context execution builder to get a `DurableFuture`, collects those futures into a `Vec`, then awaits each in turn.

For the language-agnostic mental model, see `resonate-recursive-fan-out-pattern-typescript`.

## When to use

- Batch processing where items are independent
- Web crawling / tree traversal with dynamic depth
- Map-reduce shaped workflows
- Any fan-out where each leaf is a discrete, retryable unit

## Parallel fan-out in the same process

```rust
use resonate::prelude::*;

#[resonate::function]
async fn enrich_batch(ctx: &Context, order_ids: Vec<String>) -> Result<Vec<String>> {
    let mut futures = Vec::with_capacity(order_ids.len());

    // fire off N children in parallel
    for id in order_ids {
        let fut = ctx.run(enrich_one, id).spawn().await?;
        futures.push(fut);
    }

    // await all; order preserved
    let mut results = Vec::with_capacity(futures.len());
    for f in futures {
        results.push(f.await?);
    }

    Ok(results)
}

#[resonate::function]
async fn enrich_one(order_id: String) -> Result<String> {
    Ok(format!("enriched-{}", order_id))
}
```

Each `enrich_one` is a durable promise. If the parent crashes mid-loop, the children are stored server-side; on parent replay, `f.await?` lookups hit the stored results.

## Parallel fan-out across workers

```rust
#[resonate::function]
async fn parallel_enrich(ctx: &Context, ids: Vec<String>) -> Result<Vec<String>> {
    let mut futures = Vec::with_capacity(ids.len());
    for id in ids {
        let fut = ctx
            .rpc::<String>("enrich_one", id)
            .target("poll://any@enrichment-workers")
            .spawn()
            .await?;
        futures.push(fut);
    }
    let mut results = Vec::with_capacity(futures.len());
    for f in futures {
        results.push(f.await?);
    }
    Ok(results)
}
```

The Resonate server handles fair dispatch across any process polling `enrichment-workers`.

## Recursive fan-out

A durable function calling itself for tree traversal:

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct CrawlInput {
    url: String,
    depth: u32,
}

#[derive(serde::Serialize, serde::Deserialize)]
struct CrawlOutput {
    url: String,
    depth: u32,
    links: Vec<String>,
    children: Vec<CrawlOutput>,
}

#[resonate::function]
async fn crawl(ctx: &Context, input: CrawlInput) -> Result<CrawlOutput> {
    let links = ctx.run(fetch_links, input.url.clone()).await?;

    if input.depth == 0 {
        return Ok(CrawlOutput {
            url: input.url,
            depth: 0,
            links,
            children: vec![],
        });
    }

    // fan out to each link, one depth deeper
    let mut futures = Vec::with_capacity(links.len());
    for link in &links {
        let sub = CrawlInput { url: link.clone(), depth: input.depth - 1 };
        let fut = ctx.rpc::<CrawlOutput>("crawl", sub).spawn().await?;
        futures.push(fut);
    }

    let mut children = Vec::with_capacity(futures.len());
    for f in futures {
        children.push(f.await?);
    }

    Ok(CrawlOutput { url: input.url, depth: input.depth, links, children })
}

#[resonate::function]
async fn fetch_links(url: String) -> Result<Vec<String>> {
    // ...
    Ok(vec![])
}
```

## Bounded parallelism

Process in chunks of K to cap in-flight invocations:

```rust
#[resonate::function]
async fn process_bounded(
    ctx: &Context,
    (items, concurrency): (Vec<String>, usize),
) -> Result<Vec<String>> {
    let mut results = Vec::with_capacity(items.len());

    for chunk in items.chunks(concurrency) {
        let mut futures = Vec::with_capacity(chunk.len());
        for item in chunk {
            let fut = ctx.run(process_one, item.clone()).spawn().await?;
            futures.push(fut);
        }
        for f in futures {
            results.push(f.await?);
        }
    }

    Ok(results)
}

#[resonate::function]
async fn process_one(item: String) -> Result<String> {
    Ok(format!("processed-{}", item))
}
```

Each chunk of up to `concurrency` runs in parallel; the next chunk only starts after the previous fully settles.

## Partial-failure tolerance

By default, a `?` on `.await?` bubbles the first child error up. To tolerate individual failures:

```rust
#[resonate::function]
async fn enrich_tolerant(ctx: &Context, ids: Vec<String>) -> Result<Vec<std::result::Result<String, String>>> {
    let mut futures = Vec::with_capacity(ids.len());
    for id in ids {
        let fut = ctx.run(enrich_one, id).spawn().await?;
        futures.push(fut);
    }
    let mut out = Vec::with_capacity(futures.len());
    for f in futures {
        match f.await {
            Ok(v) => out.push(Ok(v)),
            Err(e) => out.push(Err(e.to_string())),
        }
    }
    Ok(out)
}
```

(Note the return type uses `std::result::Result` to disambiguate from Resonate's aliased `Result`.)

## Distinct Rust idioms

- **`.spawn().await?`** — double-await is intentional: first await starts the spawn; second await on the returned `DurableFuture` gets the actual `T`
- **`Vec::with_capacity(n)`** — idiomatic capacity pre-allocation for known-size collections
- **`.chunks(n)` slice method** — bounded parallelism cleaner than iterator fiddling
- **Turbofish `ctx.rpc::<T>(...)`** — needed when the return type isn't inferable from context
- **`.map_err(|e| e.to_string())`** for error-type narrowing in tolerant-fan-out cases
- **`Clone`** on input types — borrowed slices (`&[String]`) can't cross `ctx.run` boundaries; own the data

## Avoid

- Unbounded recursion without a depth counter — the promise tree grows; Rust's default stack is fine but the server-side promise graph isn't free
- Holding borrowed references across `.await?` — `.spawn()` and `.await?` mean owned data only

## Related skills

- `resonate-basic-durable-world-usage-rust` — `ctx.run`, `ctx.rpc`, `.spawn()`, builder options
- `resonate-saga-pattern-rust` — fan-out inside or alongside sagas
- `durable-execution` — foundational replay semantics
- `resonate-recursive-fan-out-pattern-typescript` / `-python` — sibling SDKs
