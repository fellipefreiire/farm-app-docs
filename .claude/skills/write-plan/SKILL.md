---
name: write-plan
description: Opus-only planning skill. Refines requirements, reads all relevant coding patterns, and writes a complete self-contained implementation spec to docs/plans/. Ends with a HANDOFF — never implements code.
---

# Write Plan

Opus-only. Produces a complete, self-contained implementation spec that the local model can execute without reading any other file.

**This skill never writes implementation code.** If code samples are needed to clarify contracts, they are written as spec — not as working implementation.

---

## Step 0 — Read configuration

Read `docs/CLAUDE.md` configuration block. Confirm:
- `USE_GITHUB_ISSUES` value (show it explicitly)
- Current git branch: `git branch --show-current` in both `backend/` and `frontend/` as needed

---

## Step 1 — Refinement

Read in parallel:
- `docs/reminders.md` — check `health-check-counter`: if N ≥ HEALTH_CHECK_THRESHOLD, warn the user and ask whether to proceed or run `/health-check` first
- `docs/rules/<domain>.md` for the domain involved — if missing, block and ask user to run `/domain-discovery` first
- `docs/product.md` and `docs/architecture.md` — for context on scope and module boundaries
- GitHub Issue if `USE_GITHUB_ISSUES = true`: `gh issue view <n> --repo DOCS_REPO`

Evaluate:
- Is the scope clear? (what to build, what not to build)
- Which entities are affected?
- What is the expected behavior on error?
- Who can perform this action (authorization)?

**If anything is missing or ambiguous:** present the specific gaps and ask before proceeding. Never assume.

**If complete:** present a summary of what will be built and ask for explicit approval before advancing to Step 2.

If `USE_GITHUB_ISSUES = true` and no issue exists yet: create one in DOCS_REPO after refinement is complete. The issue body contains the refined requirements.

---

## Step 2 — Read coding patterns

Launch an `Explore` agent (Haiku) to read ALL coding pattern files relevant to this feature:

**Always include:**
- `docs/coding-patterns/backend/use-case.md`
- `docs/coding-patterns/backend/entity.md`
- `docs/coding-patterns/backend/repository.md`
- `docs/coding-patterns/backend/controller.md`
- `docs/coding-patterns/backend/tests.md`

**Include if applicable:**
- `docs/coding-patterns/backend/events.md` — if domain events are involved
- `docs/coding-patterns/backend/query-bus.md` — if cross-domain queries are needed
- `docs/coding-patterns/backend/mapper.md` — if new Prisma model is involved
- `docs/coding-patterns/backend/presenter.md` — if new response shape
- `docs/coding-patterns/frontend/schema.md` — if frontend is in scope
- `docs/coding-patterns/frontend/action.md` — if server actions are involved
- `docs/coding-patterns/frontend/component.md` — if new components
- `docs/coding-patterns/frontend/page.md` — if new page
- `docs/coding-patterns/frontend/design-system.md` — always include when frontend is in scope

The Explore agent must **return the extracted content** from each file — not just confirm it read them. If the agent returns nothing, re-run before proceeding.

Also read existing files in the affected module (entities, use cases, controllers) to understand current shape.

---

## Step 3 — Write the plan

Write the plan to `docs/plans/YYYY-MM-DD-<feature-name>.md` using `docs/plans/SPEC_TEMPLATE.md` as the base structure.

The plan must be **fully self-contained**: the local model must be able to implement it without reading any other file.

**Mandatory sections:**

### Patterns to read
List the exact pattern files the local model must read before starting. Do not list files that are not relevant.

### Context
Current branch, affected domain, scope boundaries.

### Contracts
Exact TypeScript interfaces, types, and function signatures. No "roughly like this" — exact code.

For backend use cases:
```typescript
// Exact signature required
export class CreateXxxUseCase {
  async execute(input: CreateXxxInput): Promise<Either<XxxError | YyyError, { data: Xxx }>>
}
```

For frontend schemas:
```typescript
// Exact Zod schema
export const xxxSchema = z.object({ ... })
```

### Files
- **Create:** path → what it does
- **Modify:** path → exactly what to change and where
- **Do not touch:** path → why

### Implementation sequence
Numbered steps the local model follows in order. No skipping.

1. **Step name**: exact description of what to do, including pattern to follow
2. ...

For each step involving a new file: include the exact structure the file must follow (extracted from the coding pattern).

### Tests
| Scenario | Input | Expected output |
|---|---|---|
| Happy path | ... | ... |
| Error case | ... | typed error |
| Edge case | ... | ... |

Include the test class structure:
```typescript
describe('XxxUseCase', () => {
  it('should ...', async () => { ... })
  it('should return XxxError when ...', async () => { ... })
})
```

### Restrictions
- [ ] DO NOT modify [specific file] — reason
- [ ] DO NOT create migrations in this step — reason
- [ ] Use [library X] not [library Y] — reason
- [ ] Follow async/await pattern throughout

### Prisma
If schema changes are needed, include the exact model definition to add or modify:
```prisma
model Xxx {
  id        String   @id @default(uuid())
  ...
}
```

---

## Step 4 — Self-review checklist

Before emitting the HANDOFF, verify and answer each item explicitly:

- [ ] Contracts are exact TypeScript — no approximations
- [ ] Every file to create/modify is listed with its path
- [ ] Implementation sequence is ordered with no gaps
- [ ] Tests have explicit input/output for each scenario
- [ ] Restrictions are explicit (what NOT to do)
- [ ] Patterns are embedded or referenced with exact file paths
- [ ] Prisma changes include exact model definition
- [ ] **Can this plan be executed without reading any other project file?**

If any item is unchecked → complete it before emitting the HANDOFF.

---

## ⏸ Handoff

After self-review passes:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /write-plan complete
Completed: spec written to docs/plans/YYYY-MM-DD-<feature-name>.md
Next step: in the local model terminal, run /implement docs/plans/YYYY-MM-DD-<feature-name>.md
Waiting for: confirmation that implementation is complete
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
