---
name: resonate-recursive-fan-out-pattern-typescript
description: Implement recursive fan-out patterns for parallel workflow execution. Use when breaking down complex work into independent subtasks that execute recursively with automatic deduplication and load balancing.
license: Apache-2.0
---

# Resonate Recursive Fan-Out Pattern (TypeScript)

## Overview

The recursive fan-out pattern enables workflows to spawn multiple child workflows in parallel, either collecting their results or letting them execute independently. Resonate provides two primitives for this: `ctx.detached()` for fire-and-forget execution and `ctx.beginRpc()` for parallel execution with result gathering.

**Core principle:** Break complex work into parallel subtasks that recursively break down further, with automatic deduplication via promise IDs and transparent load balancing across workers.

## Mental Model

```
Parent Workflow
      │
      ├─────────┬─────────┬─────────┐
      │         │         │         │
   Child 1   Child 2   Child 3   Child 4
      │         │         │
      ├──┬──┐   ├──┐      └──┬──┐
   G1 G2 G3  G4 G5        G6 G7

Fan-out: Parent spawns multiple children
Recursive: Children spawn grandchildren
Parallel: All execute concurrently
Deduplication: Same ID => same promise
```

## Pattern 1: Detached Fan-Out (Fire-and-Forget)

**Use when:** You want to spawn independent work without waiting for results.

### Basic Pattern

```typescript
import { Context } from "@resonatehq/sdk";

function* parentWorkflow(ctx: Context, items: string[]) {
  // Spawn detached workflows for each item
  for (const item of items) {
    yield* ctx.detached(
      processItem,
      item,
      ctx.options({ id: `process/${item}` })
    );
  }

  // Parent completes immediately, children continue independently
  return { spawned: items.length };
}

function* processItem(ctx: Context, item: string) {
  // Process the item
  yield* ctx.run(async () => {
    console.log(`Processing ${item}`);
    // ... actual work ...
  });
}
```

### With Automatic Deduplication

```typescript
function* scrapeUser(
  ctx: Context,
  userId: string,
  depth: number
): Generator<any, void, any> {
  // Fetch user data
  const user = yield* ctx.run(fetchUserData, userId);

  console.log(`Scraped: ${user.handle}`);

  // Recursively scrape followers
  if (depth > 0) {
    const followers = yield* ctx.run(getFollowers, userId);

    for (const follower of followers) {
      // CRITICAL: Use follower ID as promise ID for deduplication
      // If same follower appears in multiple lists, only one scrape happens
      yield* ctx.detached(
        scrapeUser,
        follower.id,
        depth - 1,
        ctx.options({ id: follower.id })
      );
    }
  }
}
```

**Deduplication guarantee:** Multiple calls with same ID resolve to the same promise. A user appearing in 100 follower lists is scraped exactly once.

### Pagination with Fan-Out

```typescript
function* scrapeWithPagination(
  ctx: Context,
  userId: string,
  depth: number
) {
  const user = yield* ctx.run(fetchUser, userId);

  if (depth > 0) {
    let cursor: string | undefined = undefined;

    // Paginate through followers
    do {
      const page = yield* ctx.run(getFollowersPage, userId, cursor);

      // Spawn detached scrape for each follower on this page
      for (const follower of page.followers) {
        yield* ctx.detached(
          scrapeWithPagination,
          follower.id,
          depth - 1,
          ctx.options({ id: follower.id })
        );
      }

      cursor = page.cursor;
    } while (cursor !== undefined);
  }
}
```

## Pattern 2: Parallel RPC Fan-Out (Collect Results)

**Use when:** You need to gather and combine results from parallel subtasks.

### Basic Pattern

```typescript
function* parallelProcessing(ctx: Context, items: string[]) {
  const handles = [];

  // Spawn parallel RPCs
  for (const item of items) {
    const handle = yield* ctx.beginRpc(
      processItem,
      item
    );
    handles.push(handle);
  }

  // Collect all results
  const results = [];
  for (const handle of handles) {
    const result = yield* handle;
    results.push(result);
  }

  return { results };
}

function* processItem(ctx: Context, item: string) {
  const result = yield* ctx.run(async () => {
    // Process and return result
    return `processed: ${item}`;
  });

  return result;
}
```

### Real-World Example: Recursive Research Agent

```typescript
interface ResearchMessage {
  role: "system" | "user" | "assistant" | "tool";
  content: string;
  tool_calls?: Array<{
    id: string;
    function: { name: string; arguments: string };
  }>;
}

function* research(
  ctx: Context,
  topic: string,
  depth: number
): Generator<any, string, any> {
  const messages: ResearchMessage[] = [
    { role: "system", content: SYSTEM_PROMPT },
    { role: "user", content: `Research ${topic}` }
  ];

  while (true) {
    // Prompt LLM (only allow tool access if depth > 0)
    const message = yield* ctx.run(promptLLM, messages, depth > 0);
    messages.push(message);

    // If LLM wants to research subtopics, spawn parallel research
    if (message.tool_calls) {
      const handles = [];

      // Spawn detached research for each subtopic
      for (const tool_call of message.tool_calls) {
        const tool_name = tool_call.function.name;
        const tool_args = JSON.parse(tool_call.function.arguments);

        if (tool_name === "research") {
          // Recursively research subtopic with reduced depth
          const handle = yield* ctx.beginRpc(
            research,
            tool_args.topic,
            depth - 1
          );
          handles.push([tool_call, handle]);
        }
      }

      // Collect all subtopic research results
      for (const [tool_call, handle] of handles) {
        const result = yield* handle;
        messages.push({
          role: "tool",
          tool_call_id: tool_call.id,
          content: result
        });
      }
    } else {
      // No more tool calls, return final summary
      return message.content || "";
    }
  }
}

const SYSTEM_PROMPT = `
You are a recursive research agent.
When given a broad topic, break it down into 2-3 subtopics and call the "research" tool for each.
If the topic is specific, summarize it directly without calling the tool.
`;

async function promptLLM(
  _ctx: Context,
  messages: ResearchMessage[],
  hasToolAccess: boolean
) {
  const completion = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: messages,
    tools: TOOLS,
    tool_choice: hasToolAccess ? "auto" : "none"
  });

  return completion.choices[0]?.message;
}
```

## Pattern 3: Bounded Fan-Out (Rate Limiting)

**Use when:** You need to limit parallelism to avoid overwhelming external systems.

```typescript
function* boundedFanOut(
  ctx: Context,
  items: string[],
  batchSize: number
) {
  const results = [];

  // Process in batches
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const handles = [];

    // Spawn up to batchSize parallel RPCs
    for (const item of batch) {
      const handle = yield* ctx.beginRpc(processItem, item);
      handles.push(handle);
    }

    // Wait for this batch to complete before starting next
    for (const handle of handles) {
      const result = yield* handle;
      results.push(result);
    }
  }

  return { results };
}
```

## Pattern 4: Dynamic Fan-Out (Decision-Based)

**Use when:** Children determine whether to spawn grandchildren.

```typescript
function* dynamicFanOut(
  ctx: Context,
  url: string,
  maxDepth: number,
  currentDepth: number = 0
) {
  // Fetch page
  const page = yield* ctx.run(fetchPage, url);

  // Process this page
  yield* ctx.run(processPage, page);

  // Decide whether to crawl deeper
  if (currentDepth < maxDepth && shouldCrawlDeeper(page)) {
    const links = extractLinks(page);

    // Fan out to linked pages
    for (const link of links) {
      yield* ctx.detached(
        dynamicFanOut,
        link,
        maxDepth,
        currentDepth + 1,
        ctx.options({ id: link }) // Deduplicate by URL
      );
    }
  }
}

function shouldCrawlDeeper(page: any): boolean {
  // Custom logic: only crawl if page matches criteria
  return page.isRelevant && page.linkCount < 100;
}
```

## Deduplication Strategy

**Why deduplication matters:**
- Same user can appear in multiple follower lists
- Same URL can be linked from multiple pages
- Same subtask can be triggered from different paths

**How Resonate deduplicates:**
```typescript
// First call: Creates new promise with ID "user/123"
yield* ctx.detached(scrape, "user/123", ctx.options({ id: "user/123" }));

// Second call (from different branch): Resolves to SAME promise
yield* ctx.detached(scrape, "user/123", ctx.options({ id: "user/123" }));

// Result: "user/123" is scraped exactly once, not twice
```

**Promise ID strategies:**
```typescript
// ✅ GOOD - Natural unique identifier
ctx.options({ id: user.id })
ctx.options({ id: url })
ctx.options({ id: `task/${taskId}` })

// ✅ GOOD - Deterministic composite key
ctx.options({ id: `${parentId}/${childId}` })

// ❌ BAD - Non-deterministic (random on replay)
ctx.options({ id: `${Date.now()}` })
ctx.options({ id: `${Math.random()}` })

// ✅ BEST - Auto-generated (if no deduplication needed)
yield* ctx.detached(work)  // Resonate generates deterministic ID
```

## Load Balancing

**Automatic distribution:**
```typescript
// Define worker group
const resonate = new Resonate({
  url: "http://localhost:8001",
  group: "workers"
});

// Register functions
resonate.register("processItem", processItem);

// Start multiple worker instances
// Worker 1: npm start
// Worker 2: npm start
// Worker 3: npm start

// Detached/RPC calls automatically distribute across available workers
yield* ctx.detached(processItem, item);  // Routed to worker with capacity
```

**Target specific worker group:**
```typescript
yield* ctx.beginRpc(
  processItem,
  item,
  ctx.options({
    target: "poll://any@workers"  // Any worker in "workers" group
  })
);
```

## Common Patterns

### Pattern: Map-Reduce

```typescript
function* mapReduce(ctx: Context, inputs: string[]) {
  const handles = [];

  // Map phase: spawn parallel work
  for (const input of inputs) {
    const handle = yield* ctx.beginRpc(mapFunction, input);
    handles.push(handle);
  }

  // Collect mapped results
  const mapped = [];
  for (const handle of handles) {
    const result = yield* handle;
    mapped.push(result);
  }

  // Reduce phase: combine results
  const reduced = yield* ctx.run(reduceFunction, mapped);

  return reduced;
}
```

### Pattern: Tree Traversal

```typescript
function* traverseTree(ctx: Context, nodeId: string) {
  // Process current node
  const node = yield* ctx.run(fetchNode, nodeId);
  yield* ctx.run(processNode, node);

  // Recursively process children
  for (const childId of node.children) {
    yield* ctx.detached(
      traverseTree,
      childId,
      ctx.options({ id: childId })
    );
  }
}
```

### Pattern: Retry with Fan-Out

```typescript
function* retryableFanOut(
  ctx: Context,
  items: string[],
  maxAttempts: number = 3
) {
  const handles = [];

  for (const item of items) {
    const handle = yield* ctx.beginRpc(
      retryableTask,
      item,
      maxAttempts
    );
    handles.push([item, handle]);
  }

  const results = [];
  const failures = [];

  for (const [item, handle] of handles) {
    try {
      const result = yield* handle;
      results.push({ item, result });
    } catch (error) {
      failures.push({ item, error });
    }
  }

  return { results, failures };
}

function* retryableTask(ctx: Context, item: string, maxAttempts: number) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return yield* ctx.run(processItem, item);
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      yield* ctx.sleep(1000 * attempt);  // Exponential backoff
    }
  }
}
```

## Performance Considerations

**Detached vs RPC:**
- `ctx.detached()`: Fire-and-forget, parent doesn't wait
- `ctx.beginRpc()`: Parent waits for results, blocks until all complete

**When to use detached:**
- Independent tasks (scraping, notifications, logging)
- No need for results in parent
- Maximum parallelism

**When to use RPC:**
- Need results to continue
- Aggregation/reduction operations
- Coordinated execution

**Memory:** Thousands of parallel workflows are fine. Resonate handles state externally.

## Common Pitfalls

### 1. Missing Deduplication ID

```typescript
// ❌ WRONG - Creates duplicate work
for (const follower of followers) {
  yield* ctx.detached(scrape, follower.id);  // No ID specified!
}

// ✅ CORRECT - Deduplicates by follower ID
for (const follower of followers) {
  yield* ctx.detached(
    scrape,
    follower.id,
    ctx.options({ id: follower.id })
  );
}
```

### 2. Blocking on Detached Workflows

```typescript
// ❌ WRONG - Can't await detached workflow
const handle = yield* ctx.detached(work);
const result = yield* handle;  // Error: detached workflows don't return handles

// ✅ CORRECT - Use beginRpc if you need results
const handle = yield* ctx.beginRpc(work);
const result = yield* handle;
```

### 3. Non-Deterministic Promise IDs

```typescript
// ❌ WRONG - Different ID on replay
yield* ctx.detached(
  work,
  data,
  ctx.options({ id: `task-${Date.now()}` })
);

// ✅ CORRECT - Deterministic ID
yield* ctx.detached(
  work,
  data,
  ctx.options({ id: `task-${data.id}` })
);
```

## Decision Tree

**Use recursive fan-out when:**
- Breaking work into parallel subtasks
- Recursive data structures (trees, graphs)
- Unknown depth (crawling, scraping)
- Natural deduplication via IDs

**Use detached when:**
- Fire-and-forget execution
- Independent side effects
- Don't need results

**Use beginRpc when:**
- Need to collect and combine results
- Parallel map-reduce operations
- Coordinated execution

## Summary

Recursive fan-out patterns enable:
- Massive parallelism across distributed workers
- Automatic deduplication via promise IDs
- Recursive tree/graph traversal
- Load-balanced execution
- Both fire-and-forget and result-gathering modes

**Core recipe:** Spawn children with `ctx.detached()` or `ctx.beginRpc()` → Use deterministic IDs for deduplication → Let Resonate handle distribution and recovery
