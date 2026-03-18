---
name: small-change
description: Implements a small, scoped change to an existing domain — adding a field, renaming, adjusting validation, tweaking business logic. No new domain created.
---

# Small Change

Implements a targeted change to an existing domain. Examples: add a field to an entity, change validation rules, add an optional endpoint parameter, rename a value object.

If the change introduces a new domain or requires creating new modules → use `/new-domain` or `/feature` instead.

---

## Phase 0 — Refinement

**GitHub Issue:** If the user references an issue number, use it. If no issue exists, create one in DOCS_REPO after refinement is complete — the issue body should contain the refined requirements. Use the issue number for the branch name.

Read in parallel:
- `docs/reminders.md` — **check `health-check-counter`: if N ≥ HEALTH_CHECK_THRESHOLD, warn the user and ask whether to proceed or run `/health-check` first**
- `docs/rules/<domain>.md` for the affected domain (warn if missing, continue)
- The GitHub Issue body (if it exists): `gh issue view <n> --repo DOCS_REPO`

Evaluate:
- Is the change clearly scoped? (one file, one field, one rule)
- Are there cascade effects? (e.g., adding a field → mapper, migration, presenter, frontend schema all need updating)
- Does this touch the Prisma schema? If yes, plan the migration step

**If anything is ambiguous:** ask the user before proceeding.

**If clear:** list every file that needs changing (including cascades) and ask for approval.

- If the user approves → proceed to implementation.
- If the user requests changes → adjust the plan and re-present for approval.
- If the user rejects → do not advance. Ask what they want to do instead.

---

## Branch

```bash
# Backend only
cd backend && git checkout development && git pull && git checkout -b BE-{issue-number}/{change-name}

# Frontend only
cd frontend && git checkout development && git pull && git checkout -b FE-{issue-number}/{change-name}

# Fullstack
cd backend && git checkout development && git pull && git checkout -b FS-{issue-number}/{change-name}
cd frontend && git checkout development && git pull && git checkout -b FS-{issue-number}/{change-name}
```

---

## Phase 1 — Planning

Launch an `Explore` agent (Haiku) to:
- Read the files to be changed
- Identify all cascade dependencies (e.g., a field rename touches entity, mapper, presenter, DTO, frontend schema)
- Check if Prisma schema is involved

Return:
- Complete list of files to modify
- Cascade order (what depends on what)

---

## Phase 2 — Implementation

Make changes in dependency order. Keep TDD: update existing tests first, or write new tests if none exist for the changed code. If the change touches frontend components, follow `docs/coding-patterns/frontend/accessibility.md` and `docs/coding-patterns/frontend/design-system.md` (semantic color tokens, typography scale, dark mode). **All frontend components must include `data-testid` attributes** — never use CSS selectors or text in Playwright tests.

If `backend/prisma/schema.prisma` was modified:
```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev
```

---

## Phases 3-6

Follow [`docs/shared-phases.md`](../../../shared-phases.md) for Validation, Documentation, Code Review, and Commit/PR.

**Small-change-specific notes:**
- Phase 3: **run E2E by default.** Skip E2E only if the change is strictly limited to entity fields, validation rules, or business logic fully covered by unit tests with no controller or frontend component changes. If the change touches controller signatures, API request/response shapes, frontend component behavior, or page routing, E2E is mandatory.
- Phase 4: no plan file is created for small changes — skip plan deletion
- Phase 5: include "Cascade effects handled" in the review summary
- Phase 6: commit message format: `feat(<domain>): <description>` (or `fix`/`refactor` as appropriate)

**Increment `health-check-counter` in `docs/reminders.md`** only after PR is successfully created. If PR creation fails, do not increment — fix the issue and retry the PR first.
```
health-check-counter: N+1
```

---

## Completion

After PR is created and `health-check-counter` is incremented, the small-change skill is complete. Confirm the PR URL with the user.
