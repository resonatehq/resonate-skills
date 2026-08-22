---
name: resonate-server-postgres
description: Run the Resonate protocol inside Postgres itself using resonate-pg — one SQL file of stored procedures, with no server process at all. Covers installing the schema, the pg_cron timer dependency and its silent-failure mode, reaching the protocol through resonate_rpc, starting workflows with resonate.invoke, the resonate_worker grant surface, garbage collection and id idempotency, the TypeScript/Deno-only client constraint, and the open correctness gaps. Use when a user already runs Postgres and wants durable execution without deploying anything beside it.
license: Apache-2.0
---

# Resonate on Postgres

## Overview

[`resonatehq/resonate-pg`](https://github.com/resonatehq/resonate-pg) implements the Resonate protocol as a schema of PL/pgSQL stored procedures. You load one file into a Postgres 16+ database and the database *is* the server — storage, queue, and timer, with `pg_cron` driving all three.

Three things an agent must internalize before writing any code:

1. **It is not the core server on Postgres.** Those are different things and users conflate them constantly. The core server *supports* Postgres: you run a binary and Postgres holds its state. `resonate-pg` *is* Postgres: there is no binary, no process, and no port. If the user wants a Resonate server that stores data in Postgres, send them to the core server instead — it is more mature and every SDK can talk to it.
2. **It is not drop-in, and there is no URL.** There is no HTTP server, so `RESONATE_URL` is meaningless. Workers reach the protocol by calling `resonate.resonate_rpc(jsonb)` over a database connection, which needs a client that speaks SQL.
3. **One client exists, and it targets Deno.** `@resonatehq/supabase` on JSR is the only implementation. Python, Go, Rust, and Java have nothing, and each is tracked as an open issue. Say this before the user commits, not after.

Related: [`resonate-supabase-deployments-typescript`](../resonate-supabase-deployments-typescript/SKILL.md) (deploying workers as Edge Functions against this server), [`resonate-server-nats`](../resonate-server-nats/SKILL.md) and [`resonate-server-scylladb`](../resonate-server-scylladb/SKILL.md) (the other two providers), [`resonate-server-deployment`](../resonate-server-deployment/SKILL.md) (core server on systemd).

**Scope boundary:** this skill covers *the server* — installing it, what it implements, how to operate it. For wiring and deploying a worker against it, use the Supabase deployments skill above.

## When to use this skill

- The user already runs Postgres and does not want to deploy a process beside it.
- The user asks whether Resonate can run on Supabase, or "just in my database."
- The user is operating an existing `resonate-pg` install — timers, retention, grants.

## When NOT to use this skill

- **The user is willing to run a process.** Recommend the core server on Postgres. It is the reference implementation with the deepest track record, and this is not a substitute for it.
- **The user is on Python, Go, Rust, or Java.** No client exists. Do not suggest writing one as a workaround.
- **The user is deploying worker code.** That is the Supabase deployments skill, not this one.
- **The user needs a mature, well-tested implementation.** Be honest — see Gaps. This is the youngest of the three providers.

## Licensing — say this correctly

`resonate-pg` is **Apache 2.0**, like the core server. Unlike `resonate-on-nats` and `resonate-on-scylladb`, there is no BUSL-1.1 restriction and no production-use license to buy. Do not carry the BUSL caveat over from the other two provider skills by pattern-matching.

## Requirements

- **Postgres 16+.**
- **`pg_cron`** — required, and the sole timer driver.
- **`pg_net` or `pgsql_http`** — optional; either enables HTTP push delivery to workers.

## Installing

```bash
psql -d yourdb -c "create extension if not exists pg_cron; create extension if not exists pg_net;"
psql -d yourdb -f resonate.sql
```

Applying the file also registers the timer job (`resonate_process_timeouts`) with `pg_cron`.

### The install's silent-failure mode — check this every time

If `pg_cron` is unavailable, **the install still succeeds.** It raises a `WARNING`, leaves the timer unscheduled, and returns. The result is a database where promises are created and settled correctly while every durable sleep hangs forever and no task timeout ever fires. Nothing surfaces this at runtime.

Verify both, always:

**1. Did the timer actually get scheduled?**

```sql
SELECT jobname, schedule, active FROM cron.job WHERE jobname = 'resonate_process_timeouts';
```

**2. Did the install say so?** Look for the `resonate: pg_cron enabled` notice in the install output.

Scheduling also **degrades quietly**. The installer tries, in order: `cron.schedule_in_database` at `5 seconds`, then plain `cron.schedule` at `5 seconds`, then `cron.schedule` at `* * * * *`. That last fallback works, but timer resolution drops from five seconds to one minute. If a user reports sleeps that wake "about a minute late," check which branch they landed on rather than looking for a bug in the workflow.

## Reaching the protocol

Every protocol action is dispatched by kind through one function:

```sql
SELECT resonate.resonate_rpc('{"kind":"promise.get","head":{},"data":{"id":"invoke:foo"}}');
```

Implemented kinds: `promise.create`, `promise.get`, `promise.settle`, `promise.search`, `promise.register_callback`, `promise.register_listener`; `task.create`, `task.acquire`, `task.release`, `task.heartbeat`, `task.fence`, `task.fulfill`, `task.continue`, `task.suspend`, `task.halt`, `task.get`, `task.search`; `schedule.create`, `schedule.get`, `schedule.delete`, `schedule.search`.

**Search works here.** `promise.search`, `task.search`, and `schedule.search` are all real implementations that return results. This is a genuine advantage over the NATS and ScyllaDB providers, both of which lack search entirely — do not carry their "no search" gap over to this one.

## Starting a workflow

Invocations start from SQL. Delivery is push-based: the database posts outward to the address you supply.

```sql
SELECT resonate.invoke(
  'countdown-1',        -- id
  'countdown',          -- func: the name the worker registered
  '[3]'::jsonb,         -- args
  'https://<host>/countdown',  -- target: where to push
  1,                    -- version  (default 1)
  NULL);                -- timeout  (default: now + 24h)
```

**The 24-hour default timeout is a trap for long workflows.** `timeout` defaults to `now + 86400000` ms. A workflow that sleeps for a week is timed out on day one. Any workflow whose total lifetime can exceed a day must pass an explicit `timeout`, and an agent writing a multi-day durable sleep should set it without being asked.

Push delivery goes through `net.http_post` (`pg_net`) where present, falling back to `pgsql_http`. **A failed push is raised as a `WARNING`, not an error** — the workflow does not fail, and the only trace is the Postgres log. A worker that is unreachable produces a silently stalled workflow, so check the log before debugging the workflow itself.

## Access control

Authentication is Postgres's, and the schema is configured for it rather than left open:

- All table, sequence, and function privileges are revoked from `PUBLIC`, including default privileges on future functions.
- A `resonate_worker` role gets `USAGE` on the schema plus `EXECUTE` on exactly six functions: `resonate_rpc`, `get_schema_version`, `invoke`, `dequeue_execute`, `dequeue_unblock`, `outbox_channel`.
- `resonate_rpc`, `dequeue_execute`, and `dequeue_unblock` are `SECURITY DEFINER`, and every function in the schema has `search_path` pinned to `resonate, pg_temp`.

**Give workers the `resonate_worker` role, never a superuser connection string.** Anyone who can execute `resonate_rpc` can drive the entire protocol — create, settle, and halt anything. The database connection is the security boundary, so treat that connection string the way you would treat a root credential.

## Retention and garbage collection

Completed workflows stay in the database until you delete them. Nothing runs automatically:

```sql
-- daily at 03:00: delete workflows settled more than 7 days ago
select cron.schedule('resonate-gc', '0 3 * * *',
  $$select resonate.gc((extract(epoch from now())*1000 - 7*86400000)::bigint)$$);
```

`gc(p_settled_before bigint, p_limit int DEFAULT 10000)` deletes in bounded batches.

**Idempotency expires with the row.** A workflow id is idempotent only while its promise row exists. Set the GC horizon longer than any window in which the same id might be resubmitted — otherwise a retry that arrives after collection is not deduplicated, it is a second execution. When a user picks a horizon, make them state their maximum retry window first.

## Wiring a worker (pointer, not the procedure)

`@resonatehq/supabase` on JSR wraps `@resonatehq/sdk` and exports `Resonate` and `SupabaseNetwork`. Despite the name it is a `resonate-pg` client, not a Supabase-only one — `SupabaseNetworkConfig` takes any `connectionString`, defaulting to the `SUPABASE_DB_URL` environment variable.

Two properties shape what it can do:

- It implements **only `send`**, not `recv`. There is no long-poll; the worker is invoked per message because delivery arrives by HTTP push from `pg_net`.
- It is published **for Deno**, using `npm:` import specifiers and reading `Deno.env` for the default connection string. Treat Edge-Function-style hosting as the supported path.

It pools with `prepare: false` and `max: 1` — required for compatibility with transaction poolers, which do not support prepared statements. The package is **pre-1.0**; expect its surface to move.

For the full deployment procedure, use [`resonate-supabase-deployments-typescript`](../resonate-supabase-deployments-typescript/SKILL.md).

## Gaps — state these, don't discover them in production

- **Open correctness issues against task lifecycle.** The tracker carries confirmed bugs: `task.halt` succeeding on a task that `task.get` already reports fulfilled; `task.create` claiming a lease that `task.acquire` then refuses as timed out; a `resonate:target` of `''` producing a task dispatched once and never redelivered; task-timeout handlers redispatching logically dead workflows; `promise.register_listener` accepting internal promises whose timeout is never enforced; and a settlement cascade that dispatches an awaiter whose own promise is already dead. Check [open issues](https://github.com/resonatehq/resonate-pg/issues) before recommending this for anything with real money or real users behind it.
- **No test suite in the repository.** `test/conformance.py` is a *shim*, not a test — it exposes `resonate_rpc` over HTTP so an external model-based conformance harness can drive the database. There are no unit tests of the kind the NATS provider has, and none of the oracle-diff, crash, or linearizability suites the ScyllaDB provider ships. If a user is choosing between providers on evidence of correctness, this is the weakest of the three.
- **No production reference deployments.** Nobody is running this in production yet.
- **One client, and it targets Deno.** No Python, Go, Rust, or Java. No plain-Node story.
- **The client is pre-1.0.**
- **Timers have no fallback.** If the `pg_cron` job is unscheduled, paused, or lost in a restore, durable sleeps stop waking and nothing reports it.
- **Nothing scales independently of the database.** Throughput, connection limits, and failover are whatever the Postgres deployment provides.

## Checklist before recommending this to a user

1. Do they specifically want *no process to deploy*? If they merely use Postgres → core server on Postgres.
2. Are they on Postgres 16+ with `pg_cron` available? If `pg_cron` cannot be installed, this does not work at all.
3. Are they writing workers in TypeScript on Deno? If not → no client exists.
4. Have you verified `cron.job` contains `resonate_process_timeouts` after install?
5. Do any of their workflows live longer than 24 hours? If so, they must pass an explicit `timeout` to `invoke`.
6. Have you told them the GC horizon must exceed their retry window, or resubmission stops being idempotent?
7. Are workers connecting as `resonate_worker` rather than a superuser?
8. Have you told them there is no test suite and there are open task-lifecycle bugs?
9. Did you correctly say **Apache 2.0** — not BUSL-1.1, which is the other two providers?

## Examples

[`example/countdown`](https://github.com/resonatehq/resonate-pg/tree/main/example/countdown) — an empty project to a running durable workflow in about five minutes. [`example/research`](https://github.com/resonatehq/resonate-pg/tree/main/example/research) — a fan-out research agent that suspends while searches run and streams results back over a Realtime channel.
