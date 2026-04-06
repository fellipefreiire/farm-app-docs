# [Feature / Fix / New Domain]: [Name]

> Plan written by Opus for implementation by the local model.
> **Do not modify this file during implementation** — except to update the Implementation Progress and report blocks at the bottom.
> If anything is wrong or incomplete, return to Opus for correction.

---

## Context

- **Branch:** `<branch-name>` (backend) / `<branch-name>` (frontend)
- **Domain:** `<domain-name>`
- **Scope:** backend only | frontend only | fullstack
- **GitHub Issue:** `DOCS_REPO#<n>` (or "none — USE_GITHUB_ISSUES = false")

Brief description of what this plan implements and why.

---

## Patterns to read

> The local model reads ONLY these files before implementing. No other documentation needed.

```
docs/coding-patterns/backend/use-case.md
docs/coding-patterns/backend/entity.md
docs/coding-patterns/backend/repository.md
docs/coding-patterns/backend/controller.md
docs/coding-patterns/backend/tests.md
docs/coding-patterns/backend/mapper.md          ← include if Prisma model is involved
docs/coding-patterns/backend/presenter.md       ← include if new response shape
docs/coding-patterns/backend/events.md          ← include if domain events involved
docs/coding-patterns/backend/query-bus.md       ← include if cross-domain queries needed
docs/coding-patterns/frontend/schema.md         ← include if frontend in scope
docs/coding-patterns/frontend/action.md         ← include if server actions involved
docs/coding-patterns/frontend/component.md      ← include if new components
docs/coding-patterns/frontend/page.md           ← include if new page
docs/coding-patterns/frontend/design-system.md  ← always include when frontend in scope
```

---

## Contracts

> Exact TypeScript. No approximations. The local model follows these contracts exactly.

### Backend

#### Entity

```typescript
// src/domain/<domain>/enterprise/entities/<entity>.ts
export class <Entity> extends Entity<<EntityProps>> {
  get <field>(): <Type> { return this.props.<field> }

  static create(props: <EntityProps>, id?: UniqueEntityId): Either<DomainError, <Entity>> {
    // validation
    return right(new <Entity>(props, id))
  }
}

export interface <EntityProps> {
  <field>: <Type>
  createdAt: Date
  updatedAt?: Date
}
```

#### Use Case

```typescript
// src/domain/<domain>/application/use-cases/<action>-<entity>.ts
export interface <Action><Entity>Input {
  <field>: <Type>
}

export type <Action><Entity>Output = Either<
  <ErrorA> | <ErrorB>,
  { data: <Entity> }
>

export class <Action><Entity>UseCase {
  constructor(private <entity>Repository: I<Entity>Repository) {}

  async execute(input: <Action><Entity>Input): Promise<<Action><Entity>Output> {
    // implementation
  }
}
```

#### Repository interface

```typescript
// src/domain/<domain>/application/repositories/<entity>-repository.ts
export abstract class I<Entity>Repository {
  abstract findById(id: string): Promise<<Entity> | null>
  abstract save(entity: <Entity>): Promise<void>
  abstract delete(id: string): Promise<void>
  // add other methods as needed
}
```

#### Controller

```typescript
// src/infra/http/controllers/<domain>/<action>-<entity>.controller.ts
@Controller('<route>')
export class <Action><Entity>Controller {
  constructor(private useCase: <Action><Entity>UseCase) {}

  @<HttpMethod>('<path>')
  @UseGuards(JwtAuthGuard)          // or @Public() if public endpoint
  @Roles(UserRole.<ROLE>)           // if role restriction applies
  async handle(@Body() body: <Dto>, @CurrentUser() user: UserPayload): Promise<<ResponseType>> {
    const result = await this.useCase.execute({ ... })
    if (result.isLeft()) { throw new <HttpException>() }
    return <Presenter>.toHTTP(result.value.data)
  }
}
```

#### DTO

```typescript
// src/infra/http/dtos/<action>-<entity>.dto.ts
export class <Action><Entity>Dto {
  @IsString() @IsNotEmpty()
  <field>: string

  // use class-validator decorators matching the entity invariants
}
```

### Frontend (if applicable)

#### API response schema

```typescript
// src/domains/<domain>/schemas/<entity>.schema.ts
export const <entity>Schema = z.object({
  id: z.string().uuid(),
  <field>: z.<type>(),
  createdAt: z.string().datetime(),
})

export type <Entity> = z.infer<typeof <entity>Schema>
```

#### Server action

```typescript
// src/domains/<domain>/actions/<action>-<entity>.action.ts
export async function <action><Entity>(
  data: <FormData>
): Promise<ActionResult<{ <entity>: <Entity> }>> {
  // implementation
}
```

---

## Files

### Create

- `src/domain/<domain>/enterprise/entities/<entity>.ts` — entity with business invariants
- `src/domain/<domain>/application/use-cases/<action>-<entity>.ts` — use case
- `src/domain/<domain>/application/use-cases/errors/<entity>-<problem>-error.ts` — domain errors
- `src/domain/<domain>/application/repositories/<entity>-repository.ts` — repository interface
- `src/domain/<domain>/application/use-cases/__tests__/<action>-<entity>.spec.ts` — unit tests
- `src/infra/database/prisma/mappers/<domain>/prisma-<entity>.mapper.ts` — mapper
- `src/infra/database/prisma/repositories/<domain>/prisma-<entity>.repository.ts` — Prisma implementation
- `src/infra/http/controllers/<domain>/<action>-<entity>.controller.ts` — controller
- `src/infra/http/controllers/<domain>/__tests__/<action>-<entity>.controller.e2e-spec.ts` — E2E
- `src/infra/http/presenters/<entity>.presenter.ts` — response presenter
- `test/repositories/<domain>/in-memory-<entity>-repository.ts` — in-memory repo for tests
- `test/factories/make-<entity>.ts` — entity factory for tests

### Modify

- `backend/prisma/schema.prisma` — add `<Model>` model (see Prisma section)
- `src/infra/http/controllers/<domain>/<domain>-controllers.module.ts` — register controller
- `src/app.module.ts` — register module (if new module)

### Do not touch

- `src/domain/<other-domain>/` — not in scope for this feature
- `<other file>` — reason

---

## Prisma schema (if applicable)

```prisma
model <Model> {
  id        String   @id @default(uuid())
  <field>   <Type>
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("<table_name>")
}
```

After any schema change, run in order:
```bash
cd backend && pnpm prisma generate
cd backend && pnpm prisma migrate dev
```

---

## Implementation sequence

> Follow this order exactly. Do not skip steps. Do not reorder.

1. **Domain errors** — create `<Entity><Problem>Error` classes in `application/use-cases/errors/`
2. **Entity** — implement with props interface, getters, static `create()` with Either return
3. **Repository interface** — abstract class with method signatures only
4. **In-memory repository** — for unit tests (implements the abstract class, stores in Map)
5. **Entity factory** — `make<Entity>()` helper for tests using `<Entity>.create()`
6. **Use case + unit tests** — TDD: write failing test → implement → refactor
7. **Prisma schema** — add model, run `prisma generate` and `prisma migrate dev`
8. **Mapper** — `toDomain()` and `toPrisma()` methods
9. **Prisma repository** — implements abstract class using `this.prisma.<model>`
10. **Presenter** — static `toHTTP()` method returning plain object
11. **DTO** — class-validator decorators matching entity invariants
12. **Controller + E2E tests** — TDD: write failing E2E → implement controller → refactor
13. **Module registration** — add controller to module, verify AppModule imports
14. **Frontend schema** — Zod schema matching API response shape
15. **API function** — typed fetch using the HTTP client
16. **Server action** — wraps API function, handles errors
17. **Store** — Zustand slice (if state management needed)
18. **Components** — with `data-testid` on all interactive elements
19. **Page** — integrate components, handle loading and error states

---

## Tests

### Backend unit tests

| Scenario | Input | Expected |
|----------|-------|----------|
| Happy path | valid input | `right({ data: <entity> })` |
| [Error case] | [invalid input] | `left(<EntityError>)` |
| [Edge case] | [boundary input] | [expected result] |

```typescript
// test structure
describe('<Action><Entity>UseCase', () => {
  let useCase: <Action><Entity>UseCase
  let repository: InMemory<Entity>Repository

  beforeEach(() => {
    repository = new InMemory<Entity>Repository()
    useCase = new <Action><Entity>UseCase(repository)
  })

  it('should <happy path description>', async () => {
    const result = await useCase.execute({ ... })
    expect(result.isRight()).toBe(true)
    expect(result.value.data.<field>).toEqual(...)
  })

  it('should return <ErrorName> when <condition>', async () => {
    const result = await useCase.execute({ ... })
    expect(result.isLeft()).toBe(true)
    expect(result.value).toBeInstanceOf(<ErrorName>)
  })
})
```

### Backend E2E tests

| Scenario | Method + Path | Body | Expected status |
|----------|--------------|------|-----------------|
| Happy path | `POST /route` | `{ ... }` | `201` |
| Unauthorized | `POST /route` (no token) | — | `401` |
| [Error case] | `POST /route` | `{ ... }` | `4xx` |

---

## Restrictions

- [ ] DO NOT modify `src/domain/<other-domain>/` — out of scope
- [ ] DO NOT skip `prisma generate` before `prisma migrate dev`
- [ ] DO NOT import repositories from other domains in use cases — use QueryBus
- [ ] DO NOT log in the domain layer (`src/domain/`)
- [ ] DO NOT use raw Tailwind colors in frontend — semantic tokens only
- [ ] DO NOT use `git add -A` or `git add .` — stage specific files only
- [ ] Follow async/await throughout — no callbacks or .then() chains
- [ ] All frontend components must have `data-testid` attributes

---

## Implementation Progress

> Updated by the local model during /implement.

→ Not started

---

## Notes from Opus

[Any additional observations, design decisions, trade-offs, or attention points for the implementor.]
