# Use Case Pattern

Use cases live in the application layer. They orchestrate domain logic and repository calls. They never import infrastructure types.

---

## File location

```
src/domain/<domain>/application/use-cases/<action>-<entity>.ts
src/domain/<domain>/application/use-cases/errors/<entity>-<problem>-error.ts
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), use cases move under the subdomain: `src/domain/<domain>/<subdomain>/application/use-cases/`.

One file per use case. One use case per file.

---

## Structure

```ts
import { Injectable } from '@nestjs/common'
import { left, right, type Either } from '@/core/either'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'
import { <Entity>NotFoundError } from './errors/<entity>-not-found-error'

type <Action><Entity>UseCaseRequest = {
  id: string
  field: Type
  actorId: string
}

type <Action><Entity>UseCaseResponse = Either<
  <Entity>NotFoundError,
  {
    data: <Entity>
  }
>

/** One-line docstring describing what this use case does. */
@Injectable()
export class <Action><Entity>UseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    id,
    field,
    actorId,
  }: <Action><Entity>UseCaseRequest): Promise<<Action><Entity>UseCaseResponse> {
    const entity = await this.<entity>Repository.findActiveById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    entity.update({ field }, actorId)

    await this.<entity>Repository.save(entity)

    return right({ data: entity })
  }
}
```

---

## Error classes

Domain-specific errors extend `BaseError` from core:

```ts
// src/domain/<domain>/application/use-cases/errors/<entity>-not-found-error.ts
import { BaseError } from '@/core/errors/use-case-error'

export class <Entity>NotFoundError extends BaseError {
  constructor(message = '<Entity> not found') {
    super(message, '<Entity>NotFoundError')
  }
}
```

Core errors available for reuse:

| Error | When to use |
|-------|------------|
| `ResourceNotFoundError` | Generic not found (use domain-specific when possible) |
| `NotAllowedError` | Authorization failure |
| `GenericUseCaseError` | Unexpected error with no specific type |

---

## Create use case

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type Create<Entity>UseCaseRequest = {
  name: string
  optionalField?: string
  actorId: string
}

type Create<Entity>UseCaseResponse = Either<
  null,
  {
    data: <Entity>
  }
>

/** Creates a new <entity>. */
@Injectable()
export class Create<Entity>UseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    actorId,
    ...data
  }: Create<Entity>UseCaseRequest): Promise<Create<Entity>UseCaseResponse> {
    const entity = <Entity>.create(data, actorId)

    await this.<entity>Repository.create(entity)

    return right({ data: entity })
  }
}
```

---

## List use case with pagination

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import type { PaginationMeta } from '@/core/repositories/pagination-params'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type List<Entity>sUseCaseRequest = {
  page?: number
  perPage?: number
  search?: string                        // free text — passed directly to repository
  active?: boolean
  status?: '<StatusA>' | '<StatusB>'    // domain-specific filters
  type?: ('<TypeA>' | '<TypeB>')[]      // always array — normalized in controller
  startDate?: Date
  endDate?: Date
}

type List<Entity>sUseCaseResponse = Either<
  null,
  {
    data: <Entity>[]
    meta: PaginationMeta
  }
>

/** Lists <entity>s with pagination and optional search. */
@Injectable()
export class List<Entity>sUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    page = 1,
    perPage = 20,
    search,
    active,
    status,
    type,
    startDate,
    endDate,
  }: List<Entity>sUseCaseRequest): Promise<List<Entity>sUseCaseResponse> {
    const [items, total] = await this.<entity>Repository.list({ page, perPage, search, active, status, type, startDate, endDate })

    const totalPages = Math.ceil(total / perPage)

    return right({
      data: items,
      meta: {
        total,
        perPage,
        totalPages,
        currentPage: page,
        nextPage: page < totalPages ? page + 1 : null,
        previousPage: page > 1 ? page - 1 : null,
      },
    })
  }
}
```

---

## List use case with cursor pagination

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import type { CursorPaginatedResult } from '@/core/repositories/pagination-params'
import { <Entity> } from '../../enterprise/entities/<entity>'
import { <Entity>Repository } from '../repositories/<entity>-repository'

type List<Entity>sCursorUseCaseRequest = {
  cursor?: string
  limit?: number
}

type List<Entity>sCursorUseCaseResponse = Either<
  null,
  CursorPaginatedResult<<Entity>>
>

/** Lists <entity>s with cursor-based pagination. */
@Injectable()
export class List<Entity>sCursorUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    cursor,
    limit = 20,
  }: List<Entity>sCursorUseCaseRequest): Promise<List<Entity>sCursorUseCaseResponse> {
    const result = await this.<entity>Repository.listWithCursor({ cursor, limit })
    return right(result)
  }
}
```

---

## Find by ID use case

Two variants depending on the use case's purpose:

```ts
type Find<Entity>ByIdUseCaseRequest = {
  id: string
}

type Find<Entity>ByIdUseCaseResponse = Either<
  <Entity>NotFoundError,
  {
    data: <Entity>
  }
>

/** Finds a <entity> by ID for display. */
@Injectable()
export class Find<Entity>ByIdUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({ id }: Find<Entity>ByIdUseCaseRequest): Promise<Find<Entity>ByIdUseCaseResponse> {
    const entity = await this.<entity>Repository.findActiveById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    return right({ data: entity })
  }
}
```

> **`findById()` vs `findActiveById()`** — use `findActiveById()` for standard reads (display, listings), edit, and toggle operations. Use `findById()` only when the use case needs the entity regardless of soft-delete status: delete and audit scenarios.

---

## Delete use case

The delete use case checks for domain interactions and decides between hard delete and soft delete. The decision rule is defined in `docs/rules/<domain>.md`.

```ts
type Delete<Entity>UseCaseRequest = {
  id: string
  actorId: string
}

type Delete<Entity>UseCaseResponse = Either<
  <Entity>NotFoundError,
  null
>

@Injectable()
export class Delete<Entity>UseCase {
  constructor(
    private <entity>Repository: <Entity>Repository,
    private <related>Repository: <Related>Repository, // injected to check interactions
  ) {}

  async execute({ id, actorId }: Delete<Entity>UseCaseRequest): Promise<Delete<Entity>UseCaseResponse> {
    const entity = await this.<entity>Repository.findById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    // Check for external references that block deletion
    // (what counts as "interaction" is defined per domain in docs/rules/<domain>.md)
    const hasExternalReferences = await this.<related>Repository.existsFor<Entity>(id)

    if (hasExternalReferences) {
      // Soft delete — entity must be preserved for audit/reporting
      entity.softDelete(actorId)
      await this.<entity>Repository.save(entity)
    } else {
      // Hard delete — no external references, safe to remove permanently
      // Own children (e.g. prices, stock) are deleted automatically via
      // onDelete: Cascade defined in the Prisma schema — no code needed here
      await this.<entity>Repository.delete(id)
    }

    return right(null)
  }
}
```

**Rules:**
- What counts as "an interaction" is domain-specific — document in `docs/rules/<domain>.md`
- Own children (prices, stock) are deleted automatically via `onDelete: Cascade` in the Prisma schema — no application code needed
- External references (orders, references from other domains) trigger soft delete instead
- Soft delete uses `entity.softDelete(actorId)` + `repository.save()` — same as any mutation
- Hard delete uses `repository.delete(id)` — no domain event needed (entity is gone)
- The user never sees the distinction — the endpoint always responds the same way

---

## Toggle status use case

```ts
type Toggle<Entity>StatusUseCaseRequest = {
  id: string
  actorId: string
}

type Toggle<Entity>StatusUseCaseResponse = Either<
  <Entity>NotFoundError,
  {
    data: <Entity>
  }
>

/** Toggles the active status of a <entity>. */
@Injectable()
export class Toggle<Entity>StatusUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({
    id,
    actorId,
  }: Toggle<Entity>StatusUseCaseRequest): Promise<Toggle<Entity>StatusUseCaseResponse> {
    const entity = await this.<entity>Repository.findActiveById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    entity.toggleActive(actorId)

    await this.<entity>Repository.save(entity)

    return right({ data: entity })
  }
}
```

---

## Multi-entity domains — use case with WatchedList

When an aggregate has a WatchedList, the use case only interacts with the **aggregate repository**. WatchedList sync (new items, removed items) is handled inside `repository.save()` — the use case never touches the join repository directly.

```ts
@Injectable()
export class Create<Entity>UseCase {
  constructor(
    private <entity>Repository: <Entity>Repository,
    // ← no join repository injection — sync is handled inside repository.save()
  ) {}

  async execute({ actorId, ...request }: Create<Entity>UseCaseRequest) {
    const entityId = new UniqueEntityID()  // pre-generate so children can reference it

    const relatedList = new <Entity><Related>List([
      <Related>.create({ ...request.related, <entity>Id: entityId }),
    ])

    const entity = <Entity>.create({
      ...request,
      relatedItems: relatedList,
    }, actorId, entityId)

    await this.<entity>Repository.create(entity)
    // join table rows are created inside create() — no extra call needed

    return right({ data: entity })
  }
}
```

**On edit** — reconstruct the WatchedList from the request, pass to aggregate, call `save()`:

```ts
const relatedList = new <Entity><Related>List(
  request.relatedItems.map((item) =>
    <Related>.create({ ...item, <entity>Id: entity.id })
  )
)

entity.update({ ...request, relatedItems: relatedList }, actorId)

await this.<entity>Repository.save(entity)
// save() syncs getNewItems() + getRemovedItems() atomically via $transaction — no extra call needed
```

---

## Rules

- Always `@Injectable()` — use cases are NestJS providers (this is a pragmatic trade-off: pure DDD avoids framework decorators in domain, but NestJS requires `@Injectable()` for dependency injection without custom provider factories)
- Request and response types are `type` aliases defined in the same file
- Response is always `Either<Error, { data: ... }>` — never throws
- Left side lists all possible error types — if none, use `null`
- Always include a one-line JSDoc comment describing what the use case does
- **Never import** Prisma, HTTP types (Request, Response), or NestJS decorators beyond `@Injectable()`
- **Never contain business logic** — business rules belong in the entity
- **Never call another use case** — compose at the controller level if needed
- List use cases always return `PaginationMeta` — never return unbounded arrays
- `actorId` is always passed to entity methods that mutate state (for domain events and audit trail)
- Use `findActiveById()` for standard reads, edit, and toggle operations — use `findById()` only for delete and audit scenarios that need the entity regardless of soft-delete status

---

## Anti-patterns

```ts
// ❌ business logic in use case
async execute({ price }) {
  if (price < 0) throw new Error('invalid price') // belongs in entity
}

// ❌ throws instead of Either
async execute({ id }) {
  const item = await this.repo.findById(id)
  if (!item) throw new NotFoundException() // never throw
}

// ❌ returns raw Prisma object
return right({ data: prismaRecord }) // always return domain entity

// ❌ no pagination on list
return right({ data: await this.repo.findAll() }) // always paginate

// ❌ calling another use case
constructor(
  private otherUseCase: OtherUseCase, // never inject use cases
) {}

// ❌ calling reconstitute() in use case (event never fires)
const entity = <Entity>.reconstitute(props, new UniqueEntityID()) // use create(props, actorId)

// ❌ syncing join table from use case (breaks atomicity, duplicates repository responsibility)
await this.<entity>Repository.save(entity)
await this.<entity><Related>Repository.upsertManyFor<Entity>(entity.id.toString(), items)
// ↑ use case should only call save() — join sync belongs inside the repository
```
