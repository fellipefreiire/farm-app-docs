# Farm App

> **Boilerplate:** NestJS + Next.js · Clean Architecture + DDD · Prisma + PostgreSQL · pnpm · TDD

## Configuration

```
DOCS_REPO              = fellipefreiire/farm-app-docs       ← GitHub repo for issues and plans
BACKEND_REPO           = fellipefreiire/farm-app-backend
FRONTEND_REPO          = fellipefreiire/farm-app-frontend
PR_TARGET_BRANCH       = development                        ← all PRs target this branch
PRODUCTION_BRANCH      = main                               ← triggers CI/CD, never target of PRs
HEALTH_CHECK_THRESHOLD = 5                                  ← run /health-check every N features merged
PACKAGE_MANAGER        = pnpm                               ← pnpm | npm | yarn | bun
SETTINGS_FILE          = .claude/settings.json               ← symlink to docs/.claude/settings.json
```

## Permissions

- **Settings file:** `.claude/settings.json` (symlinked from `docs/.claude/settings.json`). This is the shared, version-controlled source of truth for tool permissions. New allow/deny rules should be added here so the whole team shares them.
- **Local overrides:** `.claude/settings.local.json` is auto-created by Claude Code when permissions are approved at runtime. It is gitignored and takes precedence over `settings.json`. Do not check it into version control.
- **Bash commands:** All commands must match allowed patterns in the settings file. If a compound command mixes allowed and denied prefixes (e.g., `rm -rf ... ; pnpm ...`), split into separate tool calls. Run independent calls in parallel.
- **No direct changes to `development` or `main`:** Every change, no matter how small, must go through a feature branch with an issue and PR. See Branch Strategy below.

## Workspace Structure

```
[project-root]/
  .claude → docs/.claude     ← symlink (skills live in docs, resolved from root)
  CLAUDE.md
  backend/                   ← NestJS (independent repository)
  frontend/                  ← Next.js (independent repository)
    tests/                   ← Playwright tests
  docs/                      ← independent repository (GitHub: DOCS_REPO)
    .claude/
      skills/                ← all skills versioned here
    plans/                   ← one file per active feature, deleted after done
    rules/                   ← business rules per domain
    coding-patterns/         ← patterns per artifact type
    product.md
    flows.md
    architecture.md
    api-reference.md
    glossary.md
    reminders.md
    shared-phases.md             ← Phases 3-6 shared by all implementation skills
```

## Project Structure

### Backend (`backend/src/`)

```
src/
  core/                        ← shared primitives (Entity, ValueObject, Either, UniqueEntityId, DomainEvent, QueryBus)
    entities/
    errors/
    events/
    query-bus/                 ← QueryBus (cross-domain request/reply, see coding-patterns/backend/query-bus.md)
    repositories/
    types/
  domain/
    <domain>/                    ← flat for single-entity domains
      enterprise/
        entities/              ← domain entities and value objects
        events/                ← domain events
      application/
        use-cases/             ← one file per use case
          __tests__/
          errors/              ← domain-specific use case errors
        repositories/          ← repository interfaces (I<Domain>Repository)
        queries/               ← QueryBus query contracts (cross-domain data access)
        handlers/              ← QueryBus handlers (respond to queries from other domains)
        subscribers/           ← cross-domain event subscribers (if needed)
    <domain>/                    ← subdomain folders for multi-entity domains (see coding-patterns/backend/domain-organization.md)
      <subdomain>/
        enterprise/
          entities/
          events/
        application/
          use-cases/
            __tests__/
            errors/
          repositories/
  infra/
    database/
      prisma/
        mappers/
          <domain>/            ← toDomain / toPrisma mappers
        repositories/
          <domain>/            ← Prisma repository implementations
    http/
      controllers/
        <domain>/              ← NestJS controllers
          __tests__/           ← E2E controller tests
      presenters/              ← response formatters
      pipes/                   ← Zod validation pipes
      filters/                 ← exception filters
      dtos/                    ← request/response DTOs
    events/
      <domain>/                ← event subscribers wired to NestJS
    query-bus/                 ← QueryBusModule (@Global, wires all query handlers)
    auth/                      ← JWT + CASL authorization
    cache/                     ← Redis cache
    logger/                    ← Winston logger
    env/                       ← environment variable validation (Zod)
    decorators/                ← custom NestJS decorators
  shared/
    utils/
    rate-limit/
test/                          ← shared test helpers (outside src/)
  factories/                   ← entity factories for test setup
  repositories/
    <domain>/                  ← InMemory repository implementations
  query-bus/                   ← InMemoryQueryBus for unit tests
  cache/                       ← in-memory cache stub
  cryptography/                ← fake cryptography for tests
  utils/                       ← test utility functions
  infra/                       ← infrastructure test helpers
  e2e/                         ← end-to-end test setup (supertest)
```

### Frontend (`frontend/src/`)

```
src/
  app/                         ← Next.js App Router pages
    (public)/                  ← unauthenticated routes (sign-in, etc.)
    (private)/                 ← authenticated routes
      dashboard/               ← dashboard page
      <domain>/
        [id]/                  ← detail pages
    api/                       ← Next.js API routes (e.g. auth refresh)
  domains/
    <domain>/                  ← one folder per business domain (flat for single-entity)
      actions/                 ← Next.js server actions
      api/                     ← typed fetch functions
      components/              ← React components (all with data-testid)
      schemas/                 ← Zod schemas for API responses and forms
      store/                   ← Zustand store (if needed)
    <domain>/                  ← subdomain folders for multi-entity domains (see coding-patterns/frontend/domain-organization.md)
      <subdomain>/
        actions/
        api/
        components/
        schemas/
        store/
  shared/
    components/                ← reusable UI components
      ui/                      ← design system primitives (shadcn/ui)
      fields/                  ← form field components
    hooks/                     ← shared React hooks
    http/                      ← HTTP client, error handling, base schemas
    types/                     ← global TypeScript types
    utils/                     ← shared utility functions
tests/                         ← Playwright tests (outside src/, at frontend/tests/)
```

Setup for a new project:
```bash
ln -s ./docs/.claude .claude
```

Run Claude from `[project-root]/`:
```bash
claude --add-dir ./backend --add-dir ./frontend
```

---

## Global Rules

- Everything in English: code, comments, commit messages, API responses, error messages
- Frontend is responsible for i18n — backend never returns translated strings
- Package manager: pnpm
- Conventional Commits: `feat(scope): description`
- **TDD is mandatory** — write the test before the code (RED → GREEN → REFACTOR)
- **Never create a file without first reading the corresponding files in `docs/coding-patterns/`**
- **All frontend components must include `data-testid` attributes** — never use CSS selectors or text in Playwright tests
- **Never open a PR targeting `PRODUCTION_BRANCH`** — it triggers CI/CD to production. All PRs target `PR_TARGET_BRANCH`

## Anti-Hallucination

- **Never assume** — when in doubt, stop and ask the user. Assuming is always wrong.
- Never assume implicit behavior of libraries — check official docs before using any method
- Never generate speculative code — read coding patterns first
- Never assume dependencies not listed in `package.json` — verify before importing
- Never use a library method without confirming it exists in the installed version
- Never modify a file without reading it first
- Never remove code without understanding its full impact (check all usages)
- Validate version compatibility before using features (check `package.json` versions)
- Declare assumptions explicitly to the user before proceeding
- If there is a conflict with previous decisions or patterns, stop and flag it to the user
- Never reference a field, method, or property of an existing entity without reading its file first — memory of entity shape degrades over long sessions
- Evidence over assertions — run tests and verify output, never assume something works

## Non-Negotiable Rules

| # | Category | Rule |
|---|----------|------|
| 1 | Architecture | Clean Architecture (domain → application → infra), module boundaries, SOLID, tactical DDD |
| 2 | TDD | RED → GREEN → REFACTOR. Test first, always. No exceptions. |
| 3 | Security | OWASP Top 10, input validation (Zod pipes), auth on every non-public endpoint |
| 4 | Performance | Async I/O, avoid N+1 queries, mandatory pagination on listings |
| 5 | Resilience | Either<Error, { data: ... }> in use cases |
| 6 | Quality | AAA pattern in tests, InMemoryRepository for unit tests, low cyclomatic complexity |
| 7 | Data | Versioned Prisma migrations, mappers (toDomain/toPrisma), constraints (FK, unique). Always run `prisma migrate dev` after any schema change — database is always local. |
| 8 | Events | Cross-domain side effects via Domain Events (fire-and-forget), cross-domain data queries via QueryBus (request/reply). Use cases never import repositories from other domains. |
| 9 | Secrets | Never commit `.env` files — must be in `.gitignore`. Never hardcode credentials, tokens or secrets in code. Environment variables always read from `process.env`. If a new env var is needed, add it to `.env.example` with a placeholder value. |
| 10 | Accessibility | Semantic HTML elements (`button`, `nav`, `main`, `section`, not generic `div`). All interactive elements must be keyboard-navigable. Form inputs must have associated `label` elements. Images must have `alt` text. ARIA attributes only when semantic HTML is insufficient. |
| 11 | Logging | Winston structured JSON. Levels: `error` (unrecoverable failures — needs human attention), `warn` (recoverable issues — retries, fallbacks, cache misses), `info` (business events — entity created, payment processed), `debug` (technical details — disabled in production via `LOG_LEVEL`). Required fields: `level`, `message`, `domain`, `timestamp`. Optional: `entityId`, `actorId`, `error`. Domain layer never logs. Use cases never log directly. Controllers log `error`/`warn`. Event subscribers log `error`/`info`. Never log passwords, tokens, PII, or full request bodies. Never use `console.log` in production code. |

---

## Database Migrations

Whenever `backend/prisma/schema.prisma` is modified (new model, new field, relation change, enum change):

```bash
cd backend && pnpm prisma generate      # first — regenerates the client so IDE and code see new types
cd backend && pnpm prisma migrate dev   # second — applies schema changes to local DB and records migration
```

**If `prisma migrate dev` fails with a connection error:**
1. Identify the cause — check if the database container/service is running
2. Do not proceed with implementation — ask the user to start the database first
3. Once the user confirms DB is up, re-run the migration before continuing

Never skip the migration and proceed with implementation — the Prisma client will be out of sync with the schema and cause runtime errors.

**Migration rollback:** Prisma has no `migrate down`. If a migration fails or needs to be reverted: (1) Create a new migration that undoes the changes (add what was dropped, drop what was added). (2) Never manually edit or delete migration files that were already applied. (3) For development only: `prisma migrate reset` drops and recreates the entire DB — use only as last resort and with explicit user confirmation.

**Seed data:** Seeds live in `backend/prisma/seed.ts`. Run `pnpm prisma db seed` after a fresh `migrate dev` on a clean database. Seeds must be idempotent (use `upsert`, not `create`). Never seed production data — seeds are for development and testing only. If a new entity is added, update the seed file with representative sample data.

**Redis cache after migrations:** When adding new fields to an entity, flush Redis (`redis-cli FLUSHDB`) after migration. The `findById` repository method caches full entities — cached entries from before the migration won't have the new field, causing stale data.

---

## Branch Strategy

```
main           → production. Never target of PRs. Protected.
development    → integration branch. All PRs target here (PR_TARGET_BRANCH).
BE-{n}/name    → backend feature branch, cut from development     ({n} = GitHub Issue number in DOCS_REPO)
FE-{n}/name    → frontend feature branch, cut from development     ({n} = GitHub Issue number in DOCS_REPO)
FS-{n}/name    → fullstack feature branch, same name in both repos ({n} = GitHub Issue number in DOCS_REPO)
hotfix/name    → cut from main. PR targets main, then cherry-pick to development.
```

Rules:
- Feature branches always cut from `development`, never from `main`
- Never commit directly to `main` or `development`
- Branch deleted after PR merge
- Hotfix: merge to `main` first, then cherry-pick the commit to `development`. If the cherry-pick conflicts, follow the merge conflict rules above. If the hotfix code was significantly refactored in `development`, create a manual commit that applies the same fix adapted to the `development` codebase instead of forcing the cherry-pick.
- Merge conflicts: read both sides before acting. If the conflict is in code you wrote in the current session, resolve it following the coding patterns. If the conflict involves code written by others or in a previous session, present both versions to the user and ask how to resolve. For generated files (`pnpm-lock.yaml`): accept the target branch version and run `pnpm install`. For Prisma migrations: accept the target branch and run `prisma migrate dev`.

---

## Commands

> All commands below use `pnpm` as the default. Replace with `PACKAGE_MANAGER` value if different.

```bash
# Backend
cd backend && pnpm dev
cd backend && pnpm test
cd backend && pnpm test:e2e
cd backend && stryker run --mutate <changed-files>
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev

# Frontend
cd frontend && pnpm dev
cd frontend && pnpm build
cd frontend && pnpm lint

# E2E (Playwright)
cd frontend && pnpm test:e2e
cd frontend && pnpm test
```

---

## Dependency Management

When a library update is detected (via `pnpm outdated` in `/health-check` or manually):

### Patch and Minor versions
Update freely. Run Phase 3 validation after (see Shared Phases below). If tests pass, done.

### Major versions — classify the breaking change first

**1. Read the changelog and migration guide before updating anything.**

Then classify:

| Type of breaking change | Action |
|------------------------|--------|
| Architectural change (routing system, rendering model, module system, build pipeline) | **Revert** — do not update. Create GitHub Issue to plan a dedicated migration. |
| API rename or signature change (`func.exec()` → `func.init()`) | **Update all usages** — search entire codebase, replace all occurrences, run Phase 3 |
| Deprecated function with replacement | **Update to new API** — replace all deprecated usages, run Phase 3 |

**Architectural change examples that warrant revert:**
- Next.js Pages Router → App Router
- NestJS module system changes
- Prisma query API overhaul
- Major auth library restructure

**If in doubt whether a change is architectural → revert and create an issue.** Never guess.

### After any major update
Run full Phase 3 validation. If anything fails → revert the update, investigate separately.

---

## Model Selection

| Model | When to use |
|-------|-------------|
| **Haiku** | Simple, fast tasks: reading files, searching, running commands, conformance checks |
| **Sonnet** | Most tasks: implementation, planning, code review |
| **Opus** | Architectural decisions affecting 2+ domains, all brainstorming sessions, debugging that spans 3+ files across layers, `/architecture-discussion` |

Subagent model reference:

| Subagent | Task | Model |
|----------|------|-------|
| `Explore` reading coding patterns | simple read | Haiku |
| `Explore` verifying conformance | simple comparison | Haiku |
| `Bash` running tests/commands | execution | Haiku |
| `general-purpose` implementing waves 1-3 | writing code | Sonnet |
| `general-purpose` implementing waves 4-5 | writing code | Sonnet |
| Planning agent (Phase 1) | analysis + plan | Sonnet |
| Main agent during brainstorming | design decisions | Opus |
| Main agent during hard bug fix | complex debugging | Opus |

> Model names refer to the Claude model family (Haiku, Sonnet, Opus). Always use the latest available version of each family.

---

## Entry Point

Every conversation starts here. **Never skip this.**

### Step 0 — Project setup check

Before anything else, verify the `.claude` symlink exists (`ls -la .claude`). If missing, create it: `ln -s ./docs/.claude .claude`. If `docs/.claude/` doesn't exist, ask the user to verify they are in the project root. If not in project root, ask the user to `cd` to it before continuing.

Then check if `docs/product.md` exists and has content.

- If missing or empty → invoke `/project-setup` immediately. Do not proceed until setup is complete.
- If populated → continue to Step 1.

### Step 1 — Load context

Load available docs. For each missing doc, act according to the track (see routing table below):

| Doc | Missing action |
|-----|---------------|
| `docs/product.md` | Block — ask user to fill before proceeding |
| `docs/architecture.md` | Block — ask user to fill before proceeding |
| `docs/flows.md` | Warn — continue, note the gap |
| `docs/rules/<domain>.md` | Depends on track (see below) |
| `docs/reminders.md` | Skip silently |

**Health-check counter:** read `docs/reminders.md` for the line `health-check-counter: N`. If N ≥ HEALTH_CHECK_THRESHOLD, **stop and warn the user** before proceeding:
> "You have merged N features since the last health-check. Consider running `/health-check` before continuing. Do you want to proceed anyway or run the health-check first?"

**Wait for explicit user confirmation before continuing.** The user must either approve proceeding or choose to run `/health-check` first.

The counter is incremented at the end of Phase 6 (after PR is created) in `/feature`, `/new-domain`, and `/small-change`. `/bugfix` and `/refactor` do **not** increment the counter — they don't add new functionality. It resets to 0 after `/health-check` completes.

### Step 2 — Identify track

Read the user's message and classify:

| User says | Track |
|-----------|-------|
| "define domain / what fields / what rules / how modules connect" | **Knowledge** |
| "create domain / implement from scratch" | **New Domain** |
| "add feature / new endpoint / new screen" | **Feature** |
| "change / update / rename / add field" | **Small Change** |
| "bug / broken / error / not working" | **Bug Fix** |
| "refactor / clean up / reorganize" | **Refactor** |
| "update dependency / migrate / upgrade library" | **Migration** |
| "evaluate boilerplate / review framework / check boilerplate" | **Evaluate** |
| "what's missing / check / audit / health" | **Quality** |

**Feature vs Small Change — decision rule:**

Use `/feature` when the request requires **any** of:
- A new use case
- A new endpoint (new controller)
- A new page or screen in the frontend

Use `/small-change` when the request is limited to:
- A new field on an existing entity or schema
- An adjustment to an existing use case (no new use case created)
- An adjustment to an existing component or page (no new page created)
- A validation or business rule change in existing code

When in doubt, prefer `/feature` — it includes a refinement phase that will catch scope that `/small-change` would miss.

If still ambiguous after applying the rule → ask the user to clarify before proceeding.

### Step 3 — Invoke skill

| Track | Skill | `docs/rules/<domain>.md` missing |
|-------|-------|----------------------------------|
| Knowledge | `/domain-discovery` `/flow-discovery` `/architecture-discussion` | Create as output |
| New Domain | `/new-domain` | Block — ask user to confirm running `/domain-discovery` first. Also suggest `/health-check` before starting — new domains have large architectural impact. |
| Feature | `/feature` | Block — ask user to confirm running `/domain-discovery` first. |
| Small Change | `/small-change` | Warn — continue |
| Bug Fix | `/bugfix` | Skip |
| Refactor | `/refactor` | Skip |
| Migration | `/migrate` | Skip |
| Evaluate | `/evaluate` | Skip |
| Quality | `/compliance-check` `/code-audit` `/health-check` | Warn — note gap in report |
| Project Setup | `/project-setup` | N/A — triggered automatically by Step 0 |

**When blocking for missing domain rules**, inform the user and ask for confirmation before proceeding:
> "The `<domain>` domain doesn't have documented rules in `docs/rules/<domain>.md` yet. I need to run `/domain-discovery` before proceeding. Should I start now?"

Wait for explicit user confirmation, then invoke `/domain-discovery`. After it completes and the file is created, resume the original request.

---

## Shared Phases

**See [`docs/shared-phases.md`](docs/shared-phases.md) for the full Phase 3 → Phase 6 workflow.**

These phases are used by all implementation skills (New Domain, Feature, Small Change). The phases cover:
- **Phase 3** — Validation (tests, coverage, mutation testing, type check, security audit, quality thresholds)
- **Phase 4** — Documentation (update affected docs, delete plan file)
- **Phase 5** — Code Review (self-review against coding patterns, present to user, handle feedback)
- **Phase 6** — Commit and Pull Request (fresh tests, commit, close issue, push, create PR)
