# Farm App

> NestJS + Next.js · Clean Architecture + DDD · Prisma + PostgreSQL · pnpm · TDD

## Configuration

```
DOCS_REPO              = fellipefreiire/farm-app-docs
BACKEND_REPO           = fellipefreiire/farm-app-backend
FRONTEND_REPO          = fellipefreiire/farm-app-frontend
PR_TARGET_BRANCH       = development
PRODUCTION_BRANCH      = main
USE_GITHUB_ISSUES      = true
PACKAGE_MANAGER        = pnpm
```

## Required plugins + CLIs

- `superpowers` — workflow engine (brainstorming, writing-plans, subagent-driven-development, TDD, systematic-debugging, verification-before-completion, finishing-a-development-branch, code-review)
- `claude-mem` — cross-session memory (rules: `docs/memory-rules.md`)
- `context-mode` — session state restore + generic sandbox + `/context-mode:ctx-stats`
- `context7` (MCP) — live library documentation
- `contextzip` / `tokenzip` (local CLI) — per-tool output filters (scope split: `docs/scope-split.md`)

**Not installed** (deliberate): Playwright MCP, Memory MCP, GitHub MCP, sequential-thinking — too heavy or redundant with `gh` CLI + `claude-mem`.

## Entry point

Every session starts with `superpowers:using-superpowers`. Superpowers owns
workflow, routing, and phase execution. This file only adds farm-app-specific
rules and skills.

## Shell commands

Always prefix shell commands with `tokenzip`. It routes to a specialized filter
when one exists (pnpm, tsc, vitest, playwright, prisma, git, gh, lint, next, etc.)
and passes through unchanged otherwise. Always safe. Reference: `tokenzip --help`.

## Git workflow

**Git is never an auto side effect.** No skill auto-commits, auto-pushes, or
auto-opens a PR. Integration is always explicit via `/finish`.

When implementation is complete, skills STOP and hand off. The user invokes
`/finish` when ready — which runs the farm-app doc checklist, delegates to
`superpowers:verification-before-completion`, then delegates git operations to
`superpowers:finishing-a-development-branch`.

## Non-negotiable rules

| # | Category | Rule |
|---|---|---|
| 1 | Architecture | Clean Architecture (domain → application → infra), module boundaries, SOLID, tactical DDD |
| 2 | TDD | `superpowers:test-driven-development`. RED → GREEN → REFACTOR. No exceptions. |
| 3 | Security | OWASP Top 10, input validation (Zod pipes), auth on every non-public endpoint |
| 4 | Performance | Async I/O, no N+1, mandatory pagination on listings |
| 5 | Resilience | `Either<Error, { data }>` in all use cases |
| 6 | Quality | AAA in tests, InMemoryRepository for unit tests, low cyclomatic complexity |
| 7 | Data | Versioned Prisma migrations, toDomain/toPrisma mappers, FK + unique constraints, always run `prisma migrate dev` after schema change |
| 8 | Events | Cross-domain side effects via Domain Events. Cross-domain data via QueryBus. Use cases never import repositories from other domains |
| 9 | Secrets | Never commit `.env`. Never hardcode. Read from `process.env`. New vars → `.env.example` |
| 10 | Accessibility | Semantic HTML, keyboard-navigable interactives, associated labels, alt text, ARIA only when semantic HTML insufficient |
| 11 | Logging | Winston structured JSON. Domain layer never logs. Use cases never log directly. Controllers log error/warn. No console.log in prod code. Never log secrets/PII |
| 12 | Dates | All API dates in UTC. Frontend uses `getUTC*()` always. Local-time methods only for "now" references. See `docs/coding-patterns/frontend/date-handling.md` |
| 13 | Output locale | Everything in English in code, comments, commits, API responses. Frontend owns i18n |

## Custom skills (farm-app-only, 6 total)

| Skill | Role | Why it can't be delegated |
|---|---|---|
| `/domain-discovery` | Captures business rules into `docs/rules/<domain>.md` | Nobody else knows farm-app's domains |
| `/flow-discovery` | Captures flows into `docs/flows/<domain>.md` | Nobody else knows farm-app's flows |
| `/health-check` | On-demand audit: tests, coverage, deps, architecture, doc gaps, memory hygiene | Criteria are farm-app-specific |
| `/migrate` | Classification + codebase-wide migration for a major dependency update | Classification rules are project policy |
| `/project-setup` | One-time bootstrap when cloning a new farm-app-style project | Specific to this boilerplate |
| `/finish` | Prepares branch for merge/PR. Runs farm-app doc checklist, delegates git to superpowers | Captures doc checklist (past sessions caused stale docs when skipped) |

Everything else (planning, implementing, debugging, committing, reviewing,
branching) → `superpowers` skills, invoked by `superpowers:using-superpowers`.

## Architecture invariants (short form)

- Clean Arch layers: `domain → application → infra`. Never import across layers in the wrong direction.
- Cross-domain side effects: Domain Events. Cross-domain data: QueryBus. Use cases never import another domain's repository.
- Dates: backend stores UTC, frontend always `getUTC*()`.
- Migrations: `prisma generate` then `prisma migrate dev` after every schema change. Never skip.

Full architecture: `docs/architecture.md`. Domain boundaries: `docs/rules/`. Flows: `docs/flows/<domain>.md`.

## Database migrations

```bash
cd backend && tokenzip pnpm prisma generate
cd backend && tokenzip pnpm prisma migrate dev
```

If `migrate dev` fails: stop, fix DB state, ask user before proceeding. Never
skip the migration. After adding cached fields, flush Redis (`redis-cli FLUSHDB`).

## Branch strategy

```
main         → production, protected, never target of PRs
development  → integration, target of all PRs
BE-{n}/name  → backend feature, cut from development, {n} = issue number in DOCS_REPO
FE-{n}/name  → frontend
FS-{n}/name  → fullstack, same name in both repos
hotfix/name  → cut from main, PR to main, cherry-pick to development
```

Never commit to `main` or `development` directly. Never open a PR targeting
`main`. Full flow (conflicts, merges, rebases) → `/finish`.

## Commands

```bash
# Backend (prefix with tokenzip per "Shell commands" above)
cd backend && tokenzip pnpm dev
cd backend && tokenzip pnpm test
cd backend && tokenzip pnpm test:e2e
cd backend && tokenzip pnpm prisma generate
cd backend && tokenzip pnpm prisma migrate dev

# Frontend
cd frontend && tokenzip pnpm dev
cd frontend && tokenzip pnpm build
cd frontend && tokenzip pnpm lint
cd frontend && tokenzip pnpm test:e2e
```

## Doc index

| Doc | Purpose |
|---|---|
| `docs/product.md` | Product identity, stakeholders, goals |
| `docs/architecture.md` | System overview, domains, module diagram, cross-cutting, infra, constraints |
| `docs/adr/*.md` | Architecture Decision Records (one file per decision, chronological) |
| `docs/flows/<domain>.md` | User + system flows, per domain (mirrors `rules/`) |
| `docs/rules/<domain>.md` | Canonical business rules per domain |
| `docs/coding-patterns/backend/**` | Backend patterns (layer + phase scoped, split by variant) |
| `docs/coding-patterns/frontend/**` | Frontend patterns (layer + phase scoped, split by variant) |
| `docs/verification-rules.md` | Farm-app QA rules (thresholds, architectural conformance, Stryker, Playwright troubleshooting) |
| `docs/glossary.md` | Terminology (UI ↔ code) |
| `docs/roadmap.md` | Product roadmap + Pós-MVP backlog |
| `docs/plans/<feature>.md` | Active plans (working dir) |
| `docs/memory-rules.md` | What goes in `claude-mem` at project scope |
| `docs/scope-split.md` | Which tool owns which command output |

## Anti-hallucination

- Never assume — when in doubt, stop and ask.
- Never use a library method without confirming it exists in the installed version.
- Never modify a file without reading it first.
- Never reference a field/method of an entity without reading its file — memory degrades over long sessions.
- Evidence over assertions — run tests and verify.

Enforced by `superpowers:verification-before-completion`.
