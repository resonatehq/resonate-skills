---
name: resonate-basic-durable-world-usage-python
description: Core patterns for writing Resonate durable functions in Python — generator syntax with yield, Context API reference (run, begin_run, rpc, begin_rpc, detached, promise, sleep, deterministic time and random, dependency injection), and the determinism rules that make recovery work. Use when writing any function body decorated with @resonate.register.
license: Apache-2.0
---

# Resonate Basic Durable World Usage — Python

## Overview

Durable functions are the payload of Resonate. They are regular Python generators — ordinary `def` functions with `yield` inside — that the SDK treats as a sequence of durable checkpoints. Every `yield` is a save point: if the process crashes, the function resumes from the last successful `yield` on another process or a restart.

This skill covers the Context API (`ctx.*`) that you use inside those functions. The ephemeral-world counterpart (`Resonate` class, registration, top-level invocation) lives in `resonate-basic-ephemeral-world-usage-python`.

## The contract

Every durable function:

1. Is registered with `@resonate.register` or `resonate.register(fn)` in the ephemeral world.
2. Takes `ctx` (the Context) as its first parameter; any other parameters after that are the invocation args.
3. Uses `yield` — not `await`, not `yield from` — to interact with `ctx`. Each yield is a checkpoint.
4. Is deterministic when replayed: no direct `time.time()`, no `random.random()`, no raw network I/O outside `ctx.run`.

Minimal shape:

```py
@resonate.register
def process_order(ctx, order_id: str):
    # step 1 — durable: its result is persisted
    order = yield ctx.run(load_order, order_id)

    # step 2 — durable: runs on a remote worker
    charge = yield ctx.rpc("charge_card", order)

    # step 3 — ordinary Python control flow; free to branch, loop, raise
    if not charge["ok"]:
        raise RuntimeError(f"payment failed: {charge['reason']}")

    return {"order_id": order_id, "status": "paid"}
```

If this function crashes after step 1 but before step 2, it resumes at step 2 on restart — `load_order` is NOT re-run. Its result is read from the promise store.

## Same-process invocation: `ctx.run` and `ctx.begin_run`

`ctx.run` invokes another function in the same process, synchronously. The durable function blocks until it returns, and the result is checkpointed.

```py
@resonate.register
def foo(ctx, arg):
    result = yield ctx.run(bar, arg)
    # ...

def bar(ctx, arg):
    # bar can be a plain function or another durable generator
    return arg.upper()
```

`ctx.begin_run` is the asynchronous form. It returns a promise you can yield later — useful for bounded parallelism:

```py
@resonate.register
def foo(ctx, items):
    # kick off three in parallel
    promises = [yield ctx.begin_run(process, item) for item in items]
    # wait for them all
    results = [(yield p) for p in promises]
    return results
```

`ctx.run` is the default; reach for `ctx.begin_run` only when you need concurrency in a single function body.

## Remote invocation: `ctx.rpc`, `ctx.begin_rpc`, `ctx.detached`

`ctx.rpc` invokes a registered function on a remote worker process. The caller blocks until the remote returns.

```py
@resonate.register
def orchestrator(ctx, batch_id: str):
    # "worker-fn" must be registered in a worker process the caller can reach
    result = yield ctx.rpc("worker_fn", batch_id)
    return result
```

`ctx.begin_rpc` is the async form; it returns a promise for later awaiting. `ctx.detached` is fire-and-forget:

```py
@resonate.register
def notify_then_work(ctx, payload):
    yield ctx.detached("send_slack_notification", payload)
    # the notification promise is not implicitly awaited — the orchestrator continues
    return (yield ctx.rpc("do_the_real_work", payload))
```

Use `detached` when the side effect should survive independently of the parent's success.

## Options

Options configure an individual call. For Context invocations, options **chain after** the call:

```py
from resonate.retry_policies import Exponential, Never

@resonate.register
def foo(ctx, arg):
    result = yield ctx.run(bar, arg).options(
        timeout=30.0,                   # seconds
        retry_policy=Exponential(),     # or Constant(), Linear(), Never()
        target="poll://any@workers",    # only meaningful for rpc / begin_rpc
        tags={"priority": "high"},
        idempotency_key="custom-ikey",
        durable=True,
        version=1,
    )
    return result
```

(Note: the ephemeral-world `resonate.options(...)` is a *prefix* chain; Context `.options(...)` is a *suffix* chain. The symmetry breaks here, but both forms accept the same option dict.)

Retry policies are importable via `from resonate.retry_policies import Exponential, Constant, Linear, Never`.

## External promises: `ctx.promise`

`ctx.promise` creates (or retrieves) a promise the durable function can block on. The promise is resolved or rejected from outside — typically by a webhook, UI, or another process — via `resonate.promises.resolve/reject`.

```py
@resonate.register
def approval_flow(ctx, order_id: str):
    # block until someone resolves this promise
    p = yield ctx.promise(id=f"approval:{order_id}")
    decision = yield p

    if decision.get("approved"):
        yield ctx.run(fulfill, order_id)
    else:
        yield ctx.run(cancel, order_id)
```

Without an ID, one is generated. With an ID, the promise is idempotent — the same ID returns the same promise object, so retries don't create duplicates.

You can also pass data the resolver will see:

```py
p = yield ctx.promise(data={"needs_review_by": "alice@acme.io"})
```

## Determinism: time and randomness

Never call `time.time()` or `random.random()` directly in a durable function — on replay, the values change and the function branches differently than it did originally.

Use the deterministic versions:

```py
@resonate.register
def foo(ctx, arg):
    # returns seconds-since-epoch as a float; stable across replay
    t = yield ctx.time.time()

    # returns a float in [0, 1); stable across replay
    r = yield ctx.random.random()
    # ...
```

These are yielded like any other Context call because their value must be persisted at the checkpoint.

## `ctx.sleep`

`ctx.sleep` sleeps for a duration in **seconds** (float). No upper bound — sleep for days, weeks, whatever. The Resonate server holds the continuation; no process needs to stay alive.

```py
@resonate.register
def daily_digest(ctx, user_id: str):
    while True:
        yield ctx.sleep(24 * 60 * 60)  # 24 hours
        yield ctx.rpc("send_digest", user_id)
```

Unlike TS (which uses milliseconds), **Python uses seconds**. `ctx.sleep(5)` sleeps 5 seconds, not 5 milliseconds.

## Dependencies

`ctx.get_dependency(name)` retrieves a dependency set in the ephemeral world via `resonate.set_dependency(name, obj)`:

```py
@resonate.register
def write_to_db(ctx, record):
    db = ctx.get_dependency("db")
    # treat db like a normal resource — not yielded; dependencies are references, not checkpoints
    yield ctx.run(_insert_row, db, record)
```

Wrap the actual I/O in a helper passed to `ctx.run` so the side effect is checkpointed:

```py
def _insert_row(ctx, db, record):
    db.execute("INSERT INTO orders VALUES (...)", record)
```

## Side effects and the `ctx.run` envelope

Any side effect — HTTP call, DB write, file write, print — belongs inside a function that the durable function reaches via `ctx.run`. This ensures the effect is executed exactly once: the durable function yields, the effect runs, the result is persisted, and on replay the stored result is returned instead of re-executing the effect.

Bad (replays re-execute the side effect):

```py
@resonate.register
def foo(ctx, arg):
    requests.post(WEBHOOK_URL, json=arg)  # ← runs every replay
    return "done"
```

Good (side effect is checkpointed exactly once):

```py
def send_webhook(ctx, arg):
    requests.post(WEBHOOK_URL, json=arg)
    return "sent"

@resonate.register
def foo(ctx, arg):
    yield ctx.run(send_webhook, arg)
    return "done"
```

## What is NOT available in Python (vs TS)

- `ctx.panic()` / `ctx.assert()` — not implemented in the Python SDK. Use standard Python `raise` for invariant violations; Resonate treats raises from durable functions as failures and applies the retry policy.
- `ctx.schedule(...)` (cron-style invocation registration) — currently ephemeral-world only in Python (see the `ScheduleStore` on `resonate.schedules`).

These are not drift in this skill — they're real SDK-feature deltas between TS and Python that a reader should know about.

## Common shapes

**Sequential pipeline with retries:**

```py
@resonate.register
def ingest(ctx, source_url: str):
    data = yield ctx.run(fetch, source_url).options(retry_policy=Exponential())
    parsed = yield ctx.run(parse, data)
    yield ctx.run(persist, parsed)
    return {"count": len(parsed)}
```

**Parallel fan-out within a single function:**

```py
@resonate.register
def enrich_batch(ctx, ids: list[str]):
    promises = [yield ctx.begin_rpc("enrich_one", i) for i in ids]
    return [(yield p) for p in promises]
```

**Recursive workflow (same function calls itself):**

```py
@resonate.register
def crawl(ctx, url: str, depth: int):
    page = yield ctx.run(fetch, url)
    if depth > 0:
        for link in page.links:
            yield ctx.detached("crawl", link, depth - 1)
    return page.id
```

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — the Client API companion
- `resonate-basic-debugging-python` — diagnosing stuck workflows, missing registrations, non-determinism regressions
- `durable-execution` — the foundational concept
- `resonate-philosophy` — "do not build these things" mindset
