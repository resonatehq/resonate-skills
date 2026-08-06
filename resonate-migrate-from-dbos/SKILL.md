---
name: resonate-migrate-from-dbos
description: >-
  Port a DBOS application to Resonate, pattern by pattern. Use when migrating
  DBOS workflows/steps, durable queues, send/recv and setEvent/getEvent
  communication, durable sleep, scheduled (cron) workflows, child workflows, or
  saga-style compensation to the Resonate SDK. Maps canonical dbos-inc demo and
  SDK examples to their Resonate equivalents across TypeScript, Python, Rust, and
  Go, with per-SDK API notes and honest coverage gaps. Foundational skill —
  delegates to the per-SDK pattern skills for idiomatic target code.
license: Apache-2.0
---

# Migrate from DBOS to Resonate

A pattern-by-pattern playbook for porting a DBOS application to Resonate.
Identify which DBOS construct each piece of the source uses, apply the matching
transform, and reach for the linked per-SDK skill for idiomatic target code.

## Ground rules

- **Never invent DBOS code.** Quote it from the user's source or a named
  `dbos-inc/dbos-demo-apps` / `dbos-transact-*` file. If you can't find it, say so.
- **DBOS has SDKs for TypeScript, Python, Go, and Java — but no Rust SDK.** Don't
  emit DBOS Rust. When migrating to Resonate Rust, map from the DBOS Python or
  TypeScript idiom.
- **Resonate has no `@DBOS.workflow`/`@DBOS.step` split.** A step is just a
  function made durable by `ctx.run`. Don't invent decorators or a step type.
- **A DBOS workflow must be deterministic** — side effects live in steps. The same
  discipline carries to Resonate: do side-effecting work inside `ctx.run` so it is
  checkpointed, not re-executed on replay.
- **State coverage honestly.** A missing example ≠ impossible; it means no worked
  reference exists yet.
- **Verify APIs against the pinned version before emitting** — both the DBOS and
  Resonate SDK surfaces drift between versions.

## Pinned SDK versions (latest released at time of writing)

| SDK | Resonate (target) | DBOS (source) |
|---|---|---|
| TypeScript | `@resonatehq/sdk` v0.11.4 (npm) | `@dbos-inc/dbos-sdk` v4.19.8 (npm) |
| Python | `resonate-sdk` v0.6.7 (PyPI) | `dbos` v2.23.0 (PyPI) |
| Rust | `resonate-sdk` v0.6.0 (crates.io) | — (no DBOS Rust SDK) |
| Go | pre-release, tracks `main` | `dbos-transact-golang` v0.17.0 |

## Core mappings (apply everywhere)

| DBOS | Resonate |
|---|---|
| `@DBOS.workflow()` / `DBOS.registerWorkflow(fn)` / `dbos.RegisterWorkflow(ctx, fn)` | a registered function (`@resonate.register`, `resonate.register("name", fn)`, `#[resonate::function]`, `resonate.Register(r, "name", fn)`) |
| `@DBOS.step()` / `DBOS.runStep(fn)` / `dbos.RunAsStep(ctx, fn)` | a plain function invoked via `ctx.run(fn, args)` |
| `DBOS.start_workflow(fn, ...)` / `DBOS.startWorkflow(fn)(...)` / `dbos.RunWorkflow(ctx, fn, args)` (child workflow) | `ctx.rpc("name", args)` or `ctx.run(fn, args)` — invoke by registered name; recursion is trivial |
| durable queue: `DBOS.register_queue` + `DBOS.enqueue_workflow(...)` then `handle.get_result()` (fan-out) | start each non-blocking (`ctx.beginRun` / `ctx.rfi` / `.spawn()` / `ctx.RPC`), then await each |
| `DBOS.recv(topic, timeout)` (block for a message) | one latent durable promise: `p = ctx.promise()` then await `p` |
| `DBOS.send(destId, msg, topic)` (deliver from outside) | `resonate.promises.resolve(id, value)` (HTTP-addressable from anywhere) |
| `DBOS.set_event(key, value)` / `DBOS.get_event(wfId, key)` (publish/read status) | the promise's own resolved value is the status; read it by id |
| `DBOS.sleep(...)` | `ctx.sleep(duration)` |
| `@DBOS.scheduled(cron)` / `DBOS.create_schedule(...)` / `dbos.WithSchedule(cron)` | `resonate.schedule(id, cron, fn)` (ts/rust) or `resonate.schedules.create(...)` (py) |
| try/except + explicit undo step (no saga DSL) | inline `ctx.run(undo, …)` in the error branch, guarded by what completed |
| `systemDatabaseUrl` Postgres (or Python SQLite default) | nothing — the Worker runs in-memory until you connect a Resonate Server |

---

## Pattern: Workflow + step

- **DETECT:** `@DBOS.workflow()` + `@DBOS.step()` (py), `DBOS.registerWorkflow` +
  `DBOS.runStep` / `@DBOS.workflow()`/`@DBOS.step()` static-method decorators (ts),
  `dbos.RegisterWorkflow` + `dbos.RunAsStep` (go).
- **TRANSFORM:** Register one function. Turn each step into a plain function called
  via `ctx.run`. Drop the decorator split and the `systemDatabaseUrl` config.
- **DBOS SOURCE:** `dbos-demo-apps/{python/dbos-app-starter,typescript/dbos-node-starter,golang/dbos-go-starter}`.
- **RESONATE TARGET:** `example-hello-world-{ts,py,rs,go}`.
- **RELATED SKILL:** `resonate-basic-durable-world-usage-{typescript,python,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ✅ (DBOS has no Rust source — map from py/ts).

## Pattern: Child workflows (composition + recursion)

- **DETECT:** `DBOS.start_workflow(fn, ...)` (py), `DBOS.startWorkflow(fn)(...)` (ts),
  `dbos.RunWorkflow(ctx, fn, args)` called inside a workflow (go).
- **TRANSFORM:** Replace the child-workflow call with a call to a registered
  function by name (`ctx.rpc("name", args)` / `ctx.run(fn, args)`). A function can
  recurse on itself — Resonate composes functions indefinitely. Optionally pin
  child ids with `.options(id=…)`.
- **DBOS SOURCE:** `dbos-demo-apps/{python,golang}/widget-store` (`start_workflow`/`RunWorkflow`
  of the dispatch workflow). DBOS does not ship a recursive example.
- **RESONATE TARGET:** `example-recursive-factorial-{ts,py,rs,go}`.
- **RELATED SKILL:** `resonate-recursive-fan-out-pattern-{typescript,python,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ✅.

## Pattern: Fan-out / fan-in (durable queues → parallel + join)

- **DETECT:** `DBOS.register_queue(...)` + a loop of `DBOS.enqueue_workflow(queue, fn, x)`
  then `handle.get_result()` (py), `DBOS.startWorkflow(fn, { queueName })(x)` then
  `h.getResult()` (ts), `dbos.NewWorkflowQueue` + `dbos.RunWorkflow(..., dbos.WithQueue(q.Name))`
  then `handle.GetResult()` (go).
- **TRANSFORM:** Start each unit non-blocking — `ctx.beginRun` (ts) / `ctx.rfi` (py)
  / `ctx.run(...).spawn()` (rs) / `ctx.RPC` (go); each returns a future immediately.
  Then await each future. Start ALL before awaiting ANY, or the work serializes.
- **NOTE:** DBOS queues also provide concurrency limits, rate limits, priority, and
  debouncing. Resonate's fan-out primitive is parallelism + join, not a managed
  queue — if the source relies on per-queue flow control, plan how to reproduce it
  (e.g. a worker `group` plus application-level limiting) and say so.
- **DBOS SOURCE:** DBOS docs Python queue tutorial + TypeScript programming guide
  (the `process_tasks` enqueue-N-then-`get_result` fan-out), `dbos-transact-golang`
  README (10-task fan-out). (`dbos-demo-apps/python/queue-patterns` shows fair-queue /
  rate-limit / debounce, not a plain fan-out.)
- **RESONATE TARGET:** `example-fan-out-fan-in-{ts,py,rs,go}`.
- **RELATED SKILL:** `resonate-recursive-fan-out-pattern-{typescript,python,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ✅.

## Pattern: Durable sleep

- **DETECT:** `DBOS.sleep(seconds)` (py), `DBOS.sleep(ms)` (ts), `dbos.Sleep(ctx, duration)` (go).
- **TRANSFORM:** Replace with `ctx.sleep(duration)`. **Mind the units — they match
  DBOS almost exactly:**
  - TypeScript: **milliseconds** (`ctx.sleep(ms)`). DBOS `DBOS.sleep` is also ms.
  - Python: **seconds** as float (`ctx.sleep(secs)`). DBOS `DBOS.sleep` is also seconds.
  - Go: `time.Duration` (`ctx.Sleep(d)` then `f.Await(nil)`). DBOS `dbos.Sleep(ctx, d)` is also `time.Duration`.
  - Rust: `std::time::Duration` (`ctx.sleep(Duration::from_secs(n)).await?`).
- **DBOS SOURCE:** `dbos-demo-apps/{python,golang}/widget-store` (`DBOS.sleep(1)` /
  `dbos.Sleep(ctx, time.Second)`), `dbos-demo-apps/typescript/widget-store/src/shop.ts`
  (`DBOS.sleep(1000)`).
- **RESONATE TARGET:** `example-durable-sleep-{ts,py,rs,go}`.
- **RELATED SKILL:** `resonate-durable-sleep-scheduled-work-{typescript,rust,go}` (no
  Python variant yet — for Python durable-sleep, delegate to `resonate-basic-durable-world-usage-python`).
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ✅.

## Pattern: Scheduled (cron) workflows

- **DETECT:** `@DBOS.scheduled(cron)` decorator over `@DBOS.workflow()` with a
  `(scheduled, actual)` datetime signature (py/ts), `DBOS.create_schedule(...)` /
  `apply_schedules(...)` (py/ts), `dbos.WithSchedule(cron)` registration option (go).
- **TRANSFORM:** Register a schedule against the Resonate Server: `resonate.schedule(id, cron, fn, ...args)`
  (ts/rust) or `resonate.schedules.create(id=…, cron=…, promise_id=…, …)` (py). The
  server fires the promise on the cron; the worker runs the registered function.
- **MIND THE CRON FIELDS:** DBOS uses a **6-field** cron with a leading seconds component
  (`* * * * * *` = every second). Resonate takes a **standard 5-field** cron (`* * * * *` =
  every minute). Drop the leading seconds field when migrating, or a `* * * * * *` copied
  verbatim silently fires far more often than intended.
- **DBOS SOURCE:** `dbos-transact-py/tests/test_scheduler_decorator.py`,
  `dbos-transact-ts/tests/scheduler_decorator.test.ts`, `dbos-transact-golang` README.
- **RESONATE TARGET:** `example-schedule-{ts,py,rs}`.
- **RELATED SKILL:** `resonate-durable-sleep-scheduled-work-{typescript,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ⚠️ (no `example-schedule-go` yet — the Go
  schedules API is new and tracks `main`; map by analogy).

## Pattern: Communication → durable promises (human-in-the-loop)

- **DETECT:** `DBOS.recv(topic, timeout)` + `DBOS.send(destId, msg, topic)` and/or
  `DBOS.set_event(key, value)` + `DBOS.get_event(wfId, key)` (py/ts/go; Go uses
  generic forms `dbos.Recv[T]` / `dbos.Send` / `dbos.SetEvent` / `dbos.GetEvent[T]`).
- **TRANSFORM:** Replace the `recv`/`set_event` pair with a single latent durable
  promise (`p = ctx.promise()`; await `p`). Surface `p.id` to whoever will resolve it.
  Replace the external `DBOS.send(...)` with `resonate.promises.resolve(id, value)`.
  The promise's resolved value is the status — you don't need a separate
  `get_event` channel.
- **RESOLVE API — verify against pinned version:**
  - ts (0.11.4): the `data` field is base64-encoded JSON —
    `const data = Buffer.from(JSON.stringify(v), "utf8").toString("base64"); resonate.promises.resolve(id, { data })`
    (the codec base64-decodes `data`, so a raw `JSON.stringify(v)` round-trips to garbage)
  - py (0.6.7): `resonate.promises.resolve(id=…, ikey=…)` — the
    `example-human-in-the-loop-py` gateway resolves by id with an idempotency key,
    not a data payload; confirm the kwargs against the example before emitting.
  - rs (0.6.0): `resonate.promises.resolve(&id, Value::from_serializable(v)?)`
    (the example repo may use `json!(v)` — verify it compiles against your released crate version; use the `Value` form if not)
  - go (main): **no high-level resolve yet** — use the CLI
    `resonate promises resolve <id> --value '{"headers":{},"data":"…"}'` or the
    lower-level sender/promise-settle path.
- **DBOS SOURCE:** `dbos-demo-apps/{python,typescript,golang}/widget-store`,
  `dbos-demo-apps/python/agent-inbox` (richer HITL with `recv` + `set_event`).
- **RESONATE TARGET:** `example-human-in-the-loop-{ts,py,rs,go}`.
- **RELATED SKILL:** `resonate-human-in-the-loop-pattern-{typescript,python,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ✅ (Go resolve is lower-level).

## Pattern: Saga / compensation

- **DETECT:** a DBOS workflow that, on a failed/declined step, calls an explicit
  undo step (e.g. `undo_reserve_inventory()` / `undoSubtractInventory()` /
  `undoReserveInventory`) in the failure branch. DBOS has no saga DSL.
- **TRANSFORM:** Run each step with `ctx.run`. On failure, run the undo inline in
  the `catch`/error branch, guarded by which steps actually completed. Make
  compensations idempotent. Same shape as DBOS — no compensation stack, no helper.
- **DBOS SOURCE:** `dbos-demo-apps/{python,typescript,golang}/widget-store`
  (checkout workflow), `dbos-demo-apps/python/reliable-refunds-langchain`.
- **RESONATE TARGET:** `example-saga-booking-ts`, `example-money-transfer-{py,rs}`.
- **RELATED SKILL:** `resonate-saga-pattern-{typescript,python,rust,go}`.
- **COVERAGE:** ts ✅ py ✅ rs ✅ go ⚠️ (no Go example yet — map by analogy).

---

## What does not map cleanly (do not fabricate)

- **Exactly-once Kafka consumers** (`@DBOS.kafka_consumer`) are a DBOS built-in with
  no direct Resonate example. The closest Resonate shape is a worker that creates a
  promise keyed by the message's idempotency key, but there is no worked sample — say so.
- **Durable queue flow control** (concurrency/rate-limit/priority/debounce) is a DBOS
  queue feature. Resonate fan-out is parallelism + join only; reproduce limits at the
  application or worker-group level and flag the gap.
- **Built-in observability dashboard + workflow-management APIs** (`list_workflows`,
  `cancel`, `resume`, `fork`) are DBOS features without a 1:1 Resonate analog.

## Coverage gaps (Resonate examples that don't exist yet)

- Scheduled work: no `example-schedule-go`.
- Saga: no Go example (`example-money-transfer-go` / `example-saga-booking-go`).
- DBOS has no Rust SDK, so all Rust migrations map from the DBOS Python/TypeScript idiom.

## Source of truth

- Resonate examples: https://github.com/resonatehq-examples
- DBOS demos + SDKs: https://github.com/dbos-inc (`dbos-demo-apps`, `dbos-transact-{py,ts,golang}`)
- DBOS docs: https://docs.dbos.dev
- Side-by-side guide: https://docs.resonatehq.io/evaluate/coming-from/dbos
- Concepts first: see the `durable-execution` and `resonate-philosophy` skills.
