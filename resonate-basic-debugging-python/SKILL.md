---
name: resonate-basic-debugging-python
description: Debug and troubleshoot Resonate applications using the Python SDK v0.7.0 and Resonate Server v0.9.x. Use when investigating stuck workflows, non-deterministic replays, registration errors, async/await mistakes, unexpected retries, or server connectivity issues.
license: Apache-2.0
---

# Resonate Debug and Troubleshoot — Python

## Overview

Use this skill to diagnose Resonate runtime issues in Python. The Python SDK v0.7.0 uses `async def` + `await` (not generators); failure modes differ from TypeScript's generator-based v1 API and from the legacy v0.6.x Python SDK. Focus on reproducible symptoms, promise state, registration, and determinism.

For the language-agnostic replay + recovery mental model, read `durable-execution` first.

## Triage flow

1. Classify the failure: import/syntax error, runtime SDK error, server error code, workflow stuck/hanging, or unexpected replay behavior.
2. Confirm Python is ≥ 3.12 and `resonate-sdk>=0.7.0` is installed.
3. Confirm the Resonate server is reachable: `resonate dev` starts it on `localhost:8001` (Rust v0.9.x, single port — no separate store port).
4. Confirm the worker process is running, connected to the right URL and group, and registered for the right functions.
5. Inspect the promise state and call graph for the invocation ID.
6. Validate determinism — no `time.time()`, no `random.random()`, no un-checkpointed side effects in the orchestrator body.

## Version + compat pins

- **Python ≥ 3.12** — the SDK uses recent typing syntax. 3.11 and below won't import.
- **`resonate-sdk>=0.7.0`** — async/await rewrite; incompatible with v0.6.x API.
- **Resonate server v0.9.x** (Rust, single port 8001) — run with `resonate dev` for local development.
- **`resonate-sdk` on PyPI** (not `resonate` — the bare name is a different package).

If `from resonate.resonate import Resonate` fails, you either have the wrong package installed or you're on Python < 3.12.

## Async/await pitfalls (Python-specific)

These are the most common Python-only footguns with the v0.7.0 SDK.

### Forgot `await` before a Context call

```python
async def foo(ctx: Context, arg: str) -> str:
    ctx.run(bar, arg)   # missing await; returns a future the SDK sees but you lose the result
    return "done"
```

The future is tracked by structured concurrency (the runtime will still join it before settling `foo`), but the result is lost and your orchestrator continues without it. Fix:

```python
result = await ctx.run(bar, arg)
```

### Used `yield` instead of `await`

```python
async def foo(ctx: Context, arg: str) -> str:
    result = yield ctx.run(bar, arg)    # wrong — this is generator syntax
```

The v0.7.0 SDK uses `async def` + `await`, not `def` + `yield`. Mixing them produces a `TypeError` or unexpected behavior. Use `await`.

### Made the function a plain `def` instead of `async def`

```python
def foo(ctx: Context, arg: str) -> str:   # missing async
    result = await ctx.run(bar, arg)       # SyntaxError
```

All durable functions must be `async def`. The SDK registers and dispatches them as coroutines.

### Not awaiting `r.promises.*`

```python
r.promises.resolve(approval_id, Value(data={"ok": True}))   # missing await
```

All `r.promises.*` methods are async. Missing `await` means the resolve is never sent. Fix:

```python
await r.promises.resolve(approval_id, Value(data={"ok": True}))
```

### Not calling `await r.stop()` on shutdown

The SDK needs to drain in-flight work cleanly. Always:

```python
try:
    result = await r.run(id, fn, arg).result()
finally:
    await r.stop()
```

## Registration errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `Function 'X' is not registered` | `rpc` target name doesn't match a registered name on any reachable worker | Check `r.register(fn)` is called; ensure the calling name and registered name match (if you used `r.register(fn, name="custom")` the caller must use `"custom"`) |
| `Function version must be greater than zero` | Passing `version=0` explicitly | Default is 1; omit `version` or pass a positive integer |
| `Args unencodeable` | Argument to `run`/`rpc` not JSON-serializable | Use dicts, lists, strings, numbers, bools, None, or types with custom encoders registered |

## Promise-state errors

| Code | Meaning | Typical cause |
|---|---|---|
| 40900 | Promise already exists | Two calls used the same invocation ID; treat as idempotency — the second call returns the first's result. If you wanted a fresh run, change the ID |
| 40400 | Promise not found | Wrong ID, wrong server URL, or promise was purged |
| 40300 | Already resolved | Normal if you're checking idempotency; the value is the first resolution |
| 40301 | Already rejected | Normal; the reason is the first rejection |
| 40303 | Already timed out | Promise's deadline passed; re-create with a new ID if you want to retry |

Inspect from the CLI:

```shell
resonate promises get <promise-id>
resonate tree <invocation-id>
resonate promises search <id-prefix>
```

## "My workflow hangs"

Three common causes, in order of likelihood:

1. **No worker for the target group.** If your `rpc` targets `"backend"` but no process has registered with `group="backend"`, the invocation is queued forever. Check: `resonate promises get <id>` → state stays `pending`; confirm your worker process is running and connected.
2. **Forgot `await`** (see pitfalls above). The orchestrator advanced past a step without waiting for it.
3. **Server not reachable.** Confirm `resonate dev` is running on port 8001; check `RESONATE_URL` env var.

## "My workflow replays things it shouldn't"

Replays are normal — Resonate re-executes the durable function from the top on every resumption, using stored promise values for the awaited checkpoints. Side effects **outside** the `ctx.run` envelope re-execute on every replay.

```python
async def foo(ctx: Context, arg: str) -> str:
    print(f"starting {arg}")              # prints on every replay
    log.info(f"starting {arg}")           # logs on every replay
    result = await ctx.run(work, arg)
    return result
```

Fix: move any side effect into a helper called via `ctx.run`:

```python
async def log_start(ctx: Context, arg: str) -> None:
    log.info(f"starting {arg}")

async def foo(ctx: Context, arg: str) -> str:
    await ctx.run(log_start, arg)         # logged exactly once, persisted
    result = await ctx.run(work, arg)
    return result
```

## Non-determinism regressions

Common non-deterministic code that breaks recovery:

```python
async def bad(ctx: Context, x: str) -> str:
    t = time.time()                        # changes between runs
    r = random.random()                    # changes between runs
    path = os.environ.get("FEATURE")      # changes if env mutates
    if t > 1700000000 or r > 0.5:
        return await ctx.run(branch_a)
    return await ctx.run(branch_b)
```

Each resumption may hit a different branch than the original run. Wrap non-deterministic values in leaf functions via `ctx.run`:

```python
async def _get_time(ctx: Context) -> float:
    return time.time()

async def _get_random(ctx: Context) -> float:
    return random.random()

async def good(ctx: Context, x: str) -> str:
    t = await ctx.run(_get_time)
    r = await ctx.run(_get_random)
    # Now t and r are stable across replays (stored at first execution)
```

## Minimal repro templates

Worker:

```python
# worker.py
from __future__ import annotations
import asyncio, os
from typing import TYPE_CHECKING
from resonate.resonate import Resonate

if TYPE_CHECKING:
    from resonate.context import Context

r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"), group="workers")

async def ping(ctx: Context, name: str) -> str:
    return f"pong {name}"

r.register(ping)

async def main() -> None:
    try:
        await asyncio.Event().wait()    # block forever; SDK polls for work
    finally:
        await r.stop()

asyncio.run(main())
```

Client:

```python
# client.py
from __future__ import annotations
import asyncio, os, time
from resonate.resonate import Resonate

async def main() -> None:
    r = Resonate(url=os.environ.get("RESONATE_URL", "http://localhost:8001"))
    try:
        result = await r.options(target="workers").rpc(
            f"ping-{time.time_ns()}", "ping", "alice"
        ).result()
        print(result)
    finally:
        await r.stop()

asyncio.run(main())
```

If the client hangs, check: worker running? same URL? worker in the right group? `resonate promises get <id>` → what state is it in?

## CLI one-liners

```shell
# local dev server — zero config, single port 8001
resonate dev

# inspect a workflow's tree of promises
resonate tree <invocation-id>

# get a specific promise's state
resonate promises get <id>

# resolve/reject external promises from the CLI (e.g., for HITL testing)
resonate promises resolve <id> --data '{"approve": true}'
resonate promises reject <id> --data '{"reason": "denied"}'

# search by prefix or tags
resonate promises search 'order:*'
```

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — Client APIs; useful when the failure is at the process-entry layer
- `resonate-basic-durable-world-usage-python` — Context APIs; useful when the failure is inside a durable function
- `durable-execution` — foundational; the mental model for recovery + replay
- `resonate-philosophy` — the "do not build these things" list; many debug sessions end up being about patterns that were already warned against here
