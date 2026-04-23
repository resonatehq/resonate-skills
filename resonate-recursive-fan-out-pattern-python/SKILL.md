---
name: resonate-recursive-fan-out-pattern-python
description: Implement recursive fan-out in Python for parallel workflow execution — spawn N sub-workflows from a parent, optionally recurse deeper, await results, handle partial failure. Use when processing a tree, batch, or crawl where the work shape is dynamic and each child is independently durable.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern — Python

## Overview

Recursive fan-out is when a durable function spawns child invocations (either of itself or of siblings), optionally waits for them in parallel, and possibly continues recursing. Each child is its own Resonate promise; if a worker crashes, each child resumes independently.

The pattern is naturally expressed in Python as a list comprehension over `ctx.begin_run` or `ctx.begin_rpc` followed by a yield-to-await loop. This is different from TS's map-over-promises shape only in syntax; the semantics are identical.

## When to use

- Batch processing where items are independent
- Web crawling / tree traversal with dynamic depth
- Map-reduce style workflows
- Any fan-out where each leaf is a discrete, retryable unit of work

**Don't use** for parallel I/O within a single step (use async clients directly inside a `ctx.run` envelope) or for a tight inner loop (overhead of a promise per item dominates).

## Parallel fan-out in the same process

Use `ctx.begin_run` to start children without blocking; collect promises; yield them to await:

```py
from resonate import Resonate

resonate = Resonate()

@resonate.register
def enrich_batch(ctx, order_ids: list[str]):
    # fire off N children in parallel
    promises = [(yield ctx.begin_run(enrich_one, oid)) for oid in order_ids]

    # await all; order preserved
    results = [(yield p) for p in promises]
    return results


def enrich_one(ctx, order_id: str):
    # ... enrichment logic ...
    return {"order_id": order_id, "enriched": True}
```

Each `enrich_one` call is an independent durable promise. If the parent worker crashes after launching children but before awaiting, the children continue; on parent replay, the `yield p` lookups hit the stored promise values.

## Parallel fan-out across workers

Use `ctx.begin_rpc` to dispatch children to remote workers:

```py
@resonate.register
def parallel_enrich(ctx, order_ids: list[str]):
    promises = [
        (yield ctx.begin_rpc("enrich_one", oid).options(target="poll://any@enrichment-workers"))
        for oid in order_ids
    ]
    return [(yield p) for p in promises]
```

This horizontally scales across any `enrichment-workers` group. Fair queueing is the Resonate server's responsibility; your code doesn't manage worker pools.

## Recursive fan-out

A durable function calling itself is legal — useful for tree traversal and crawlers:

```py
@resonate.register
def crawl(ctx, url: str, depth: int):
    page = yield ctx.run(fetch_page, url)

    if depth <= 0:
        return {"url": url, "depth": 0, "links": page.links}

    # fan out to each link, one depth deeper
    promises = [
        (yield ctx.begin_rpc("crawl", link, depth - 1))
        for link in page.links
    ]
    children = [(yield p) for p in promises]

    return {"url": url, "depth": depth, "links": page.links, "children": children}
```

Recursion depth is only bounded by Python's own recursion limit (about 1000 by default) and by the promise graph's storage capacity. For very deep or very wide trees, prefer `ctx.detached` (fire-and-forget) so the parent's promise tree doesn't grow unboundedly.

## Bounded parallelism

`begin_run` + `yield` is fully parallel. If you need at-most-K in-flight, split the input:

```py
from itertools import islice

def _chunks(iterable, k):
    it = iter(iterable)
    while True:
        batch = list(islice(it, k))
        if not batch:
            return
        yield batch

@resonate.register
def process_bounded(ctx, items: list[str], concurrency: int = 10):
    results = []
    for batch in _chunks(items, concurrency):
        promises = [(yield ctx.begin_run(process_one, item)) for item in batch]
        results.extend([(yield p) for p in promises])
    return results
```

Each batch of up to `concurrency` runs in parallel; the next batch only starts after the previous one fully settles.

## Partial failure handling

By default, if any child raises, its promise rejects and the parent's `yield p` re-raises. If you want to continue on individual failures:

```py
@resonate.register
def enrich_tolerant(ctx, order_ids: list[str]):
    promises = [(yield ctx.begin_run(enrich_one, oid)) for oid in order_ids]

    results = []
    for p in promises:
        try:
            results.append((yield p))
        except Exception as err:
            results.append({"error": str(err)})
    return results
```

Each child's error is caught individually; the parent returns a mixed list of successes and error dicts.

## Idempotency via stable invocation IDs

Every recursive or fan-out call should use a stable invocation ID so replays don't create new invocations:

```py
@resonate.register
def enrich_batch(ctx, batch_id: str, order_ids: list[str]):
    # pass the batch + item IDs so the invocation is stable across retries
    promises = [
        (yield ctx.begin_run(enrich_one, oid).options(id=f"enrich:{batch_id}:{oid}"))
        for oid in order_ids
    ]
    return [(yield p) for p in promises]
```

Without a stable ID, a replay creates a fresh child invocation instead of reusing the original's result.

## Distinct Python idioms

- **List comprehension with `yield` inside:** `[(yield ctx.begin_run(...)) for x in items]` — Python-specific expression. TS achieves the same with `map`/`for` loops but no single-line shape.
- **`islice` from `itertools`** for bounded parallelism — cleaner than manual indexing.
- **`try/except` per-promise** — Python's narrow scoping lets you catch one child's failure without affecting siblings, naturally.
- **No `Promise.all` or `Promise.allSettled`** — yielding the list sequentially already gives all-or-first-error semantics; per-promise try/except gives allSettled semantics.

## Related skills

- `resonate-basic-durable-world-usage-python` — `ctx.begin_run`, `ctx.begin_rpc`, `ctx.detached`, options chain
- `resonate-saga-pattern-python` — fan-out inside a saga for parallel forward steps
- `resonate-human-in-the-loop-pattern-python` — fan-out where each child waits for its own human approval
- `durable-execution` — foundational replay semantics
