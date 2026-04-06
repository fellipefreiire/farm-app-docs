---
name: new-domain
description: Creates a new domain from scratch — entities, use cases, repository, controller, events, tests, and frontend slice. Requires docs/rules/<domain>.md to exist first.
---

# New Domain

Creates a complete new domain from scratch. A domain is a bounded context with its own entities, use cases, repository contract, infrastructure, HTTP layer, and frontend slice.

**Hard gate:** `docs/rules/<domain>.md` must exist and be populated before this skill runs. If missing → run `/domain-discovery` first.

**Suggestion:** run `/health-check` before starting — new domains have large architectural impact.

---

## Phase 0 — Refinement

**GitHub Issue:** If the user references an issue number, use it. If no issue exists, create one in DOCS_REPO after refinement is complete — the issue body should contain the refined requirements. Use the issue number for the branch name.

Read in parallel:
- `docs/reminders.md` — **check `health-check-counter`: if N ≥ HEALTH_CHECK_THRESHOLD, warn the user and ask whether to proceed or run `/health-check` first**
- `docs/rules/<domain>.md`
- `docs/architecture.md`
- The GitHub Issue body (if it exists): `gh issue view <n> --repo DOCS_REPO`

Evaluate:
- Is the domain boundary clear? (what belongs here, what doesn't)
- Are entities, value objects, and relationships defined in the rules file?
- Are use cases listed with their inputs, outputs, and business rules?
- Are there open questions in the rules that block implementation?
- Does this domain communicate with existing domains? If yes, are the contracts defined?

**A domain is ambiguous if any of these questions cannot be answered from the rules file:**
- Which entities compose the domain?
- Which use cases are needed (at minimum)?
- Are there dependencies on other domains, and are those contracts defined?

**If anything is missing or ambiguous:** present the specific gaps to the user and ask before proceeding.

**If complete:** present a summary of the full domain structure (entities, use cases, endpoints, frontend screens) and ask for explicit approval.

- If the user approves → proceed to Phase 1.
- If the user requests changes → adjust the plan and re-present for approval.
- If the user rejects → do not advance. Ask what they want to do instead.

---

## Phase 1 — Planning

Launch an `Explore` agent (Haiku) to:
- Read `docs/coding-patterns/backend/` — all files
- Read `docs/coding-patterns/frontend/` — all files
- Read `docs/architecture.md` to identify integration points with existing domains

Return:
- Complete file list for the new domain (backend + frontend)
- Creation order respecting layer dependencies organized in waves
- Which files can be created in parallel within each wave
- Integration points: which existing modules need changes

**Wave structure (backend):**
- Wave 1: entity, value objects, domain errors, domain events (no dependencies)
- Wave 2: use cases, repository interface (depends on Wave 1)
- Wave 3: in-memory repository, unit tests (depends on Wave 2)
- Wave 4: Prisma schema, mapper, Prisma repository (depends on Wave 2)
- Wave 5: controller, presenter, Zod pipes, module registration (depends on Wave 4)

**Wave structure (frontend):**
- Wave 1: schema, API function (no dependencies)
- Wave 2: store, server action (depends on Wave 1)
- Wave 3: components with `data-testid` (depends on Wave 2)
- Wave 4: page integration (depends on Wave 3)

Write the plan to `docs/plans/YYYY-MM-DD-<domain-name>.md`.

---

## Phase 2 — Implementation

**TDD is mandatory. Write the test before the code. RED → GREEN → REFACTOR.**

Execute waves sequentially. Within each wave, launch parallel `general-purpose` agents (Sonnet) for independent files.

**Backend layer order:**

1. **Domain layer** (`src/domain/<domain>/enterprise/`)
   - `entities/<entity>.ts` — entity with business invariants
   - `value-objects/*.ts` — value objects
   - `events/<entity>-<action>-event.ts` — domain events

2. **Application layer** (`src/domain/<domain>/application/`)
   - `use-cases/<action>-<entity>.ts` — one file per use case
   - `use-cases/errors/<entity>-<problem>-error.ts` — domain-specific errors
   - `repositories/<entity>-repository.ts` — repository interface (abstract class)
   - Each use case: Either<DomainError, { data: ... }>, no infrastructure imports

3. **Infrastructure — Database** (`src/infra/database/prisma/`)
   - Update `prisma/schema.prisma` with new model
   - Run migrations: `cd backend && pnpm prisma generate && pnpm prisma migrate dev`
   - `mappers/<domain>/prisma-<entity>.mapper.ts` — toDomain / toPrisma
   - `repositories/<domain>/prisma-<entity>.repository.ts`

4. **Infrastructure — HTTP** (`src/infra/http/`)
   - `controllers/<domain>/<action>-<entity>.controller.ts`
   - `presenters/<entity>.presenter.ts`

5. **Infrastructure — Events** (if domain events are consumed cross-module)
   - `subscribers/<entity>/on-<entity>-<action>.ts`

6. **Module registration**
   - `src/infra/http/controllers/<domain>/<domain>-controllers.module.ts`
   - Register in `AppModule`

**Frontend layer order:**

1. `src/domains/<domain>/schemas/` — Zod schemas matching API contracts
2. `src/domains/<domain>/api/` — typed fetch functions
3. `src/domains/<domain>/actions/` — server actions
4. `src/domains/<domain>/store/` — Zustand store (if needed)
5. `src/domains/<domain>/components/` — React components with `data-testid`
6. `src/app/(private)/<domain>/page.tsx` — page integration

Follow `docs/coding-patterns/frontend/accessibility.md` for all new components and pages — semantic HTML, keyboard navigation, labels, alt text.

Follow `docs/coding-patterns/frontend/design-system.md` for all frontend work — semantic color tokens (never raw colors), typography scale, dark mode readiness.

---

## Phases 3-5

Follow [`docs/shared-phases.md`](../../../shared-phases.md) for Validation, Documentation, Code Review, and Commit/PR.

**New-domain-specific notes:**
- Phase 4: update `docs/architecture.md` with the new domain and Mermaid diagram. Delete `docs/plans/YYYY-MM-DD-<domain-name>.md`

---

## Completion

After Phase 5 approval:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⏸ HANDOFF — /new-domain complete
Completed: domain implemented, reviewed and approved
Next step: run /commit when you are ready to commit and push
Waiting for: explicit /commit invocation — do not proceed automatically
DO NOT continue past this point.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
