---
name: refactor
description: Cleans up, reorganizes, or improves existing code without changing behavior. All tests must pass before and after.
---

# Refactor

Improves the internal structure of existing code without changing its observable behavior. Examples: extract a value object, reduce cyclomatic complexity, rename for clarity, split a large use case, move a module.

**Hard rule:** tests must pass before the refactor starts and after it ends. If tests are failing before you begin → fix them first (use `/bugfix`).

---

## Phase 0 — Scope Definition

Read:
- `docs/reminders.md` — **check `health-check-counter`: if N ≥ HEALTH_CHECK_THRESHOLD, inform the user** (refactors don't increment it, but the user should know a health-check is due)
- The files to be refactored
- `docs/coding-patterns/` for the relevant artifact types

Define clearly:
- What is being improved (clarity, complexity, structure, naming)
- What is NOT changing (behavior, API contracts, public interfaces)
- Which files are affected

**Present the scope to the user and ask for approval before starting.**

---

## Baseline

Before making any changes, confirm all tests pass:

```bash
cd backend && pnpm test
cd frontend && pnpm build && pnpm lint  # if frontend is in scope
```

If tests are failing → stop. Fix with `/bugfix` first.

---

## Implementation

Apply changes incrementally. After each logical step, verify tests still pass:

```bash
cd backend && pnpm test && tsc --noEmit
```

Common refactor patterns:
- **Extract value object** — move primitive validation into a typed class
- **Extract use case** — split a use case doing too many things
- **Rename** — update all references (entity, mapper, tests, imports)
- **Reduce complexity** — replace conditionals with polymorphism or early returns
- **Move module** — update all imports, module registration, and barrel exports

---

## Validation

> **Note:** The quality thresholds below follow the same standards as Phase 3 in `docs/shared-phases.md`. Refer to that document for the full threshold table and failure handling.

```bash
# Backend
cd backend && pnpm test --coverage
cd backend && tsc --noEmit

# Frontend (if in scope)
cd frontend && pnpm build
cd frontend && pnpm lint
cd frontend && tsc --noEmit
```

Coverage must not decrease after the refactor. If it does → add tests to compensate.

---

## Code Review

Present to the user:

```
## Refactor — <description>

### What was improved
<what was changed and why — no behavior changes>

### Files modified
- path/to/file.ts — <what changed>

### Test results (before → after)
- Before: X/X passing, coverage X%
- After: X/X passing, coverage X%

### Behavior preserved
- All existing tests pass unchanged
```

**Wait for explicit user approval before committing.**

---

## Completion

After Code Review approval:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /refactor complete
Completed: refactor implemented, validated, and approved
Next step: run /commit when you are ready to commit and push
Waiting for: explicit /commit invocation — do not proceed automatically
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Refactors do **not** increment the `health-check-counter`.
