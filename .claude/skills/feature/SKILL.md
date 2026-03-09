---
name: feature
description: Implements a new feature (endpoint, screen, or fullstack slice) from a GitHub Issue. Follows Phase 0 → 6 workflow.
---

# Feature

Implements a new feature from a GitHub Issue. A feature is scoped to an existing domain — if a new domain is needed, use `/new-domain` instead.

---

## Phase 0 — Refinement

**GitHub Issue:** If the user references an issue number, use it. If no issue exists, create one in DOCS_REPO after refinement is complete — the issue body should contain the refined requirements. Use the issue number for the branch name.

Read in parallel:
- `docs/reminders.md` — **check `health-check-counter`: if N ≥ HEALTH_CHECK_THRESHOLD, warn the user and ask whether to proceed or run `/health-check` first**
- `docs/rules/<domain>.md` for the domain involved
- The GitHub Issue body (if it exists): `gh issue view <n> --repo DOCS_REPO`

Evaluate:
- Is the scope clear? (what to build, what to change)
- Is the domain rule file populated? If not → block, ask the user to confirm running `/domain-discovery` first. After `/domain-discovery` completes and `docs/rules/<domain>.md` is created, resume Phase 0 — re-validate the file and continue evaluation.
- Are there open questions in the issue or rules that affect this feature?
- Are there related reminders that apply?

**A feature is ambiguous if any of these questions cannot be answered from the issue:**
- Which entities are affected?
- What is the expected behavior on error?
- Who can perform this action (authorization)?

**If anything is missing or ambiguous:** present the specific gaps to the user and ask before proceeding.

**Plan-based feature gating:** If the project uses subscription plans and this feature is restricted to specific plans, the access control should be implemented via CASL abilities (e.g., `can('access', 'AdvancedReports')` for Pro plan users). This only applies if the project has plan-based access defined in `docs/product.md`.

**If complete:** present a summary of what will be built and ask for explicit approval.

- If the user approves → proceed to Phase 1.
- If the user requests changes → adjust the plan and re-present for approval.
- If the user rejects → do not advance. Ask what they want to do instead.

---

## Branch

After Phase 0 approval, create the branch(es):

```bash
# Backend only
cd backend && git checkout development && git pull && git checkout -b BE-{issue-number}/{feature-name}

# Frontend only
cd frontend && git checkout development && git pull && git checkout -b FE-{issue-number}/{feature-name}

# Fullstack — same name in both repos
cd backend && git checkout development && git pull && git checkout -b FS-{issue-number}/{feature-name}
cd frontend && git checkout development && git pull && git checkout -b FS-{issue-number}/{feature-name}
```

**If branch creation fails:** check the error — if the branch already exists (`git checkout` to it), if the working tree is dirty (`git stash` or ask the user). Do not proceed to Phase 1 without a clean branch.

---

## Phase 1 — Planning

Launch an `Explore` agent (Haiku) to:
- Read `docs/architecture.md` to understand where the feature fits
- Read `docs/coding-patterns/backend/` and/or `docs/coding-patterns/frontend/` for the artifact types involved (always include `design-system.md` when frontend is in scope)
- Read existing files in the affected module (controllers, use cases, entities)

Return:
- Final list of files to create or modify
- Creation order (respecting dependencies)
- What can be parallelized

Write the plan to `docs/plans/YYYY-MM-DD-<feature-name>.md`.

---

## Phase 2 — Implementation

**TDD is mandatory. Write the test before the code. RED → GREEN → REFACTOR.**

For each artifact, follow this order:
1. Write the failing test
2. Write the minimum code to make it pass
3. Refactor

Implementation order (backend):
1. Domain: entity changes, new value objects, domain events
2. Application: use case(s) with Either<Error, Result>
3. Infrastructure: repository, mapper, Prisma schema (if changed)
4. HTTP: controller, presenter, Zod validation pipe

Implementation order (frontend):
1. Schema (Zod)
2. API function
3. Server action
4. Store (if needed)
5. Components (with `data-testid` on all interactive elements)
6. Page integration

Follow `docs/coding-patterns/frontend/accessibility.md` for all new components and pages — semantic HTML, keyboard navigation, labels, alt text.

Follow `docs/coding-patterns/frontend/design-system.md` for all frontend work — semantic color tokens (never raw colors), typography scale, dark mode readiness.

**Parallelize independent artifacts** using multiple `general-purpose` agents (Sonnet) when files have no dependency between them.

If `backend/prisma/schema.prisma` was modified:
```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev
```

---

## Phases 3-6

Follow [`docs/shared-phases.md`](../../../shared-phases.md) for Validation, Documentation, Code Review, and Commit/PR.

**Feature-specific notes:**
- Phase 4: delete `docs/plans/YYYY-MM-DD-<feature-name>.md`
- Phase 6: commit message format: `feat(<domain>): <description>`
- Phase 6: for fullstack features, create backend PR first (`Refs DOCS_REPO#<issue-number>`), then frontend PR (`Closes DOCS_REPO#<issue-number>`). Merge order: backend first, then frontend.

**Increment `health-check-counter` in `docs/reminders.md`** only after PR(s) are successfully created. If PR creation fails, do not increment — fix the issue and retry the PR first.
```
health-check-counter: N+1
```

---

## Completion

After PR(s) are created and `health-check-counter` is incremented, the feature skill is complete. Confirm the PR URL(s) with the user.
