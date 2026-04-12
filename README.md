# Claude Boilerplate

Production-ready boilerplate for building full-stack applications with **NestJS** (backend) and **Next.js** (frontend), designed to work with **Claude Code** as an AI-powered development assistant.

## What is this?

This is an opinionated project template that combines:

- **Clean Architecture + DDD** — Domain-driven design with clear layer separation (domain → application → infrastructure)
- **TDD workflow** — Test-first development enforced by coding patterns and AI skills
- **AI-assisted development** — 6 farm-app skills + superpowers plugin for workflow orchestration
- **Comprehensive documentation** — Coding patterns, business rules, and architectural decisions versioned alongside code

The boilerplate comes with a fully functional **User** domain (authentication, authorization, CRUD) and **Audit Log** domain (event-driven activity tracking) as working examples.

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

Open `CLAUDE.md` and replace the configuration variables at the top:

```
DOCS_REPO              = your-org/my-project-docs
BACKEND_REPO           = your-org/my-project-backend
FRONTEND_REPO          = your-org/my-project-frontend
PR_TARGET_BRANCH       = development
PACKAGE_MANAGER        = pnpm             ← pnpm | npm | yarn | bun
```

### 3. Verify the symlink

The `.claude` symlink should already point to `docs/.claude`. If not:

```bash
ln -s ./docs/.claude .claude
```

### 4. Install dependencies

Replace `pnpm` with your chosen package manager throughout:

```bash
cd backend && pnpm install
cd ../frontend && pnpm install
```

### 5. Generate JWT RS256 keys

The backend uses RS256 (asymmetric) signing for JWT tokens:

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

> **Never commit `.pem` files.** For production, use a secrets manager.

### 6. Set up environment variables

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Edit `backend/.env`:
- `DATABASE_URL` — PostgreSQL connection string
- `DATABASE_USERNAME` / `DATABASE_PASSWORD` / `DATABASE_NAME` — used by `docker-compose.yml` to create the PostgreSQL container
- `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` — paste the base64 keys from step 5

Edit `frontend/.env`:
- `NEXT_PUBLIC_API_URL` — Backend URL (default: `http://localhost:3333`)
- `API_URL` — Server-side backend URL (default: `http://localhost:3333`)

### 7. Start infrastructure

```bash
cd backend && docker compose up -d
```

This starts PostgreSQL and Redis containers.

### 8. Set up the database

```bash
cd backend
pnpm prisma generate
pnpm prisma migrate dev --name init
pnpm prisma db seed   # creates default admin user (admin@example.com / 123456)
```

### 9. Update frontend cookie prefix

Edit `frontend/src/shared/constants/cookies.ts` and replace the prefix with your project name:

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

Backend runs on `http://localhost:3333`, frontend on `http://localhost:3000`.

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

Frontend E2E tests use **MSW** to mock backend responses. They don't require a running backend — the Next.js server intercepts API requests via `instrumentation.ts` when `NODE_ENV=test`.

## Using with Claude Code

This boilerplate is designed to work with [Claude Code](https://claude.com/claude-code). Run it from the project root:

```bash
claude --add-dir ./backend --add-dir ./frontend
```

### Available skills

**Farm-app skills** (project-specific):

| Skill | When to use |
|-------|-------------|
| `/project-setup` | Initial project configuration |
| `/domain-discovery` | Define a new domain's rules and entities |
| `/flow-discovery` | Map user journeys and interactions |
| `/migrate` | Major dependency updates with classification |
| `/health-check` | Full project health evaluation |
| `/finish` | Prepare branch for merge/PR with doc checklist |

**Superpowers skills** (workflow engine — provided by plugin):

Planning, implementing (TDD), debugging, code review, branching, and git operations are all handled by the `superpowers` plugin. See `CLAUDE.md` for details.

### Workflow

1. Fill `docs/product.md` with your product context
2. Fill `docs/architecture.md` with your system design
3. Use `/domain-discovery` to define each domain's rules
4. Use `/flow-discovery` to map user journeys
5. Claude follows TDD (test first), coding patterns, and creates PRs targeting `development`
6. Use `/finish` when implementation is complete

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

The `docs/coding-patterns/` directory contains 101 documented pattern files organized by layer and variant:

**Backend:** entity, use-case (10 variants), repository (8 variants), controller (15 variants), events, mapper, presenter, tests (6 variants), auth, logging, env-validation, cache, decorator, health-check, mutation-testing, rate-limit, shared-utils, query-bus, domain-organization

**Frontend:** component (7 variants), page (9 variants), action, schema, api, api-route, store, hook, design-system, accessibility, e2e-test (11 variants), unit-test, error-boundary, loading, shared (8 variants), date-handling, domain-organization, audit-integration

Each pattern includes file location, code examples, rules, and anti-patterns. Claude reads these before writing any code.

## License

MIT
