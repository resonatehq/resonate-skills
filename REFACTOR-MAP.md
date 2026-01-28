# Skills Refactoring Map

## Target Skills (11 total)

1. resonate-basic-ephemeral-world-usage-typescript
2. resonate-basic-durable-world-usage-typescript
3. resonate-basic-debugging-typescript
4. resonate-basic-reasoning-typescript
5. resonate-http-service-design-typescript
6. resonate-gcp-deployments-typescript
7. resonate-supabase-deployments-typescript
8. resonate-advanced-reasoning
9. resonate-human-in-the-loop-pattern-typescript
10. resonate-saga-pattern-typescript
11. resonate-lovable-usage-prompt

## Mapping from Existing

### Direct Renames (add -typescript suffix)

| Existing | Target | Action |
|----------|---------|--------|
| resonate-basic-ephemeral-world-usage | resonate-basic-ephemeral-world-usage-typescript | Rename directory |
| resonate-basic-durable-world-usage | resonate-basic-durable-world-usage-typescript | Rename directory |
| resonate-http-service-design | resonate-http-service-design-typescript | Rename directory |
| resonate-basic-reasoning | resonate-basic-reasoning-typescript | Rename directory |

### Transform/Merge

| Existing | Target | Action |
|----------|---------|--------|
| resonate-debug-troubleshoot | resonate-basic-debugging-typescript | Rename + update name field |
| resonate-google-cloud-typescript | resonate-gcp-deployments-typescript | Rename + update name field |
| resonate-supabase-typescript | resonate-supabase-deployments-typescript | Rename + fix filename (skill.md → SKILL.md) |
| resonate-advanced-reasoning | resonate-advanced-reasoning | Keep as-is (no -typescript suffix) |

### Create New

| Source | Target | Action |
|--------|---------|--------|
| Patterns/wait-for-external-event.md | resonate-human-in-the-loop-pattern-typescript | Create new skill, adapt pattern content |
| (New content) | resonate-saga-pattern-typescript | Create from scratch using Patterns/ + docs |
| (New content) | resonate-lovable-usage-prompt | Create specialized prompt for Lovable dev |

### Content to Distribute or Remove

| Existing | Action |
|----------|--------|
| resonate-coding-style | Distribute best practices into other skills, then delete |
| resonate-develop-typescript | Distribute practical TS patterns, then delete |
| resonate-distributed-async-await | Reference in advanced-reasoning, delete standalone |
| resonate-operate-deploy | Remove (deployment covered by GCP/Supabase skills) |
| Integration.md | Remove (legacy) |
| Patterns/README.md | Remove (incorporated into pattern skills) |
| RESONATE_INSTRUCTIONS_1.md | Remove (legacy) |

## Refactoring Order

1. ✅ Create REFACTOR-MAP.md
2. Rename existing skills (add -typescript suffixes where needed)
3. Fix supabase skill filename (skill.md → SKILL.md)
4. Update name fields in YAML frontmatter
5. Create resonate-human-in-the-loop-pattern-typescript
6. Create resonate-saga-pattern-typescript
7. Create resonate-lovable-usage-prompt
8. Delete obsolete skills and files
9. Verify all 11 target skills exist
10. Update any cross-references between skills
