---
name: resonate-basic-debugging-python
description: Debug and troubleshoot Resonate applications using the Python SDK and Resonate Server. Use when investigating stuck workflows, non-deterministic replays, registration errors, generator mistakes, unexpected retries, or server-compat issues specific to the Python SDK's v0.6.7 legacy-server line.
license: Apache-2.0
---

# Resonate Debug and Troubleshoot — Python

## Overview

Use this skill to diagnose Resonate runtime issues in Python. The Python SDK's failure modes are different from TypeScript's — generator-specific pitfalls, asyncio traceback shapes, server-compat caveats on v0.6.7. Focus on reproducible symptoms, promise state, registration, and determinism.

For the language-agnostic replay + recovery mental model, read `durable-execution` first.

## Triage flow

1. Classify the failure: import/syntax error, runtime SDK error, server error code, workflow stuck/hanging, or unexpected replay behavior.
2. Confirm Python is ≥ 3.12 and `resonate-sdk` is installed at the version your code expects.
3. Check server compat: Python SDK v0.6.7 **requires the legacy Resonate server protocol**. Connecting to a v0.9.x server will appear as a long-poller that never produces messages.
4. Confirm the worker process is running, connected to the right URL, and registered for the right group.
5. Inspect the promise state and call graph for the invocation ID.
6. Validate determinism — no `time.time()`, no `random.random()`, no un-checkpointed side effects.

## Version + compat pins

- **Python ≥ 3.12** — the SDK uses recent typing syntax. 3.11 and below won't import.
- **Python SDK v0.6.7** — legacy-server only. A v0.9.x-compatible Python release is on the roadmap; check release notes before upgrading the server without upgrading the SDK.
- **`resonate-sdk` on PyPI** (not `resonate` — the bare name is a different package).

If `from resonate import Resonate` fails, you either have the wrong package installed or you're on Python < 3.12.

## Generator pitfalls (Python-specific)

These are the most common Python-only footguns.

### Forgot `yield` before a Context call

```py
@resonate.register
def foo(ctx, arg):
    ctx.run(bar, arg)  # ← missing yield; silently returns a promise object the SDK can't see
    return "done"
```

The function returns immediately, no checkpoint is recorded, and `bar` may or may not have run depending on SDK internals. Fix:

```py
result = yield ctx.run(bar, arg)
```

### Used `await` instead of `yield`

```py
@resonate.register
async def foo(ctx, arg):        # ← don't make it async
    result = await ctx.run(...) # ← Resonate Python uses yield, not await
```

Durable functions in the Python SDK are **plain generators**, not coroutines. Use `def` (not `async def`) and `yield` (not `await`). Mixing them produces a `TypeError: object cannot be used in await expression` or a silent no-op depending on where the mismatch is.

### `yield from` instead of `yield`

```py
@resonate.register
def foo(ctx, arg):
    yield from ctx.run(bar, arg)   # ← wrong; ctx.run returns a single RFC, not an iterable
```

`yield from` delegates to a sub-generator; Context calls are not sub-generators. Use `yield` (single value).

### Function doesn't yield at all

```py
@resonate.register
def foo(ctx, arg):
    return do_work(arg)       # ← no yield → not a generator → Resonate treats it as plain
```

If a registered function has no `yield`, Python doesn't treat the body as a generator. Resonate may accept it as a trivial pass-through, but you lose checkpointing — a crash in `do_work` restarts the whole thing.

Add at least one `yield` (or wrap `do_work` as `yield ctx.run(do_work, arg)`) to make the function durable.

## Registration errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `Function 'X' is not registered` | `rpc` target name doesn't match a registered name on any reachable worker | Check `@resonate.register` decorator is applied; ensure the calling name and registered name match (if you used `resonate.register(fn, name="custom")` the caller must use `"custom"`) |
| `Function 'X' (version Y) is already registered` | Duplicate registration in the same process | Register once per process; if you're reloading modules in a dev loop, check hot-reload config |
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

1. **No worker for the target group.** If your `rpc` targets `poll://any@workers` but no process has registered with `group="workers"`, the invocation is queued forever. Check: `resonate promises get <id>` → state stays `pending`; check your worker process is running and connected.
2. **Forgot `yield`** (see Generator pitfalls). The function returned before the Context call executed.
3. **v0.6.7 SDK + v0.9.x server.** Legacy SDK can't read v0.9.x messages. Check both versions.

## "My workflow replays things it shouldn't"

Replays are normal — Resonate re-runs the durable function from the top on every resumption, using stored promise values for the yielded checkpoints. Side effects **outside** the `ctx.run` envelope re-execute on every replay.

```py
@resonate.register
def foo(ctx, arg):
    print(f"starting {arg}")              # ← prints on every replay
    log.info(f"starting {arg}")           # ← logs on every replay
    result = yield ctx.run(work, arg)
    return result
```

Fix: move any side effect into a helper called via `ctx.run`:

```py
def log_start(ctx, arg):
    log.info(f"starting {arg}")

@resonate.register
def foo(ctx, arg):
    yield ctx.run(log_start, arg)         # ← logged exactly once, persisted
    result = yield ctx.run(work, arg)
    return result
```

## Non-determinism regressions

Common non-deterministic code that breaks recovery:

```py
@resonate.register
def bad(ctx, x):
    t = time.time()                       # ← changes between runs
    r = random.random()                   # ← changes between runs
    path = os.environ.get("FEATURE")      # ← changes if env mutates
    if t > 1700000000 or r > 0.5:
        yield ctx.run(branch_a)
    else:
        yield ctx.run(branch_b)
```

Each resumption may hit a different branch than the original run, leading to contradictory checkpoints. Use the deterministic versions:

```py
@resonate.register
def good(ctx, x):
    t = yield ctx.time.time()
    r = yield ctx.random.random()
    # env reads should happen once, at registration time, or be captured as args
```

## Minimal repro templates

Worker:

```py
# worker.py
from resonate import Resonate

resonate = Resonate.remote(host="localhost")

@resonate.register
def ping(ctx, name: str):
    return f"pong {name}"

if __name__ == "__main__":
    import time
    while True:
        time.sleep(60)
```

Client:

```py
# client.py
from resonate import Resonate

resonate = Resonate.remote(host="localhost")
print(resonate.rpc("ping:alice", "ping", "alice"))
```

If the client hangs, check: worker running? same host/port? worker in the right group? `resonate promises get ping:alice` → what state is it in?

## CLI one-liners

```shell
# local dev server — zero config
resonate dev

# inspect a workflow's tree of promises
resonate tree <invocation-id>

# get a specific promise's state
resonate promises get <id>

# resolve/reject external promises from the CLI (e.g., for HITL testing)
resonate promises resolve <id> --data '{"approved": true}'
resonate promises reject <id> --data '{"reason": "denied"}'

# search by prefix or tags
resonate promises search 'order:*'
```

## What is NOT in the Python SDK yet

- `ctx.panic()` / `ctx.assert()` — use standard `raise` for invariant violations
- `.schedule()` on the Resonate class (for cron-style registration) — the `ScheduleStore` at `resonate.schedules` is available but the docs do not yet show a top-level `schedule(...)` method

Don't chase these as "drift in the skill" — they're genuine SDK surface-area gaps between TS and Python, and both surface a documentation or implementation item if an agent needs them.

## Related skills

- `resonate-basic-ephemeral-world-usage-python` — Client APIs; useful when the failure is at the process-entry layer
- `resonate-basic-durable-world-usage-python` — Context APIs; useful when the failure is inside a durable function
- `durable-execution` — foundational; the mental model for recovery + replay
- `resonate-philosophy` — the "do not build these things" list; many debug sessions end up being about patterns that were already warned against here
