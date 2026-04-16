# Resonate Skills

Agent skills for building with [Resonate](https://resonatehq.io) — a durable execution platform for long-running, crash-safe workflows.

Each skill teaches a coding agent (Claude Code, Cursor, or any skill-aware agent) how to reason about and write Resonate code for a specific task: deploying servers, building HTTP services, implementing sagas, coordinating human-in-the-loop approvals, and more.

## What's in this repo today

- **5 foundational** (language-agnostic) skills — concepts, mental models, and operational patterns that apply across every Resonate SDK.
- **13 TypeScript** per-SDK skills — idiomatic usage of the TypeScript SDK.
- **Python and Rust** per-SDK coverage is being added in a staged expansion; the Rust SDK is early-development (v0.1.0, not yet on crates.io) and its skill coverage will track SDK stability.

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

**Per-SDK skills** teach idiomatic usage of a specific Resonate SDK. Their directory names and frontmatter carry an SDK suffix: `-typescript`, `-python`, or `-rust`. When the same concept applies across SDKs (e.g. sagas, HTTP services, debugging), each SDK gets its own skill — the per-SDK version expresses the pattern in the SDK's natural idioms rather than translating mechanically.

### Foundational

**Start here:**
- [`resonate-philosophy`](resonate-philosophy/SKILL.md) — The foundational mindset. Read this first.
- [`durable-execution`](durable-execution/SKILL.md) — The core concept and Resonate's approach.

**Reasoning & operations:**
- [`resonate-advanced-reasoning`](resonate-advanced-reasoning/SKILL.md) — Mapping the Distributed Async Await spec to SDK patterns.
- [`resonate-server-deployment`](resonate-server-deployment/SKILL.md) — Install and configure the Resonate server on Linux with systemd.
- [`resonate-lovable-usage-prompt`](resonate-lovable-usage-prompt/SKILL.md) — Building Resonate apps in Lovable.dev.

### Per-SDK: TypeScript

**Core SDK usage:**
- [`resonate-basic-ephemeral-world-usage-typescript`](resonate-basic-ephemeral-world-usage-typescript/SKILL.md) — Client APIs: initialization, registration, top-level invocations.
- [`resonate-basic-durable-world-usage-typescript`](resonate-basic-durable-world-usage-typescript/SKILL.md) — Context APIs inside `function*` bodies.
- [`resonate-basic-debugging-typescript`](resonate-basic-debugging-typescript/SKILL.md) — Investigating stuck workflows, error codes, unexpected replays.

**Patterns:**
- [`resonate-saga-pattern-typescript`](resonate-saga-pattern-typescript/SKILL.md) — Distributed transactions with compensation logic.
- [`resonate-recursive-fan-out-pattern-typescript`](resonate-recursive-fan-out-pattern-typescript/SKILL.md) — Parallel workflow execution via recursive fan-out.
- [`resonate-human-in-the-loop-pattern-typescript`](resonate-human-in-the-loop-pattern-typescript/SKILL.md) — Approval gates and manual review workflows.
- [`resonate-external-system-of-record-pattern-typescript`](resonate-external-system-of-record-pattern-typescript/SKILL.md) — Consistency across systems without distributed transactions.
- [`resonate-durable-sleep-scheduled-work-typescript`](resonate-durable-sleep-scheduled-work-typescript/SKILL.md) — Timers, countdowns, and delayed execution via `ctx.sleep()`.

**HTTP & authentication:**
- [`resonate-http-service-design-typescript`](resonate-http-service-design-typescript/SKILL.md) — HTTP services with Resonate behind route handlers.
- [`resonate-token-authentication-typescript`](resonate-token-authentication-typescript/SKILL.md) — JWT auth and prefix-based authorization.

**Deployments:**
- [`resonate-gcp-deployments-typescript`](resonate-gcp-deployments-typescript/SKILL.md) — Deploy TypeScript workers to Google Cloud Functions.
- [`resonate-supabase-deployments-typescript`](resonate-supabase-deployments-typescript/SKILL.md) — Resonate workflows on Supabase Edge Functions.

### Per-SDK: Python

Coming. The Python SDK (`resonate-sdk` on PyPI) ships to real users; skill coverage is being added to parity with TypeScript.

### Per-SDK: Rust

Coming. The Rust SDK is v0.1.0 and not yet on crates.io — skill coverage tracks SDK stability and will land once the API surface is stable.

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
2. **Is it idiomatic usage of a specific SDK's API?** → **Per-SDK.** Suffix the directory name and frontmatter `name` with the SDK: `-typescript`, `-python`, or `-rust`.
3. **Is it operational or deployment knowledge?** → **Per-SDK** if the SDK shapes the deployment (e.g. Cloud Functions runtime, worker registration). **Foundational** if the operation is truly language-independent (e.g. installing the Resonate server binary).

**Default to per-SDK.** Promote a skill to foundational only when it contains no SDK-specific APIs, syntax, or runtime idioms. Debug skills, for example, are per-SDK: error codes, replay tells, and diagnostic tooling differ enough across SDKs that a single unified debug skill would either read as abstract or balloon past an agent's working context. The concept of replay and durability lives in `durable-execution` and `resonate-advanced-reasoning`; the SDK-specific tells live in each SDK's debug skill.

### Same concept, multiple SDKs

When a pattern translates across SDKs (e.g. saga with compensation), each SDK gets its own skill — written idiomatically for that SDK, not find-and-replaced from another. If the natural expression differs materially (async/await vs generator syntax, `Result` vs exceptions), the skill reflects that. If the expressions are genuinely identical, consider whether the shared logic belongs in a foundational skill with the per-SDK skills covering only the surface syntax.

### Out of scope

- **Mirroring SDK reference documentation.** Skills are behaviour-oriented ("when would I reach for this?") — the per-SDK guides at [docs.resonatehq.io/docs/develop/](https://docs.resonatehq.io/docs/develop/) are the reference. Skills cite them.
- **New deployment targets without SDK validation.** A new deployment skill requires the target to have validated SDK support. Currently covered: GCP Cloud Functions, Supabase Edge Functions, and the Linux/systemd server install. Requests for AWS, Cloudflare, Fly, and others are tracked separately and land once an SDK and a benchmark cycle confirm the deployment path.
- **Frontmatter changes.** The Anthropic Agent Skills spec (`name`, `description`, `license`) is fixed. Only skill content is up for discussion.

### Submitting a skill

1. Create `your-skill-name[-typescript|-python|-rust]/SKILL.md` with the frontmatter block.
2. Keep the `description` specific — it's what agents use to decide when to load the skill.
3. Put example snippets inline; put long-form references under `your-skill-name/references/`.
4. Keep prose tight: skills are read by LLMs with finite context.
5. One skill per PR unless they're tightly coupled (e.g. the TypeScript and Python variants of the same pattern).

## License

Apache License 2.0. See [LICENSE](LICENSE).
