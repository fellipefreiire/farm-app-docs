# Backend Use Case Pattern — Index

Navigation hub for farm-app backend use case patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree**.

## Files

| Variant | File | When to read |
|---|---|---|
| Create | [create.md](create.md) | Implementing a create use case |
| List (offset pagination) | [list.md](list.md) | Listing with offset pagination |
| List (cursor pagination) | [list-cursor.md](list-cursor.md) | Listing with cursor pagination |
| Find by ID | [find-by-id.md](find-by-id.md) | Looking up a single entity by ID |
| Delete | [delete.md](delete.md) | Deleting an entity |
| Toggle status | [toggle-status.md](toggle-status.md) | Toggling entity status with an audit trail |
| Per-domain audit list | [audit-list.md](audit-list.md) | Paginated audit log listing per entity |
| WatchedList | [watchedlist.md](watchedlist.md) | Multi-entity domains using WatchedList |
| Cross-domain data | [cross-domain.md](cross-domain.md) | Reading data from another domain via QueryBus |

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

## Rules

- Always `@Injectable()` — use cases are NestJS providers (this is a pragmatic trade-off: pure DDD avoids framework decorators in domain, but NestJS requires `@Injectable()` for dependency injection without custom provider factories)
- Request and response types are `type` aliases defined in the same file
- Response is always `Either<Error, { data: ... }>` — never throws
- Left side lists all possible error types — if none, use `null`
- Always include a one-line JSDoc comment describing what the use case does
- **Never import** Prisma, HTTP types (Request, Response), or NestJS decorators beyond `@Injectable()`
- **Never import** repositories or error classes from other domains — use QueryBus for cross-domain data (see `query-bus.md`)
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
