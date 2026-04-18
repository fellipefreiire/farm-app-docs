# Claude Boilerplate

Production-ready boilerplate for full-stack apps with **NestJS** (backend) and **Next.js** (frontend), designed for **Claude Code** as AI dev assistant.

## What is this?

Opinionated project template combining:

- **Clean Architecture + DDD** — Domain-driven design, clear layer separation (domain → application → infrastructure)
- **TDD workflow** — Test-first dev enforced by coding patterns and AI skills
- **AI-assisted development** — 6 farm-app skills + superpowers plugin for workflow orchestration
- **Comprehensive documentation** — Coding patterns, business rules, architectural decisions versioned with code

Ships with functional **User** domain (auth, authorization, CRUD) and **Audit Log** domain (event-driven activity tracking) as working examples.

## Stack

| Layer | Technology |
|-------|-----------|
| Backend | NestJS 11, TypeScript, Prisma 7, PostgreSQL, Redis |
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| Auth | JWT (RS256) + CASL role-based authorization |
| Validation | Zod (both frontend and backend) |
| Testing | Vitest (unit), Playwright (E2E), Stryker (mutation) |
| UI | shadcn/ui (Radix UI), react-hook-form, Zustand |
| HTTP | ky (frontend), supertest (backend E2E) |
| Logging | Winston (structured JSON) |
| API Mocking | MSW (Mock Service Worker) for frontend E2E without backend |

## Project structure

```
project-root/
├── CLAUDE.md                ← AI instructions and project rules
├── .claude → docs/.claude   ← symlink (skills and settings)
├── backend/                 ← NestJS application
│   ├── src/
│   │   ├── core/            ← shared primitives (Entity, ValueObject, Either)
│   │   ├── domain/          ← business domains (10 domains)
│   │   ├── infra/           ← infrastructure (HTTP, database, auth, cache, logger)
│   │   └── shared/          ← shared utilities
│   ├── test/                ← test helpers (factories, in-memory repos)
│   ├── prisma/              ← schema, seeds
│   └── docker-compose.yml   ← PostgreSQL + Redis
├── frontend/                ← Next.js application
│   ├── src/
│   │   ├── app/             ← pages, layouts, API routes
│   │   ├── domains/         ← business domains (11 domains)
│   │   └── shared/          ← shared components, hooks, HTTP client
│   └── tests/               ← E2E tests, MSW mocks, fixtures
└── docs/                    ← project documentation
    ├── coding-patterns/     ← 101 coding pattern files (backend + frontend)
    ├── rules/               ← business rules per domain
    ├── flows/               ← user + system flows per domain
    ├── adr/                 ← architecture decision records
    ├── plans/               ← feature plan templates
    └── .claude/skills/      ← 6 farm-app AI skills
```

## Getting started

### 1. Clone and start fresh

```bash
git clone <this-repo> my-project
cd my-project
rm -rf .git
git init
```

### 2. Configure the boilerplate

Open `CLAUDE.md`, replace config vars at top:

```
DOCS_REPO              = your-org/my-project-docs
BACKEND_REPO           = your-org/my-project-backend
FRONTEND_REPO          = your-org/my-project-frontend
PR_TARGET_BRANCH       = development
PACKAGE_MANAGER        = pnpm             ← pnpm | npm | yarn | bun
```

### 3. Verify the symlink

`.claude` symlink should point to `docs/.claude`. If not:

```bash
ln -s ./docs/.claude .claude
```

### 4. Install dependencies

Replace `pnpm` with chosen package manager throughout:

```bash
cd backend && pnpm install
cd ../frontend && pnpm install
```

### 5. Generate JWT RS256 keys

Backend uses RS256 (asymmetric) signing for JWT tokens:

```bash
# Generate key pair
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in private.pem -out public.pem

# Encode to base64 (required for .env)
cat private.pem | base64 -w 0    # copy output → JWT_PRIVATE_KEY
cat public.pem | base64 -w 0     # copy output → JWT_PUBLIC_KEY

# Clean up
rm private.pem public.pem
```

> **Never commit `.pem` files.** Use secrets manager in production.

### 6. Set up environment variables

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Edit `backend/.env`:
- `DATABASE_URL` — PostgreSQL connection string
- `DATABASE_USERNAME` / `DATABASE_PASSWORD` / `DATABASE_NAME` — used by `docker-compose.yml` to create PostgreSQL container
- `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` — paste base64 keys from step 5

Edit `frontend/.env`:
- `NEXT_PUBLIC_API_URL` — Backend URL (default: `http://localhost:3333`)
- `API_URL` — Server-side backend URL (default: `http://localhost:3333`)

### 7. Start infrastructure

```bash
cd backend && docker compose up -d
```

Starts PostgreSQL and Redis containers.

### 8. Set up the database

```bash
cd backend
pnpm prisma generate
pnpm prisma migrate dev --name init
pnpm prisma db seed   # creates default admin user (admin@example.com / 123456)
```

### 9. Update frontend cookie prefix

Edit `frontend/src/shared/constants/cookies.ts`, replace prefix with project name:

```ts
const PROJECT_PREFIX = 'my-project' // ← your project name here
```

### 10. Start development servers

```bash
# Terminal 1 — Backend
cd backend && pnpm dev

# Terminal 2 — Frontend
cd frontend && pnpm dev
```

Backend on `http://localhost:3333`, frontend on `http://localhost:3000`.

## Running tests

### Backend

```bash
cd backend && pnpm test           # unit tests
cd backend && pnpm test:e2e       # E2E tests (requires running database)
```

### Frontend

```bash
cd frontend && pnpm test          # unit tests (vitest)
cd frontend && pnpm test:e2e      # E2E tests (playwright + MSW, no backend needed)
```

Frontend E2E uses **MSW** to mock backend responses. No running backend needed — Next.js server intercepts API requests via `instrumentation.ts` when `NODE_ENV=test`.

## Using with Claude Code

Boilerplate designed for [Claude Code](https://claude.com/claude-code). Run from project root:

```bash
claude --add-dir ./backend --add-dir ./frontend
```

### Available skills

**Farm-app skills** (project-specific):

| Skill | When to use |
|-------|-------------|
| `/project-setup` | Initial project configuration |
| `/domain-discovery` | Define new domain's rules and entities |
| `/flow-discovery` | Map user journeys and interactions |
| `/migrate` | Major dependency updates with classification |
| `/health-check` | Full project health evaluation |
| `/finish` | Prepare branch for merge/PR with doc checklist |

**Superpowers skills** (workflow engine — provided by plugin):

Planning, implementing (TDD), debugging, code review, branching, git ops — all handled by `superpowers` plugin. See `CLAUDE.md` for details.

### Workflow

1. Fill `docs/product.md` with product context
2. Fill `docs/architecture.md` with system design
3. Use `/domain-discovery` to define each domain's rules
4. Use `/flow-discovery` to map user journeys
5. Claude follows TDD (test first), coding patterns, creates PRs targeting `development`
6. Use `/finish` when implementation complete

## What's included out of the box

### Backend (working examples)

- **User domain** — Entity with roles (ADMIN/USER), soft delete, domain events
- **Authentication** — JWT access/refresh tokens with RS256 signing, token rotation
- **Authorization** — CASL role-based policies per endpoint
- **Audit log** — Event-driven activity tracking (cross-domain via subscribers)
- **Rate limiting** — Per-endpoint request throttling
- **Input validation** — Zod pipes with HTML sanitization
- **Error handling** — Either pattern in use cases, HTTP error filters
- **Logging** — Winston structured JSON with request interceptor
- **Caching** — Redis with repository abstraction

### Frontend (working examples)

- **Auth domain** — Sign-in form, sign-out action, token management
- **User domain** — Create user action, Zod schemas, Zustand dialog store
- **HTTP client** — ky instance with JWT injection and typed error classes
- **Error handling** — Per-route `error.tsx`, global `not-found.tsx`, `global-error.tsx`
- **Loading states** — Skeleton screens matching page layouts
- **E2E testing** — Playwright with MSW mocks (no backend required)
- **Unit testing** — Vitest for utilities, schemas, error classes, stores
- **Design system** — shadcn/ui with semantic color tokens, dark mode support

## Coding patterns

`docs/coding-patterns/` has 101 documented pattern files organized by layer and variant:

**Backend:** entity, use-case (10 variants), repository (8 variants), controller (15 variants), events, mapper, presenter, tests (6 variants), auth, logging, env-validation, cache, decorator, health-check, mutation-testing, rate-limit, shared-utils, query-bus, domain-organization

**Frontend:** component (7 variants), page (9 variants), action, schema, api, api-route, store, hook, design-system, accessibility, e2e-test (11 variants), unit-test, error-boundary, loading, shared (8 variants), date-handling, domain-organization, audit-integration

Each pattern has file location, code examples, rules, anti-patterns. Claude reads these before writing code.

## License

MIT