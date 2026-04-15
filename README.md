# Resonate Skills

Agent skills for building with [Resonate](https://resonatehq.io) — a durable execution platform for long-running, crash-safe workflows.

Each skill teaches a coding agent (Claude Code, Cursor, or any skill-aware agent) how to reason about and write Resonate code for a specific task: deploying servers, building HTTP services, implementing sagas, coordinating human-in-the-loop approvals, and more.

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

## The skills

**Start here:**
- [`resonate-philosophy`](resonate-philosophy/SKILL.md) — The foundational mindset. Read this first.
- [`durable-execution`](durable-execution/SKILL.md) — The core concept and Resonate's approach.

**Reasoning & debugging:**
- [`resonate-advanced-reasoning`](resonate-advanced-reasoning/SKILL.md) — Mapping the Distributed Async Await spec to SDK patterns.
- [`resonate-basic-debugging-typescript`](resonate-basic-debugging-typescript/SKILL.md) — Investigating stuck workflows, error codes, unexpected replays.

**Core SDK usage (TypeScript):**
- [`resonate-basic-ephemeral-world-usage-typescript`](resonate-basic-ephemeral-world-usage-typescript/SKILL.md) — Client APIs: initialization, registration, top-level invocations.
- [`resonate-basic-durable-world-usage-typescript`](resonate-basic-durable-world-usage-typescript/SKILL.md) — Context APIs inside `function*` bodies.

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
- [`resonate-server-deployment`](resonate-server-deployment/SKILL.md) — Install and configure the Resonate server on Linux with systemd.
- [`resonate-gcp-deployments-typescript`](resonate-gcp-deployments-typescript/SKILL.md) — Deploy TypeScript workers to Google Cloud Functions.
- [`resonate-supabase-deployments-typescript`](resonate-supabase-deployments-typescript/SKILL.md) — Resonate workflows on Supabase Edge Functions.
- [`resonate-lovable-usage-prompt`](resonate-lovable-usage-prompt/SKILL.md) — Building Resonate apps in Lovable.dev.

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

These skills evolve alongside the Resonate SDK. If you hit a pattern that isn't covered, or you catch a skill teaching an outdated API, please open an issue or PR.

When writing a new skill:
1. Create `your-skill-name/SKILL.md` with frontmatter (`name`, `description`, `license: Apache-2.0`)
2. The `description` is what agents use to decide when to load the skill — make it specific
3. Put example snippets inline; put long-form references under `your-skill-name/references/`
4. Keep prose tight: skills are read by LLMs with finite context

## License

Apache License 2.0. See [LICENSE](LICENSE).
