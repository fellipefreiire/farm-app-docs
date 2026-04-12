# Backend Tests Pattern — Index (orphan tests only)

This folder contains test patterns that do not have a matching coding pattern to co-locate with.
Most test patterns have been moved to live next to the code they test:

| Test pattern | New home |
|---|---|
| Use case tests (create, list, list-cursor, delete, toggle-status, query-bus) | Inline `## Testing` section in each `use-case/<variant>.md` |
| Mapper test | Inline `## Testing` section in `backend/mapper.md` |
| Presenter test | Inline `## Testing` section in `backend/presenter.md` |
| Shared utils test | Inline `## Testing` section in `backend/shared-utils.md` |
| Controller E2E test (generic for all variants) | `controller/_testing.md` |
| Repository cache E2E test | Inline `## Testing` section in `repository/prisma.md` |

## Orphans that live here

| File | Why it stays | When to read |
|---|---|---|
| [factory.md](factory.md) | Test data factory — not tied to a specific artifact | Building test fixtures for any test |
| [core-primitive.md](core-primitive.md) | Tests for Either, DomainEvents, WatchedList — core primitives | Testing or debugging core primitives |
| [event-subscriber.md](event-subscriber.md) | `backend/events.md` is 327L; adding would push it over the 400L split threshold. Revisit after events.md is split. | Testing event handler logic in isolation |
| [event-e2e.md](event-e2e.md) | Same reason as event-subscriber.md | Testing event flow through the wired subscriber |
| [middleware-e2e.md](middleware-e2e.md) | No middleware coding pattern exists yet | Testing request middleware |

## File locations

```
src/domain/<domain>/application/use-cases/__tests__/<action>-<entity>.spec.ts              ← unit tests (use cases)
src/infra/http/controllers/<domain>/__tests__/<action>-<entity>.controller.e2e-spec.ts     ← E2E tests (controllers)
src/domain/<domain>/application/subscribers/<source-domain>/__tests__/on-<entity>-<action>.spec.ts ← subscriber unit tests
src/infra/events/<domain>/__tests__/on-<entity>-<action>.e2e-spec.ts                       ← event E2E tests
src/core/__tests__/<primitive>.spec.ts                                                      ← core primitive tests
src/infra/database/prisma/mappers/<domain>/__tests__/prisma-<entity>-mapper.spec.ts        ← mapper tests
src/infra/http/presenters/__tests__/<entity>-presenter.spec.ts                              ← presenter tests
src/shared/utils/__tests__/<util-name>.spec.ts                                              ← shared utils tests
src/infra/database/prisma/repositories/<domain>/__tests__/prisma-<entity>-repository.e2e-spec.ts ← repository cache tests
test/e2e/<middleware>.e2e-spec.ts                                                           ← middleware tests
test/factories/make-<entity>.ts                                                             ← entity factories (shared)
test/repositories/<domain>/in-memory-<entity>-repository.ts                                ← InMemory repos (shared)
test/utils/wait-for.ts                                                                      ← async polling helper
```

---

## Rules (all tests)

- **Every new use case must have a unit test** — create `<action>-<entity>.spec.ts` in the use case's `__tests__/` directory. No exceptions.
- **Every new controller must have an E2E test** — create `<action>-<entity>.controller.e2e-spec.ts` in the controller's `__tests__/` directory. No exceptions. The test infrastructure (Prisma test DB, factories, supertest) works in isolation — no external database needed.
- Unit tests: always use `InMemory` repositories — never real DB
- E2E tests: always use `@Injectable()` factories with Prisma — never raw `prisma.create()` inline
- Always follow AAA pattern: Arrange, Act, Assert
- One behavior per `it()` — keep assertions focused
- Test error paths: not found, unauthorized, validation failure, business rule violation
- Never use `any` in test setup or assertions
- Unit test file suffix: `.spec.ts` — E2E and event test file suffix: `.e2e-spec.ts`
- `.env.test` must NOT contain `DATABASE_URL` — the E2E setup (`test/setup-e2e.ts`) creates a unique schema URL from `.env`'s `DATABASE_URL`. If `.env.test` overrides it, NestJS `ConfigModule` uses the override instead of the isolated schema, and tests run against the development database

---

## Anti-patterns

```ts
// ❌ real database in unit test
const repo = new PrismaFieldRepository(prisma) // always InMemoryRepository

// ❌ no AAA structure
it('test', async () => {
  const sut = new UseCase(repo)
  const r = await sut.execute({ name: 'x' })
  expect(r.isRight()).toBe(true)
  expect(r.value.data.name).toBe('x') // no separation, mixed concerns
})

// ❌ accessing .value before checking Either
expect(result.value.data.name).toBe('x') // check isRight() first

// ❌ no table truncation in E2E beforeEach
beforeEach(async () => {
  // missing TRUNCATE — test state bleeds between runs
})

// ❌ multiple behaviors in one test
it('should create and list', async () => { ... }) // split into two tests
```
