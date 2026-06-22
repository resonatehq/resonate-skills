---
name: resonate-recursive-fan-out-pattern-python
description: Implement recursive fan-out in Python for parallel workflow execution — spawn N sub-workflows from a parent, optionally recurse deeper, await results, handle partial failure. Use when processing a tree, batch, or crawl where the work shape is dynamic and each child is independently durable.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern — Python

## Overview

Recursive fan-out is when a durable function spawns child invocations (either of itself or of siblings), optionally waits for them in parallel, and possibly continues recursing. Each child is its own Resonate promise; if a worker crashes, each child resumes independently.

The pattern is expressed in Python by launching multiple `ctx.run(...)` or `ctx.rpc(...)` calls before awaiting them — collect the futures first, then `await` them. This is different from TS's map-over-promises shape only in syntax; the semantics are identical.

## When to use

- Batch processing where items are independent
- Web crawling / tree traversal with dynamic depth
- Map-reduce style workflows
- Any fan-out where each leaf is a discrete, retryable unit of work

**Don't use** for parallel I/O within a single step (use async clients directly inside a `ctx.run` envelope) or for a tight inner loop (overhead of a promise per item dominates).

## Parallel fan-out in the same process

Launch children without blocking; collect futures; await them:

```python
from __future__ import annotations
import asyncio, os, time
from typing import TYPE_CHECKING
from resonate.resonate import Resonate

if TYPE_CHECKING:
    from resonate.context import Context

r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))

async def enrich_batch(ctx: Context, order_ids: list[str]) -> list[dict]:
    # Launch all children (returns futures immediately)
    futures = [ctx.run(enrich_one, oid) for oid in order_ids]

    # Await them all; order preserved
    results = [await f for f in futures]
    return results

async def enrich_one(ctx: Context, order_id: str) -> dict:
    # Enrichment logic — this is a leaf
    return {"order_id": order_id, "enriched": True}
```

Each `enrich_one` call is an independent durable promise. If the parent worker crashes after launching children but before awaiting, the children continue; on parent replay, awaiting the future hits the stored promise value.

Note: structured concurrency guarantees the parent cannot settle until all spawned children (even unawaited ones) complete. Explicitly awaiting them gives you the results.

## Parallel fan-out across workers

Use `ctx.rpc` to dispatch children to remote workers:

```python
async def parallel_enrich(ctx: Context, order_ids: list[str]) -> list[dict]:
    futures = [
        ctx.options(target="enrichment-workers").rpc("enrich_one", oid)
        for oid in order_ids
    ]
    return [await f for f in futures]
```

This horizontally scales across any `enrichment-workers` group. Fair queueing is the Resonate server's responsibility; your code doesn't manage worker pools.

## Recursive fan-out

A durable function calling itself via rpc is legal — useful for tree traversal and crawlers:

```python
async def crawl(ctx: Context, url: str, depth: int) -> dict:
    page = await ctx.run(fetch_page, url)

    if depth <= 0:
        return {"url": url, "depth": 0, "links": page["links"]}

    # Fan out to each link, one depth deeper — dispatched by name via server
    futures = [
        ctx.rpc("crawl", link, depth - 1)
        for link in page["links"]
    ]
    children = [await f for f in futures]

    return {"url": url, "depth": depth, "children": children}
```

For very deep or very wide trees, prefer `ctx.detached` (fire-and-forget) so the parent's promise tree doesn't grow unboundedly.

## The canonical SDK example (fibonacci)

The SDK's `examples/fibonacci` shows recursive fan-out in three modes — `rpc`, `run`, and `mix` — where each recursive call goes through the server or stays local:

```python
from resonate.resonate import Resonate

async def fib_rpc(ctx, n: int) -> int:
    if n <= 1:
        return n
    a = await ctx.rpc("fib_rpc", n - 1)
    b = await ctx.rpc("fib_rpc", n - 2)
    return a + b

async def fib_run(ctx, n: int) -> int:
    if n <= 1:
        return n
    a = await ctx.run(fib_run, n - 1)
    b = await ctx.run(fib_run, n - 2)
    return a + b
```

## Bounded parallelism

Fully parallel fan-out. If you need at-most-K in-flight, batch the input:

```python
from itertools import islice

def _chunks(iterable, k):
    it = iter(iterable)
    while True:
        batch = list(islice(it, k))
        if not batch:
            return
        yield batch

async def process_bounded(ctx: Context, items: list[str], concurrency: int = 10) -> list[dict]:
    results = []
    for batch in _chunks(items, concurrency):
        futures = [ctx.run(process_one, item) for item in batch]
        results.extend([await f for f in futures])
    return results
```

Each batch of up to `concurrency` runs in parallel; the next batch only starts after the previous one fully settles.

## Partial failure handling

By default, if any child raises, the parent's `await f` re-raises. To continue on individual failures:

```python
async def enrich_tolerant(ctx: Context, order_ids: list[str]) -> list[dict]:
    futures = [ctx.run(enrich_one, oid) for oid in order_ids]

    results = []
    for f in futures:
        try:
            results.append(await f)
        except Exception as err:
            results.append({"error": str(err)})
    return results
```

Each child's error is caught individually; the parent returns a mixed list of successes and error dicts.

## Idempotency via stable invocation IDs

Fan-out children get deterministic ids automatically (e.g., `{parent_id}.1`, `{parent_id}.2`). If you dispatch via `ctx.rpc` with explicit ids, keep them stable across retries:

```python
async def enrich_batch(ctx: Context, batch_id: str, order_ids: list[str]) -> list[dict]:
    # rpc with explicit ids ensures replay reuses existing promises
    futures = [
        ctx.options(version=1).rpc(f"enrich:{batch_id}:{oid}", oid)
        for oid in order_ids
    ]
    return [await f for f in futures]
```

## Distinct Python idioms

- **Futures launched before awaited:** `futures = [ctx.run(fn, x) for x in items]` then `[await f for f in futures]` — plain list comprehensions. No special syntax needed.
- **`islice` from `itertools`** for bounded parallelism — cleaner than manual indexing.
- **`try/except` per-future** — Python's narrow scoping lets you catch one child's failure without affecting siblings.
- **No `Promise.all` or `Promise.allSettled`** — `[await f for f in futures]` gives all-or-first-error semantics; per-future try/except gives allSettled semantics.
- **Structured concurrency** — unawaited children are still joined by the runtime before the parent resolves; you don't need to track them explicitly to prevent orphans.

## Related skills

- `resonate-basic-durable-world-usage-python` — `ctx.run`, `ctx.rpc`, `ctx.detached`, `ctx.options`
- `resonate-saga-pattern-python` — fan-out inside a saga for parallel forward steps
- `resonate-human-in-the-loop-pattern-python` — fan-out where each child waits for its own human approval
- `durable-execution` — foundational replay semantics
