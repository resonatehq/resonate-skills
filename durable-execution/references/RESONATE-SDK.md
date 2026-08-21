# Resonate SDK — API Reference

TypeScript SDK for the Resonate wire protocol. Zero external dependencies.

**This is the TypeScript SDK reference** (`@resonatehq/sdk`, reference reflects `0.11.4`). Python, Rust, and Go users: see the per-SDK skills (`resonate-basic-ephemeral-world-usage-{python,rust,go}` and `resonate-basic-durable-world-usage-{python,rust,go}`) for idiomatic syntax in your language. For default values across all four SDKs (retry policy numbers, `ttl`, `timeout`, env vars), use the `resonate-defaults` skill instead of re-deriving them here.

**Package:** `@resonatehq/sdk`
**Two engines, one package:**

| Import | Function shape | Style |
|---|---|---|
| `@resonatehq/sdk` | `function*` generator | `yield` each durable step |
| `@resonatehq/sdk/async` | `async function` | `await` each durable step (eager — see [Async Engine](#async-engine-resonatehqsdkasync)) |

Both engines share the same server, wire protocol, `Promises`/`Schedules` sub-clients, and retry-policy classes. Pick one per function; a codebase can mix generator- and async-registered functions against the same server.

**Primary exports (`@resonatehq/sdk`):** `Resonate`, `type Context`, `type ResonateFunc`, `type ResonateHandle`, `Constant`, `Exponential`, `Linear`, `Never`, `type RetryPolicy`, `type Func`, `OptionsBuilder`, `Registry`. Also re-exported for advanced configuration: `HttpNetwork`, `PollMessageSource`, `LocalNetwork`, `type Network`, `Codec`, `ConsoleLogger`, `type Logger`, `NoopEncryptor`, `type Encryptor`, `WallClock`, `AsyncHeartbeat`, `NoopHeartbeat`, `ResonateTimeoutException`.

**Async-engine exports (`@resonatehq/sdk/async`):** `Resonate`, `type ResonateFunc`, `type ResonateHandle`, `type ResonateSchedule`, `type Context`, `type Info`, `type AnyFunc`, `DurablePromise`, `type DetachedHandle`, `Core`. Retry policies (`Constant`, `Exponential`, `Linear`, `Never`, `type RetryPolicy`) are re-exported unchanged from the root package.

---

## Generator Engine (`@resonatehq/sdk`)

### Resonate Class

#### Constructor

```typescript
import { Resonate, type Context } from "@resonatehq/sdk";

const resonate = new Resonate({
  url: "http://localhost:8001",   // Server URL. Falls back to RESONATE_URL, then "http://localhost:8001".
  group: "default",               // Worker group name.
  pid: "worker-1",                // Process identifier. Defaults to a random UUID.
  ttl: 60_000,                    // Task lease TTL — MILLISECONDS, default 60_000 (60s).
  token: "eyJhbG...",             // Bearer token. Falls back to RESONATE_TOKEN.
  timeout: 10_000,                // HTTP request timeout. Falls back to RESONATE_TIMEOUT, then 10_000ms.
  verbose: false,                 // Shorthand for logLevel: "debug".
  logLevel: "warn",               // Takes precedence over verbose.
  prefix: "",                     // ID prefix applied to generated ids. Falls back to RESONATE_PREFIX.
});
```

> The constructor's own JSDoc comment calls `ttl` "seconds" and shows its default as `1 * util.MIN`. `util.MIN` is `60 * util.SEC` where `util.SEC = 1000` — i.e. the computed default is `60_000`, and every consumer of `ttl` (the task lease, `AsyncHeartbeat`'s `delay = ttl / 2`) treats it as **milliseconds**. Pass `ttl` in ms; the JSDoc's unit label is wrong, not the code.

#### Configuration Reference

| Option | Env var fallback | Default | Description |
|---|---|---|---|
| `url` | `RESONATE_URL` | `"http://localhost:8001"` | Server base URL. If neither resolves, the client uses a local in-memory network. |
| `group` | — | `"default"` | Worker group name. |
| `pid` | — | random UUID | Process identifier. |
| `ttl` | — | `60_000` (ms) | Task lease TTL. |
| `token` | `RESONATE_TOKEN` | none | Bearer token for HTTP auth. |
| `timeout` | `RESONATE_TIMEOUT` | `10_000` (ms) | HTTP request timeout. |
| `verbose` | — | `false` | Shorthand for `logLevel: "debug"`. |
| `logLevel` | — | `"warn"` | `ConsoleLogger` level; overrides `verbose`. |
| `logger` | — | `ConsoleLogger` | Custom `Logger` implementation. |
| `encryptor` | — | `NoopEncryptor` | Payload encryptor. |
| `network` | — | undefined | Custom `Network` implementation (bypasses `url`). |
| `prefix` | `RESONATE_PREFIX` | `""` | Prefix applied to every generated id. |

There is no `auth`, `processId`, or `heartbeatInterval` field. Heartbeat cadence is derived internally as `ttl / 2` and is not separately configurable.

#### Methods

**`register(name, func, options?)`** / **`register(func, options?)`** — Registers a function (by explicit name, or inferred from `func.name`) and returns a `ResonateFunc<F>` handle.

```typescript
const processOrder = resonate.register("processOrder", function* (ctx: Context, sku: string) {
  const price = yield* ctx.run(lookupPrice, sku);
  return price;
}, { version: 2 });  // optional version, default 1

// The handle carries convenience methods bound to this function:
const total = await processOrder.run("order-123", "SKU-42");
```

**`run<T>(id, funcOrName, ...args): Promise<T>`** — Dispatches a workflow and resolves with its **value**. `id` is a required, explicit first argument (not an options field, and not auto-generated). A function reference must already be registered; a string name dispatches by lookup. Mirrors `ctx.run`'s "call" semantics (see [Yielding: call vs. invoke](#yielding-call-vs-invoke)) — it waits for the result.

```typescript
const result = await resonate.run("order-123", processOrder, "SKU-42");
// or, by registered name:
const result = await resonate.run("order-123", "processOrder", "SKU-42");
```

**`beginRun<T>(id, funcOrName, ...args): Promise<ResonateHandle<T>>`** — Begins a workflow and resolves immediately with a handle, without waiting for completion.

**`rpc<T>(id, funcOrName, ...args): Promise<T>`** / **`beginRpc<T>(id, funcOrName, ...args): Promise<ResonateHandle<T>>`** — Same call/begin split as `run`/`beginRun`, but always dispatches through the resolved `target` group rather than assuming local execution.

**`ResonateHandle<T>`** — `{ id: string; result(): Promise<T>; done(): Promise<boolean> }`. Returned by `beginRun`, `beginRpc`, and `get`.

**`schedule(name, cron, funcOrName, ...args): Promise<ResonateSchedule>`** — Creates a persisted cron schedule; each tick creates a fresh durable promise (id embeds the schedule name and timestamp) and dispatches `funcOrName`. Returns `{ delete(): Promise<void> }`.

```typescript
const schedule = await resonate.schedule("daily-report", "0 0 * * *", generateReport);
await schedule.delete();
```

**`get<T>(id): Promise<ResonateHandle<T>>`** — Returns a handle to an existing durable promise by id.

**`options(opts?): Options`** — Builds a resolved `Options` object by merging `opts` (`tags`, `target`, `timeout`, `version`, `retryPolicy`) with the client's defaults. Useful for inspecting effective configuration or constructing a reusable trailing-argument options object.

**`setDependency(name, obj)`** — Registers a named dependency retrievable inside any function via `ctx.getDependency(name)`.

**`stop(): Promise<void>`** — Shuts down the network transport and message source, stops the heartbeat loop, clears subscriptions. Not usable afterward.

There is no `invoke()` and no `start()` on this class. The client is ready to dispatch and receive as soon as it is constructed.

#### `resonate.promises` — Promises Sub-client

```typescript
resonate.promises.get(id): Promise<PromiseRecord>
resonate.promises.create(id, timeoutAt, { headers?, data?, tags? }?): Promise<PromiseRecord>
resonate.promises.createWithTask(id, timeoutAt, pid, ttl, { headers?, data?, tags? }?): Promise<{ promise: PromiseRecord; task?: TaskRecord }>
resonate.promises.resolve(id, { headers?, data? }?): Promise<PromiseRecord>
resonate.promises.reject(id, { headers?, data? }?): Promise<PromiseRecord>
resonate.promises.cancel(id, { headers?, data? }?): Promise<PromiseRecord>
resonate.promises.registerCallback(awaited, awaiter): Promise<{ promise: PromiseRecord }>
resonate.promises.registerListener(awaited, address): Promise<{ promise: PromiseRecord }>
```

This is the full method set. `settle` exists internally but is **private** — `resolve`/`reject`/`cancel` are the public entry points. **There is no `search` method** on this client; `promise.search` exists only at the wire-protocol level (see [Wire Protocol Essentials](#wire-protocol-essentials)) and has no SDK convenience wrapper.

#### `resonate.schedules` — Schedules Sub-client

```typescript
resonate.schedules.get(id): Promise<ScheduleRecord>
resonate.schedules.create(id, cron, promiseId, promiseTimeout, {
  description?, tags?, promiseHeaders?, promiseData?, promiseTags?,
}?): Promise<ScheduleRecord>
resonate.schedules.delete(id): Promise<undefined>
```

---

### Context API

Every generator function receives a `Context` as its first parameter. `Context` methods return a value that must be `yield`-ed — the engine drives the generator by feeding results back into it.

#### Yielding: call vs. invoke

Every durable operation comes in two forms, and the difference is not cosmetic:

- **Call** (`ctx.run`, `ctx.rpc`, `ctx.sleep`) — `yield` **auto-awaits**. You get the settled value directly on the one `yield`; execution blocks until it's ready.
- **Invoke** (`ctx.beginRun`, `ctx.beginRpc`, `ctx.promise`, `ctx.detached`) — `yield` returns a **`Future<T>`** immediately, without waiting. `yield` the `Future` itself (a second `yield`) to suspend until it settles.

```typescript
class Future<T> {
  readonly id: string;
  get state(): "pending" | "completed";
  get value(): { kind: "value"; value: T } | { kind: "error"; error: any } | undefined;
  getValue(): T;   // throws "Future is not ready" if pending; re-throws a stored error
}
```

Fan-out is `beginRun`/`beginRpc` (or `promise`) started in a loop, collected by yielding each `Future` afterward:

```typescript
function* checkout(ctx: Context, skus: string[]) {
  const futures = [];
  for (const sku of skus) {
    futures.push(yield* ctx.beginRun(priceItem, sku));   // begins, doesn't wait
  }
  const prices = [];
  for (const f of futures) {
    prices.push(yield* f);                               // suspends until this one settles
  }
  return prices;
}
```

#### `yield* ctx.run(fn, ...args, opts?)` — Local Call

```typescript
const price = yield* ctx.run(lookupPrice, sku);
const price = yield* ctx.run("lookupPrice", sku);         // dispatch by registered name
```

Runs within the current task. Child promise id: `{ctx.id}.{seq}` (deterministic, auto-incremented — see [Deterministic Child IDs](#deterministic-child-ids)). On replay, a settled child returns its cached value; the function body does not re-execute.

#### `yield* ctx.beginRun(fn, ...args, opts?)` — Local Invoke

Same dispatch as `ctx.run`, but returns a `Future` immediately (see above) instead of blocking. This is the SDK's fan-out primitive.

#### `yield* ctx.rpc(fn|name, ...args, opts?)` — Remote Call

```typescript
const receipt = yield* ctx.rpc(chargeCard, [orderId, amount], {
  target: "payment-workers",     // plain group name — the SDK resolves it to poll://any@payment-workers
  timeout: 30_000,
  version: 2,
});
```

Creates a child promise tagged `resonate:target`, dispatched to any worker in the resolved group. Suspends until settled (auto-await, like `ctx.run`).

#### `yield* ctx.beginRpc(fn|name, ...args, opts?)` — Remote Invoke

Same dispatch as `ctx.rpc`, invoke-style — returns a `Future` immediately.

#### `yield* ctx.detached(fn|name, ...args, opts?)` — Spawn a Fresh Root

```typescript
function* playGame(ctx: Context, n: number) {
  // ...play one game (ctx.run, ctx.sleep, etc.)...
  yield* ctx.detached(playGame, n + 1);   // last statement, then return
}
```

Spawns a workflow as an independent root promise — new `originId`, breaks lineage from the parent. It survives parent completion and replays in its own isolated scope. Invoke-style (`Future` on yield); typically fire-and-forget, so the `Future` usually isn't collected.

Use this to bound replay scope in forever-loop workflows: re-rooting per logical unit of work caps the history each replay walks. **The re-root must happen inside the per-iteration function**, not in a parent loop — `for (;;) yield* ctx.detached(work, n)` still grows the *parent's* history every iteration and reproduces the same bug. Left unbounded, a forever loop's cold-start replay eventually exceeds the acquired-task lease and the server reassigns the task mid-execution (error 1199).

#### `yield* ctx.sleep(ms)` / `yield* ctx.sleep({ for?, until? })` — Durable Sleep

```typescript
yield* ctx.sleep(5000);                          // 5 seconds
yield* ctx.sleep({ until: new Date(deadline) });  // absolute deadline
```

Creates a child promise the server auto-resolves when the timer fires. Call-style — suspends and returns `undefined`. Survives crashes.

#### `yield* ctx.promise(opts?)` — Latent External Promise

```typescript
const approvalFuture = yield* ctx.promise({
  timeout: 86_400_000,           // 24 hours
  tags: { approver: "admin" },
});
const decision = yield* approvalFuture;   // suspends until settled externally
```

Takes only `{ timeout?, data?, tags? }` — **no `id`** (the id is auto-derived as `{ctx.id}.{seq}`, same as every other child). Invoke-style like `beginRun`/`beginRpc`: the first `yield` registers the promise and hands back a `Future`; `yield` that `Future` to actually suspend until someone settles it externally via `resonate.promises.resolve`/`.reject`/`.cancel` or the raw `promise.settle` wire call.

#### `ctx.panic(condition, msg?)` / `ctx.assert(condition, msg?)`

`yield`-ed like the others but produce a `DIE`, not a value. `panic` aborts the pass if `condition` is **true**; `assert` aborts if `condition` is **false**. No promise is settled and the task is released for redelivery — this is an environment-abort signal, not a business-logic error, and a surrounding `try/catch` cannot suppress it (the abort is recorded before the throw).

#### `ctx.getDependency<T>(key): T | undefined`

Reads a value registered via `resonate.setDependency(key, value)`. Not yielded — synchronous.

#### `ctx.options(opts?): Options`

`Partial<Omit<Options, "id">>` — every `Options` field except `id`, since an `Options` built here is typically reused across several calls and `id` must stay unique per invocation. Pass the result as the trailing argument to `run`/`rpc`/`beginRun`/`beginRpc`.

#### `yield* ctx.date.now()` / `yield* ctx.math.random()`

Durable, replay-safe substitutes for `Date.now()`/`Math.random()`. Both are call-style `LFC<number>` — a plain `yield` returns the value directly, and the value is cached like any other step so replay doesn't recompute it. Use these instead of the bare globals for anything that must stay identical across a replay.

#### Read-only Context fields

`ctx.id`, `ctx.originId`, `ctx.prefixId`, `ctx.parentId`, `ctx.branchId`, `ctx.info: { attempt: number; timeout: number; version: number }`.

---

### Durable Function Rules

```typescript
function* myWorkflow(ctx: Context, input: string) {
  const step1 = yield* ctx.run(transform, input);
  const step2 = yield* ctx.rpc("validate", step1);
  yield* ctx.sleep(1000);
  return step2;
}
```

| Rule | Why |
|---|---|
| Use `function*` (generator), not `async function` | The engine drives execution via `yield`; `async function`s go through `@resonatehq/sdk/async` instead |
| First param is always `Context` | Engine injects it |
| `yield` each step | Without `yield`, the step's op object is constructed but never dispatched |
| `return` the final value | The root promise resolves with it |
| `throw` to reject | The root promise rejects with the error |
| No bare `await` inside a generator | Bypasses the durable-execution engine entirely |
| No non-deterministic logic between yields | `Math.random()`/`Date.now()` differ on replay — use `ctx.math.random()`/`ctx.date.now()` |
| Side effects inside `yield* ctx.run(...)` | Bare side effects re-execute on every replay pass |
| Register before dispatching | A function received via SSE that isn't registered throws `REGISTRY_FUNCTION_NOT_REGISTERED` |

### Deterministic Child IDs

Every yielded op produces a child promise with a deterministic id, `{ctx.id}.{seq}`:

```
Root ID: "order-123"
  yield* ctx.run(...)      → "order-123.0"
  yield* ctx.run(...)      → "order-123.1"
  yield* ctx.sleep(...)    → "order-123.2"
  yield* ctx.rpc(...)      → "order-123.3"
```

`seq` increments on every op the context creates and resets to `0` on each fresh execution context. This is what makes replay recognizable: the engine sees child `.0` already settled and skips re-running it. An explicit `id` may still be passed in the trailing options argument to override the auto-derived one.

### Preload Cache

When a task is acquired (or re-acquired after suspend), the server sends a `preload` array of already-settled child promises. The engine caches these and returns cached values for settled steps.

```
First run:  step0 (execute) → step1 (execute) → step2 (crash)
Replay:     step0 (preload) → step1 (preload) → step2 (execute) → step3 (execute)
```

Steps 0 and 1 return their cached results instantly on replay; the function body does not re-execute for them. Execution resumes live from step 2.

---

## Async Engine (`@resonatehq/sdk/async`)

Opt-in entry point that reuses the same network/codec/registry/task plumbing as the generator engine, routed through a separate `Core`. Functions are plain `async function`s; every durable operation is **eager** — calling `ctx.run(...)` begins the work immediately, before you `await` it. Awaiting right away is call-and-wait; holding several unawaited handles and awaiting them later is fan-out. There is no `beginRun`/`beginRpc` here — the eager model makes the begin/call split from the generator engine redundant.

```typescript
import { Resonate, type Context } from "@resonatehq/sdk/async";
```

### Resonate Class

Constructor takes the same options as the generator engine (`url`, `group`, `pid`, `ttl`, `token`, `timeout`, `verbose`, `logLevel`, `logger`, `encryptor`, `network`, `prefix`), with identical defaults.

```typescript
register<F extends AnyFunc>(name: string, func: F, options?: { version?: number }): ResonateFunc<F>;
register<F extends AnyFunc>(func: F, options?: { version?: number }): ResonateFunc<F>;

run<T>(id: string, funcOrName: AnyFunc | string, ...args): Promise<ResonateHandle<T>>;
rpc<T>(id: string, funcOrName: AnyFunc | string, ...args): Promise<ResonateHandle<T>>;

schedule(name: string, cron: string, funcOrName: AnyFunc | string, ...args): Promise<ResonateSchedule>;
get<T>(id: string): Promise<ResonateHandle<T>>;
options(opts?: Partial<Pick<Options, "tags" | "target" | "timeout" | "version" | "retryPolicy">>): Options;
setDependency(name: string, obj: any): void;
stop(): Promise<void>;

readonly promises: Promises;   // identical sub-client, shared with the generator engine
readonly schedules: Schedules;
```

The critical divergence from the generator engine: **`run`/`rpc` resolve to a `ResonateHandle<T>`, not the value.** `ResonateHandle<T>` is the same shape as the generator engine's: `{ id: string; result(): Promise<T>; done(): Promise<boolean> }`. To get the value, `await handle.result()`.

```typescript
const handle = await resonate.run("order-123", processOrder, "SKU-42");
const total = await handle.result();
```

`ResonateFunc<F>` returned by `register()` here only carries `run`, `rpc`, and `options` — no `beginRun`/`beginRpc` (nothing to "begin" separately from — `run`/`rpc` already return a handle).

### Context

```typescript
interface Info {
  readonly id: string;
  readonly parentId: string;
  readonly originId: string;
  readonly prefixId: string;
  readonly branchId: string;
  readonly timeoutAt: number;
  readonly attempt: number;
  readonly version: number;
  readonly func: string;
  getDependency<T>(key: string): T | undefined;
}

interface Context extends Info {
  run<T>(funcOrName: AnyFunc | string, ...args): DurablePromise<T>;
  rpc<T>(funcOrName: AnyFunc | string, ...args): DurablePromise<T>;
  detached<T>(funcOrName: AnyFunc | string, ...args): DurablePromise<DetachedHandle>;
  promise<T>(opts?: { timeout?: number; data?: any; tags?: Record<string, string> }): DurablePromise<T>;
  sleep(ms: number): DurablePromise<void>;
  sleep(opts: { for?: number; until?: Date }): DurablePromise<void>;
  options(opts?: Partial<Omit<Options, "id">>): Options;
  panic(condition: boolean, msg?: string): void;
  assert(condition: boolean, msg?: string): void;
  date: { now(): DurablePromise<number> };
  math: { random(): DurablePromise<number> };
}
```

A "leaf" function types its first parameter as `Info` instead of `Context` — it gets identity and dependency lookup but no durable operations, so it never suspends and always completes in a single pass.

```typescript
async function processOrder(ctx: Context, sku: string): Promise<number> {
  const price = await ctx.run(lookupPrice, sku);
  const receipt = await ctx.rpc(chargeCard, sku, price);
  await ctx.sleep(1000);
  return receipt.total;
}
```

**Fan-out** — call several ops without awaiting, then await them together:

```typescript
async function checkout(ctx: Context, skus: string[]): Promise<number[]> {
  const pending = skus.map((sku) => ctx.run(priceItem, sku));  // each begins immediately
  return Promise.all(pending);                                  // await together
}
```

### `DurablePromise<T>`

The branded return value of every `Context` durable operation. Implements `Promise<T>` (`then`/`catch`/`finally`) and additionally exposes the durable promise id:

```typescript
class DurablePromise<T> implements Promise<T> {
  readonly id: string;   // "" if the op failed before a durable promise was created
}
```

It behaves like a normal JS promise **except** that if the underlying durable promise is still pending remotely, `await`-ing it **hangs** for the current pass rather than resolving or rejecting — the engine ends the pass out of band (suspend), not via the await settling. A real rejection only ever means a genuine failure.

### Async Function Rules

| Rule | Why |
|---|---|
| `async function`, not `function*` | The engine drives execution via `await`; generators go through `@resonatehq/sdk` (root package) instead |
| First param is `Context` (or `Info` for a leaf function) | Engine injects it; `Info`-typed functions never suspend |
| Only `await` `DurablePromise` values returned from `ctx` | Awaiting a timer, plain I/O, or any non-durable promise lets the engine assign durable ids in completion order — this cross-wires ids on replay |
| Call an op to begin it; `await` later to fan out | `ctx.run`/`ctx.rpc`/etc. start the instant you call them, not when you `await` |
| No non-deterministic logic outside `ctx.date.now()`/`ctx.math.random()` | Same replay-determinism requirement as the generator engine |
| Register before dispatching | Same registry, shared with the generator engine |

---

## Options & Retry Policies

### `Options`

```typescript
class Options {
  readonly id: string | undefined;
  readonly tags: Record<string, string>;
  readonly target: string;
  readonly timeout: number;
  readonly version: number;
  readonly retryPolicy: RetryPolicy | undefined;
  readonly nonRetryableErrors: Array<new (...args: any[]) => Error>;
}
```

| Field | Default | Notes |
|---|---|---|
| `id` | `undefined` | No auto-generated UUID — every top-level call requires an explicit `id` as its first positional argument. |
| `tags` | `{}` | |
| `target` | `"default"` | A plain group name. The SDK resolves it to `poll://any@{target}` internally — do not construct the `poll://` URI yourself. |
| `timeout` | `86_400_000` ms (`24 * util.HOUR`) | |
| `version` | `0` | |
| `retryPolicy` | `undefined` at the `Options` layer | Resolved per call: `opts.retryPolicy ?? (isGeneratorFunction(func) ? new Never() : new Exponential())`. A `function*` retries never by default; any other function retries with `Exponential()`. |
| `nonRetryableErrors` | `[]` | |

`resonate.options(opts?)` (top level) only accepts `tags | target | timeout | version | retryPolicy` — it omits `nonRetryableErrors`, unlike `ctx.options(opts?)` which accepts every field except `id`.

### Retry Policies

All four implement `RetryPolicy { next(attempt: number): number | null; encode(): { type: string; data: any } }` — `next` returns the delay in ms before the given attempt, or `null` once retries are exhausted.

```typescript
new Constant({ delay = 1000, maxRetries = Number.MAX_SAFE_INTEGER })
// next(attempt) = delay, while attempt <= maxRetries

new Exponential({ delay = 1000, factor = 2, maxRetries = Number.MAX_SAFE_INTEGER, maxDelay = 30_000 })
// next(attempt) = min(delay * factor ** attempt, maxDelay)

new Linear({ delay = 1000, maxRetries = Number.MAX_SAFE_INTEGER })
// next(attempt) = delay * attempt

new Never()
// next(attempt) = null always — no retries
```

For per-SDK default *values* (including how these compare to Python/Rust/Go), use the `resonate-defaults` skill rather than re-deriving them from this file.

---

## Wire Protocol Essentials

Both engines speak the same wire protocol: a single `POST /` endpoint with a JSON envelope.

```json
{
  "kind": "promise.create",
  "head": { "corrId": "uuid", "version": "" },
  "data": { "id": "my-promise", "timeoutAt": 9999999999999, "tags": {} }
}
```

**Encoding:** payload values are `base64(utf8(JSON))`, via the SDK's `Codec` (and an optional `Encryptor`, default `NoopEncryptor`):

```typescript
function encode(value: unknown): string {
  return btoa(unescape(encodeURIComponent(JSON.stringify(value))));
}
function decode(data: string): unknown {
  return JSON.parse(decodeURIComponent(escape(atob(data))));
}
```

**Promise states:** `pending`, `resolved`, `rejected`, `rejected_canceled`, `rejected_timedout` (all lowercase).

**SSE transport:** workers connect via `GET {url}/poll/{group}/{pid}`; the server pushes `execute` (a task is ready) and `unblock` (an awaited promise settled) messages.

**Request kinds actually on the wire** (`promise.*`, `task.*`):

| Kind | Purpose |
|---|---|
| `promise.get` | Fetch promise state and value |
| `promise.create` | Create a durable promise (optionally with an attached task) |
| `promise.settle` | Resolve/reject/cancel a promise externally |
| `promise.register_callback` | Register one promise to be notified when another settles |
| `promise.register_listener` | Register an address (SSE connection) to be notified on settlement |
| `promise.search` | Query promises by state/tags — **wire-level only, no SDK convenience method** |
| `task.get` / `task.create` | Fetch or create a task |
| `task.acquire` | Claim a task for processing |
| `task.release` | Release a claimed task back to the pool |
| `task.suspend` | Suspend a task pending other promises (used by `yield`/`await` on a pending remote) |
| `task.fulfill` | Complete a task with a settle action |
| `task.fence` | Fenced create-or-settle, guarded by task version — the mechanism that makes lease loss safe: a stale task's fenced op is rejected by the server rather than corrupting state |
| `task.halt` / `task.continue` | Halt/resume a task |
| `task.heartbeat` | Extend one or more task leases |
| `task.search` | Query tasks |

---

## Error Handling

```typescript
resonate.register("resilient", function* (ctx: Context) {
  try {
    const result = yield* ctx.run(riskyStep);
    return result;
  } catch (err) {
    // riskyStep threw — err is the Error it threw
    return "fallback-value";
  }
});
```

- A child step throwing propagates to the parent generator/async function like any JS exception — catchable with `try`/`catch`.
- `ResonateError` (`code`, `type`, `next`, `href`, `retriable`, `serverError?`) is the SDK's own error class — thrown for registry errors (unregistered function, duplicate registration), encoding failures, `ctx.panic`/`ctx.assert` (`PANIC`), and surfaced server errors. `retriable` indicates whether the SDK considers the failure worth retrying.
- `ResonateTimeoutException` is thrown on a client-side timeout.
- `ctx.panic`/`ctx.assert` (generator) and `ctx.panic`/`ctx.assert` (async) abort the current pass without settling any promise — the task is released for redelivery. This is distinct from a thrown business error: nothing is recorded as "failed," the server simply retries the pass.
- Promise timeout (`timeoutAt` elapses) settles the promise `rejected_timedout`; awaiting/yielding it throws at that point, catchable normally.

---

## Migration Notes (pre-0.11 → 0.11.4)

Deltas relevant to code written against an older reference of this SDK — not exhaustive, but covers the API surface that changed shape entirely.

| Then | Now (`0.11.4`) | Notes |
|---|---|---|
| `resonate.invoke(name, args, opts?)` | `resonate.run(id, funcOrName, ...args)` | `id` is now a required, explicit first argument — not an options field, and not auto-generated |
| `resonate.start()` | *(no equivalent)* | The client is ready to dispatch and receive as soon as it's constructed; nothing to start |
| `new Resonate({ auth, processId, heartbeatInterval })` | `new Resonate({ token, pid, ttl })` | Heartbeat cadence is derived internally as `ttl / 2`, not separately configurable |
| Generator engine only | + `@resonatehq/sdk/async` | Async/await engine (`async function`, eager `ctx.run`/`ctx.rpc`, `DurablePromise`) — a parallel entry point, not a replacement |

**Mental model is unchanged either way:** deterministic child ids, replay via the preload cache, one wire protocol underneath. What changed is the surface you call to reach it.

---

## Checklist

**Generator engine:**

- [ ] Is the function a `function*` generator?
- [ ] Does it receive `Context` as its first parameter?
- [ ] Is every durable op `yield`-ed? (`yield* ctx.run(...)`, not just `ctx.run(...)`)
- [ ] For `beginRun`/`beginRpc`/`promise`/`detached`: are you yielding the returned `Future` a second time to collect its value?
- [ ] Are side effects inside `yield* ctx.run(...)` blocks?
- [ ] Is the function registered before anything tries to dispatch it?
- [ ] For external promises: is the id deterministic and known to the settling system?
- [ ] For RPC: is the target group's worker actually running?

**Async engine:**

- [ ] Is the function `async`, with `Context` (or `Info`, for a leaf) as its first parameter?
- [ ] Do you only `await` `DurablePromise` values returned from `ctx` — never a raw timer, fetch, or plain promise?
- [ ] For fan-out: are ops called (started) before they're awaited, not awaited one at a time?
- [ ] Is the function registered before anything tries to dispatch it?
- [ ] Is the Resonate server running and reachable at `url`?
