# Resonate Skills

Agent skills for building with [Resonate](https://resonatehq.io) — a durable execution platform for long-running, crash-safe workflows.

Each skill teaches a coding agent (Claude Code, Cursor, or any skill-aware agent) how to reason about and write Resonate code for a specific task: deploying servers, building HTTP services, implementing sagas, coordinating human-in-the-loop approvals, and more.

## What's in this repo today

- **9 foundational** (language-agnostic) skills — concepts, mental models, the server-install guides, the CLI reference, the cross-SDK defaults reference, the Temporal and DBOS migration playbooks, and the `resonate-bash` MCP tool guide that apply across every Resonate SDK.
- **15 TypeScript** per-SDK skills — idiomatic usage of the TypeScript SDK.
- **8 Python** per-SDK skills — basic usage + debugging + patterns (saga, recursive fan-out, human-in-the-loop, external system of record) + HTTP service design for the Python SDK.
- **8 Rust** per-SDK skills — basic usage + debugging + patterns (saga, recursive fan-out, durable-sleep-scheduled-work, human-in-the-loop, external system of record) for the early-development Rust SDK (v0.1.0, not yet on crates.io); every Rust skill carries an explicit v0.1.0 caveat.
- **8 Go** per-SDK skills — basic usage (ephemeral + durable) + debugging + patterns (saga, recursive fan-out, durable-sleep, human-in-the-loop, external system of record) for the pre-release Go SDK (no semver tag yet; install `@latest` or pin a commit); every Go skill carries a pre-release caveat and notes the two surfaces the Go SDK does not yet expose — a top-level `Schedule` API and a `promises` sub-client.

## What is a skill?

A skill is a directory with a `SKILL.md` file (and optional `references/`, `scripts/`, `assets/`). The frontmatter tells the agent when to load it:

```markdown
---
name: resonate-saga-pattern-typescript
description: Implement saga patterns for distributed transactions with compensation logic. Use when coordinating multi-step processes that need to maintain consistency across failures by unwinding completed steps.
license: Apache-2.0
---
```

Skills follow Anthropic's [Agent Skills](https://docs.anthropic.com/en/docs/claude-code/skills) convention and work with Claude Code, the Anthropic SDK, and other skill-aware tools.

## Taxonomy

Every skill falls into one of two categories.

**Foundational skills** are language-agnostic. They teach concepts, mental models, or operational patterns that apply across every Resonate SDK. Their directory names carry no SDK suffix.

**Per-SDK skills** teach idiomatic usage of a specific Resonate SDK. Their directory names and frontmatter carry an SDK suffix: `-typescript`, `-python`, `-rust`, or `-go`. When the same concept applies across SDKs (e.g. sagas, HTTP services, debugging), each SDK gets its own skill — the per-SDK version expresses the pattern in the SDK's natural idioms rather than translating mechanically.

### Foundational

**Start here:**
- [`resonate-philosophy`](resonate-philosophy/SKILL.md) — The foundational mindset. Read this first.
- [`durable-execution`](durable-execution/SKILL.md) — The core concept and Resonate's approach.
- [`resonate-migrate-from-temporal`](resonate-migrate-from-temporal/SKILL.md) — Coming from Temporal? Port your app pattern by pattern; maps `temporalio/samples-*` to Resonate examples across all four SDKs, with per-SDK API notes and honest coverage gaps.
- [`resonate-migrate-from-dbos`](resonate-migrate-from-dbos/SKILL.md) — Coming from DBOS? Port your app pattern by pattern; maps `dbos-inc` examples (demo apps + the SDK repos) to Resonate equivalents across all four SDKs, with per-SDK API notes and honest coverage gaps.

**Operations:**
- [`resonate-cli`](resonate-cli/SKILL.md) — Drive the Resonate server from the shell: `serve`/`dev`, promise CRUD + search, schedules, `invoke`, `tree`, tasks, and the MCP shim. Covers every subcommand and flag as of `resonate 0.9.7`, plus the small set of docs-vs-binary deltas an agent will trip on.
- [`resonate-server-deployment`](resonate-server-deployment/SKILL.md) — Install and configure the Resonate server on Linux with systemd.
- [`resonate-server-deployment-cloud-run`](resonate-server-deployment-cloud-run/SKILL.md) — Deploy the Resonate server to Google Cloud Run with Cloud SQL Postgres storage.
- [`resonate-bash`](resonate-bash/SKILL.md) — Run shell scripts as durable, asynchronous tasks via the `resonate-bash` MCP tool. What it's good at, how to install the local Resonate server + Claude Code MCP wiring, tool reference (params, target addresses, env vars, failure semantics).

**Reference:**
- [`resonate-defaults`](resonate-defaults/SKILL.md) — Cross-SDK default values for retry policies, `ctx.run` options, init, env vars, and server flags. Read this when asked "what's the default for X" instead of deflecting to "check the SDK source."

### Per-SDK: TypeScript

**Core SDK usage:**
- [`resonate-basic-ephemeral-world-usage-typescript`](resonate-basic-ephemeral-world-usage-typescript/SKILL.md) — Client APIs: initialization, registration, top-level invocations.
- [`resonate-basic-durable-world-usage-typescript`](resonate-basic-durable-world-usage-typescript/SKILL.md) — Context APIs inside `function*` bodies.
- [`resonate-basic-debugging-typescript`](resonate-basic-debugging-typescript/SKILL.md) — Investigating stuck workflows, error codes, unexpected replays.

**Reasoning:**
- [`resonate-advanced-reasoning-typescript`](resonate-advanced-reasoning-typescript/SKILL.md) — Mapping the Distributed Async Await spec to TypeScript SDK patterns and verifying correctness/durability/failure semantics.

**Patterns:**
- [`resonate-saga-pattern-typescript`](resonate-saga-pattern-typescript/SKILL.md) — Distributed transactions with compensation logic.
- [`resonate-recursive-fan-out-pattern-typescript`](resonate-recursive-fan-out-pattern-typescript/SKILL.md) — Parallel workflow execution via recursive fan-out.
- [`resonate-human-in-the-loop-pattern-typescript`](resonate-human-in-the-loop-pattern-typescript/SKILL.md) — Approval gates and manual review workflows.
- [`resonate-external-system-of-record-pattern-typescript`](resonate-external-system-of-record-pattern-typescript/SKILL.md) — Consistency across systems without distributed transactions.
- [`resonate-durable-sleep-scheduled-work-typescript`](resonate-durable-sleep-scheduled-work-typescript/SKILL.md) — Timers, countdowns, and delayed execution via `ctx.sleep()`.
- [`resonate-state-bus-pattern-typescript`](resonate-state-bus-pattern-typescript/SKILL.md) — Stream durable workflow state to browsers via an external realtime bus (Firestore, Supabase Realtime, Pub/Sub) when the worker is short-lived.

**HTTP & authentication:**
- [`resonate-http-service-design-typescript`](resonate-http-service-design-typescript/SKILL.md) — HTTP services with Resonate behind route handlers.
- [`resonate-token-authentication-typescript`](resonate-token-authentication-typescript/SKILL.md) — JWT auth and prefix-based authorization.

**Deployments:**
- [`resonate-gcp-deployments-typescript`](resonate-gcp-deployments-typescript/SKILL.md) — Deploy TypeScript workers to Google Cloud Functions.
- [`resonate-supabase-deployments-typescript`](resonate-supabase-deployments-typescript/SKILL.md) — Resonate workflows on Supabase Edge Functions.
- [`resonate-lovable-usage-prompt-typescript`](resonate-lovable-usage-prompt-typescript/SKILL.md) — Building Resonate apps in Lovable.dev (Node.js/React/TypeScript).

### Per-SDK: Python

**Core SDK usage:**
- [`resonate-basic-ephemeral-world-usage-python`](resonate-basic-ephemeral-world-usage-python/SKILL.md) — Client APIs: initialization (`Resonate(url=...)`), registration via `r.register(fn)`, top-level invocations (`r.run`/`r.rpc`), dependencies (`r.with_dependency`), external promises.
- [`resonate-basic-durable-world-usage-python`](resonate-basic-durable-world-usage-python/SKILL.md) — Context APIs inside `async def` durable functions (`await ctx.run/rpc/sleep/promise`), determinism rules, and Python-specific deltas from TypeScript.
- [`resonate-basic-debugging-python`](resonate-basic-debugging-python/SKILL.md) — Python-specific failure modes: async/await pitfalls, v0.7.0 SDK + Rust server v0.9.x, Python ≥ 3.12 pin, non-determinism regressions.

**Patterns:**
- [`resonate-saga-pattern-python`](resonate-saga-pattern-python/SKILL.md) — Distributed transactions with compensation via `try/except` and reverse-order cleanup.
- [`resonate-recursive-fan-out-pattern-python`](resonate-recursive-fan-out-pattern-python/SKILL.md) — Parallel execution via list-comprehension over `ctx.begin_run` / `ctx.begin_rpc`, recursion, bounded parallelism.
- [`resonate-human-in-the-loop-pattern-python`](resonate-human-in-the-loop-pattern-python/SKILL.md) — Workflow steps that block on `ctx.promise(id=...)` until a webhook, UI, or operator resolves.
- [`resonate-external-system-of-record-pattern-python`](resonate-external-system-of-record-pattern-python/SKILL.md) — Coordinate writes to an external SoR (Postgres, TigerBeetle, Stripe) with idempotency keys.

**HTTP & service design:**
- [`resonate-http-service-design-python`](resonate-http-service-design-python/SKILL.md) — FastAPI/Flask/Django route handlers that start or await durable workflows; worker-group separation; webhook-driven promise resolution.

**Not yet available for Python:** token-based authentication (Python SDK doesn't yet support it — planned for a future release per the docs); GCP Cloud Functions / Supabase Edge deployments (no Python shim; Supabase Edge runtime is Deno-only). Durable-sleep-scheduled-work is not a separate Python skill because `ctx.sleep(timedelta(...))` is covered in basic-durable-world; a top-level `r.schedule(...)` cron API is available in v0.7.0 (see basic-ephemeral-world skill).

### Per-SDK: Rust

**v0.1.0 caveat:** the Rust SDK is in active development and not yet on crates.io. Install via git dependency. APIs may change between releases; every Rust skill carries this caveat at the top.

**Core SDK usage:**
- [`resonate-basic-ephemeral-world-usage-rust`](resonate-basic-ephemeral-world-usage-rust/SKILL.md) — Client APIs: `Resonate::new(ResonateConfig)` / `Resonate::local()`, `#[resonate::function]` attribute macro, registration, `.run()` / `.rpc()` / `.schedule()`, promises API.
- [`resonate-basic-durable-world-usage-rust`](resonate-basic-durable-world-usage-rust/SKILL.md) — Context APIs inside `#[resonate::function]`-decorated async functions: `ctx.run`, `ctx.rpc`, `ctx.sleep`, `.spawn()` for parallelism, function kinds (Workflow / Leaf with Info / Pure leaf).
- [`resonate-basic-debugging-rust`](resonate-basic-debugging-rust/SKILL.md) — Rust-specific failure modes: serde derive errors, `ctx` vs `info` confusion, `.spawn()` double-await, tokio runtime mismatches, v0.1.0 server-compat.

**Patterns:**
- [`resonate-saga-pattern-rust`](resonate-saga-pattern-rust/SKILL.md) — Distributed transactions with `Result<T>` + match-based compensation dispatch via enum; forward path uses `?` propagation.
- [`resonate-recursive-fan-out-pattern-rust`](resonate-recursive-fan-out-pattern-rust/SKILL.md) — Parallel execution via `.spawn().await?` double-await pattern; recursion; bounded parallelism via slice `.chunks(n)`.
- [`resonate-human-in-the-loop-pattern-rust`](resonate-human-in-the-loop-pattern-rust/SKILL.md) — Workflow steps that block on `ctx.promise::<T>()` until a webhook, UI, or operator settles via `resonate.promises.resolve/reject/cancel`.
- [`resonate-external-system-of-record-pattern-rust`](resonate-external-system-of-record-pattern-rust/SKILL.md) — Coordinate writes to an external SoR (Postgres, TigerBeetle, Stripe) with idempotency keys; type-dispatched DI via `ctx.get_dependency::<T>()`.
- [`resonate-durable-sleep-scheduled-work-rust`](resonate-durable-sleep-scheduled-work-rust/SKILL.md) — `ctx.sleep(Duration)` for in-workflow durable sleep + `resonate.schedule()` cron-registered ephemeral-world scheduling.

**Docs-vs-source note:** `ctx.promise::<T>()`, `ctx.get_dependency::<T>()`, `ctx.info()`, and `resonate.with_dependency::<T>()` all exist in the v0.1.0 SDK source but are not yet covered in `docs/develop/rust.mdx`. The Rust skills above use these APIs with source-path citations; docs are expected to catch up in a future release.

**Not yet written for Rust:** HTTP service design + deployment skills. These track SDK stability and will land when the source/docs validate the relevant paths.

### Per-SDK: Go

**Pre-release caveat:** the Go SDK has no semver-tagged release yet. Install with `go get github.com/resonatehq/resonate-sdk-go@latest` (resolves to a Go pseudo-version) or pin a commit for stability. APIs may change before the first tag is cut; every Go skill carries this caveat at the top, verified against `develop/go.mdx` and the `resonatehq-examples/*-go` repos.

**Core SDK usage:**
- [`resonate-basic-ephemeral-world-usage-go`](resonate-basic-ephemeral-world-usage-go/SKILL.md) — Client APIs: `resonate.New(Config)`, the package-level generic `resonate.Register`, top-level `RegisteredFunc.Run`, `Resonate.RPC` / `Resonate.Get`, typed vs untyped handles, `RunOptions`, and `Stop` semantics (don't Stop a worker).
- [`resonate-basic-durable-world-usage-go`](resonate-basic-durable-world-usage-go/SKILL.md) — Context APIs inside durable functions: `ctx.Run` / `ctx.RPC` / `ctx.Sleep` / `ctx.Promise` / `ctx.Detached`, `Future.Await`, option structs, accessors, the bounded 3-attempt default retry, and the replay model.
- [`resonate-basic-debugging-go`](resonate-basic-debugging-go/SKILL.md) — Go-specific failure modes: the `localnet` + `NoopHeartbeat{}` requirement, `ctx.Run`'s unchecked `any` leaf signatures, replay double-fires, the `Sender().PromiseSettle` base64 encoding trap (issue #28), and `r.Stop()` silently killing a live worker.

**Patterns:**
- [`resonate-saga-pattern-go`](resonate-saga-pattern-go/SKILL.md) — Distributed transactions with explicit `(T, error)` returns, a tracked `completed` slice, and reverse-order compensation dispatched via `type Step string` + `switch`.
- [`resonate-recursive-fan-out-pattern-go`](resonate-recursive-fan-out-pattern-go/SKILL.md) — Parallel execution via the dispatch-all-then-await-all `[]*resonate.Future` pattern; recursive `ctx.RPC` self-dispatch; worker-group separation.
- [`resonate-human-in-the-loop-pattern-go`](resonate-human-in-the-loop-pattern-go/SKILL.md) — Workflow steps that park on `ctx.Promise()` until an external actor settles via the CLI, the server HTTP API, or the low-level `Sender().PromiseSettle` — Go has **no** `promises` sub-client (issue #28).
- [`resonate-external-system-of-record-pattern-go`](resonate-external-system-of-record-pattern-go/SKILL.md) — Coordinate writes to an external SoR by wrapping every interaction in its own idempotent `ctx.Run`; idempotency keys derived from `ctx.ID()`.
- [`resonate-durable-sleep-scheduled-work-go`](resonate-durable-sleep-scheduled-work-go/SKILL.md) — `ctx.Sleep(time.Duration)` for in-workflow durable sleep, countdowns, and long-horizon delays. Go has **no** top-level `Schedule` API yet; recurring work uses in-workflow `ctx.Sleep` loops or external cron → `RPC`.

**Not yet written for Go:** HTTP service design, token-authentication, and deployment skills. The Go SDK can drive `net/http` services today (see `example-node-drain-orchestrator-go`), but the dedicated skill waits on a stable API surface; deployment skills track validated paths. Two SDK surfaces are also absent at pre-release — a top-level `Schedule` API and a `promises` sub-client (issue #28) — and the relevant skills note the workarounds rather than inventing the API.

## Using these skills

### With Claude Code

Clone this repo into your project's `.claude/skills/` directory (or your user-level `~/.claude/skills/`):

```bash
cd your-project
mkdir -p .claude/skills
git clone https://github.com/resonatehq/resonate-skills .claude/skills/resonate
```

Claude Code will automatically discover them. Invoke one explicitly with `Use the resonate-saga-pattern-typescript skill to...`, or let the agent pick based on the description field.

### With the Anthropic SDK

Load `SKILL.md` files as context when you want the model to apply that skill. See Anthropic's [Agent Skills docs](https://docs.anthropic.com/en/docs/claude-code/skills) for loading patterns.

### With other agents

Skills are plain Markdown. Any agent framework that supports prompt injection or context loading can use them.

## Contributing

### Where does a new skill belong?

Decision tree:

1. **Is it a concept, mental model, or framework-agnostic principle that applies equally across every Resonate SDK?** → **Foundational.** No SDK suffix. Describe the idea in language-agnostic terms and link to per-SDK skills for concrete syntax.
2. **Is it idiomatic usage of a specific SDK's API?** → **Per-SDK.** Suffix the directory name and frontmatter `name` with the SDK: `-typescript`, `-python`, `-rust`, or `-go`.
3. **Is it operational or deployment knowledge?** → **Per-SDK** if the SDK shapes the deployment (e.g. Cloud Functions runtime, worker registration). **Foundational** if the operation is truly language-independent (e.g. installing the Resonate server binary).

**Default to per-SDK.** Promote a skill to foundational only when it contains no SDK-specific APIs, syntax, or runtime idioms. Debug skills, for example, are per-SDK: error codes, replay tells, and diagnostic tooling differ enough across SDKs that a single unified debug skill would either read as abstract or balloon past an agent's working context. The concept of replay and durability lives in `durable-execution`; the SDK-specific tells live in each SDK's debug and advanced-reasoning skills.

### Same concept, multiple SDKs

When a pattern translates across SDKs (e.g. saga with compensation), each SDK gets its own skill — written idiomatically for that SDK, not find-and-replaced from another. If the natural expression differs materially (async/await vs generator syntax, `Result` vs exceptions), the skill reflects that. If the expressions are genuinely identical, consider whether the shared logic belongs in a foundational skill with the per-SDK skills covering only the surface syntax.

### Out of scope

- **Mirroring SDK reference documentation.** Skills are behaviour-oriented ("when would I reach for this?") — the per-SDK guides at [docs.resonatehq.io/docs/develop/](https://docs.resonatehq.io/docs/develop/) are the reference. Skills cite them.
- **New deployment targets without SDK validation.** A new deployment skill requires the target to have validated SDK support. Currently covered: GCP Cloud Functions, Supabase Edge Functions, and the Linux/systemd server install. Requests for AWS, Cloudflare, Fly, and others are tracked separately and land once an SDK and a benchmark cycle confirm the deployment path.
- **Frontmatter changes.** The Anthropic Agent Skills spec (`name`, `description`, `license`) is fixed. Only skill content is up for discussion.

### Submitting a skill

1. Create `your-skill-name[-typescript|-python|-rust|-go]/SKILL.md` with the frontmatter block.
2. Keep the `description` specific — it's what agents use to decide when to load the skill.
3. Put example snippets inline; put long-form references under `your-skill-name/references/`.
4. Keep prose tight: skills are read by LLMs with finite context.
5. One skill per PR unless they're tightly coupled (e.g. the TypeScript and Python variants of the same pattern).

## License

Apache License 2.0. See [LICENSE](LICENSE).
