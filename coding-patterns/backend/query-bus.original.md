# QueryBus Pattern

The QueryBus is an in-process request/reply mechanism for cross-domain data lookups. It decouples use cases from other domains' repositories while keeping the query synchronous. When migrating to microservices, the static bus becomes a broker (Kafka, RabbitMQ) — zero application code changes.

---

## When to use

| Need | Solution |
|------|----------|
| Use case needs data **from another domain** (e.g., schedule needs harvest dates) | **QueryBus** |
| Use case needs data **from its own domain** (e.g., purchase needs inventory inputs) | **Direct repository injection** |
| Side effect after mutation (e.g., audit log after entity created) | **DomainEvents** (see `events.md`) |
| Subscriber reacting to another domain's event | **Direct repository import** (subscribers are the cross-domain bridge) |

**Rule of thumb:** Use cases never import repositories or error classes from other domains. Subscribers can.

---

## File locations

```
src/core/query-bus/query.ts                          ← Query interface
src/core/query-bus/query-handler.ts                  ← QueryHandler interface
src/core/query-bus/query-bus.ts                      ← Static QueryBus class

src/domain/<domain>/<subdomain>/application/queries/<query-name>.query.ts   ← query contract + result type
src/domain/<domain>/<subdomain>/application/handlers/<query-name>.handler.ts ← handler implementation
src/domain/<domain>/<subdomain>/application/handlers/__tests__/<query-name>.handler.spec.ts ← handler test

src/infra/query-bus/query-bus.module.ts              ← NestJS wiring (@Global)

test/query-bus/in-memory-query-bus.ts                ← test helper
```

> **Flat domains** omit the `<subdomain>` level: `src/domain/<domain>/application/queries/`.

---

## Core primitives

The three files in `src/core/query-bus/` follow the same static-class pattern as `DomainEvents`:

```ts
// query.ts
export interface Query {
  readonly queryName: string
}

// query-handler.ts
export interface QueryHandler<TQuery extends Query = Query, TResult = unknown> {
  handle(query: TQuery): Promise<TResult>
}

// query-bus.ts
export class QueryBus {
  private static handlersMap: Record<string, (query: Query) => Promise<unknown>> = {}

  public static register(queryName: string, handler: (query: Query) => Promise<unknown>): void
  public static async execute<TResult>(query: Query): Promise<TResult>
  public static clearHandlers(): void
}
```

**Comparison with DomainEvents:**

| Aspect | DomainEvents | QueryBus |
|--------|-------------|----------|
| Direction | Fire-and-forget (one-way) | Request/reply (returns data) |
| Trigger | Entity emits after mutation | Use case calls when it needs data |
| Handlers | Multiple per event | One per query |
| Return | `void` | `Promise<TResult>` |
| Registration | By event class name | By query name string |

---

## 1. Query contract

The query contract defines the request shape and the result type. It lives in the **responding** domain (the domain that owns the data). This is a pure data contract — no business logic, no dependencies.

```ts
// src/domain/<domain>/application/queries/<query-name>.query.ts
import type { Query } from '@/core/query-bus/query'

export type Find<Entity>QueryResult = {
  id: string
  name: string
  // only the fields the consumer needs — never the full entity
} | null

export class Find<Entity>Query implements Query {
  readonly queryName = 'Find<Entity>Query'

  constructor(public readonly <entity>Id: string) {}
}
```

**Rules for query contracts:**
- Result type is a plain object or `null` — never a domain entity
- Include only the fields consumers need — this is the anti-corruption layer
- IDs are always `string` (not `UniqueEntityID`)
- Dates stay as `Date`
- The `queryName` must be unique across the entire application
- The file exports both the class and the result type (consumers import both)

**Naming convention:**
- `Find<Entity>Query` — find by ID, returns single or null
- `FindActive<Entity>Query` — find only active/non-deleted, returns single or null
- `Find<Entity>sByIdsQuery` — batch lookup, returns array

---

## 2. Handler

The handler lives in the same domain as the query. It injects the domain's repository and maps the entity to the query result type.

```ts
// src/domain/<domain>/application/handlers/<query-name>.handler.ts
import { Injectable } from '@nestjs/common'
import { QueryBus } from '@/core/query-bus/query-bus'
import type { QueryHandler } from '@/core/query-bus/query-handler'
import { <Entity>Repository } from '../repositories/<entity>-repository'
import { Find<Entity>Query, type Find<Entity>QueryResult } from '../queries/<query-name>.query'

@Injectable()
export class Find<Entity>Handler
  implements QueryHandler<Find<Entity>Query, Find<Entity>QueryResult>
{
  constructor(private <entity>Repository: <Entity>Repository) {
    QueryBus.register('Find<Entity>Query', this.handle.bind(this))
  }

  async handle(query: Find<Entity>Query): Promise<Find<Entity>QueryResult> {
    const entity = await this.<entity>Repository.findById(query.<entity>Id)

    if (!entity) return null

    return {
      id: entity.id.toString(),
      name: entity.name,
    }
  }
}
```

**Rules for handlers:**
- `@Injectable()` — registered as NestJS provider
- Self-registers in constructor via `QueryBus.register()` — same pattern as event subscribers
- Maps entity to plain result type — never returns the entity directly
- One handler per query — never reuse handlers across queries
- Handler and query live in the same domain (the data owner)

---

## 3. QueryBus Module (NestJS wiring)

```ts
// src/infra/query-bus/query-bus.module.ts
import { Global, Module } from '@nestjs/common'
import { CropDatabaseModule } from '@/infra/database/prisma/repositories/crop/crop-database.module'
// ... other database modules
import { Find<Entity>Handler } from '@/domain/<domain>/application/handlers/<query-name>.handler'

@Global()
@Module({
  imports: [
    // Database modules for domains that provide handlers
    CropDatabaseModule,
  ],
  providers: [
    // All query handlers
    Find<Entity>Handler,
  ],
})
export class QueryBusModule {}
```

**Rules:**
- `@Global()` — handlers must be instantiated at startup to self-register
- Imported in `AppModule` — alongside `EventsModule`
- Each database module is imported once — even if it provides multiple handlers
- When adding a new handler: add the provider here and import its database module if not already imported

---

## 4. Using QueryBus in a use case

```ts
import { Injectable } from '@nestjs/common'
import { QueryBus } from '@/core/query-bus/query-bus'
import { Find<Entity>Query, type Find<Entity>QueryResult }
  from '@/domain/<other-domain>/application/queries/<query-name>.query'
import { <Entity>NotFoundError } from './errors/<entity>-not-found-error'

@Injectable()
export class MyUseCase {
  constructor(
    private myRepository: MyRepository,
    // no cross-domain repository here — QueryBus is static
  ) {}

  async execute(request: Request): Promise<Response> {
    const result = await QueryBus.execute<Find<Entity>QueryResult>(
      new Find<Entity>Query(request.<entity>Id),
    )

    if (!result) {
      return left(new <Entity>NotFoundError(request.<entity>Id))
    }

    // Use result.id, result.name, etc.
  }
}
```

**Key points:**
- `QueryBus` is static — no constructor injection needed
- Import the **query class** and **result type** from the other domain's `queries/` folder
- These imports are allowed because query files are pure contracts (no dependencies)
- Error classes must be **local** — never import errors from another domain

---

## 5. Local error classes

When a use case needs an error for an entity from another domain, create a local copy:

```ts
// src/domain/schedule/application/use-cases/errors/harvest-not-found-error.ts
import { BaseError } from '@/core/errors/base-error'

export class HarvestNotFoundError extends BaseError {
  constructor(identifier: string) {
    super(`Harvest "${identifier}" not found.`, 'HarvestNotFoundError')
  }
}
```

The error `name` must match the original — error filters map by `exception.name`, so the HTTP status mapping continues to work without changes.

---

## Existing queries

| Query | Domain | Result type | Used by |
|-------|--------|-------------|---------|
| `FindHarvestQuery` | crop/harvests | `{ id, status, startDate, expectedEndDate } \| null` | schedule use cases (9) |
| `FindActiveInputQuery` | inventory/inputs | `{ id, name, active } \| null` | schedule use cases (3) |
| `FindInputQuery` | inventory/inputs | `{ id, name } \| null` | edit-schedule-operation-input |
| `FindActiveSupplierQuery` | supplier | `{ id, active } \| null` | inventory/purchases use cases (2) |
| `FindUsersByIdsQuery` | user | `{ id, name }[]` | all audit log use cases (10) |

---

## Testing

### Handler tests

Test the handler directly with an InMemory repository:

```ts
import { beforeEach, describe, expect, it } from 'vitest'
import { InMemory<Entity>Repository } from 'test/repositories/<domain>/in-memory-<entity>-repository'
import { make<Entity> } from 'test/factories/make-<entity>'
import { Find<Entity>Handler } from '../<query-name>.handler'
import { Find<Entity>Query } from '../../queries/<query-name>.query'

let sut: Find<Entity>Handler
let repository: InMemory<Entity>Repository

describe('Find<Entity>Handler', () => {
  beforeEach(() => {
    repository = new InMemory<Entity>Repository()
    sut = new Find<Entity>Handler(repository)
  })

  it('should return data when entity exists', async () => {
    const entity = make<Entity>()
    repository.items.push(entity)

    const result = await sut.handle(new Find<Entity>Query(entity.id.toString()))

    expect(result).toEqual({
      id: entity.id.toString(),
      name: entity.name,
    })
  })

  it('should return null when entity does not exist', async () => {
    const result = await sut.handle(new Find<Entity>Query('non-existent'))

    expect(result).toBeNull()
  })
})
```

### Use case tests with InMemoryQueryBus

Use cases that call `QueryBus.execute()` need query responses registered in tests:

```ts
import { InMemoryQueryBus } from 'test/query-bus/in-memory-query-bus'

let queryBus: InMemoryQueryBus

describe('My Use Case', () => {
  beforeEach(() => {
    queryBus = new InMemoryQueryBus()
    // Register static response for the query
    queryBus.register('FindHarvestQuery', {
      id: 'harvest-1',
      status: 'PLANNED',
      startDate: new Date('2026-01-01'),
      expectedEndDate: new Date('2026-06-30'),
    })

    sut = new MyUseCase(myRepository)
  })

  afterEach(() => {
    queryBus.clear()
  })

  it('should return error when entity not found', async () => {
    // Override for this specific test
    queryBus.register('FindHarvestQuery', null)

    const result = await sut.execute({ ... })

    expect(result.isLeft()).toBe(true)
  })
})
```

**For tests that need different responses per ID** (e.g., copy-schedule queries two different harvests), register a dynamic handler directly:

```ts
import { QueryBus } from '@/core/query-bus/query-bus'

const harvestsMap = new Map<string, FindHarvestQueryResult>()

beforeEach(() => {
  QueryBus.register('FindHarvestQuery', async (query: any) => {
    return harvestsMap.get(query.harvestId) ?? null
  })
})

// In test:
harvestsMap.set('harvest-1', { id: 'harvest-1', status: 'ACTIVE', ... })
harvestsMap.set('harvest-2', { id: 'harvest-2', status: 'PLANNED', ... })
```

---

## Adding a new query

When a use case needs data from another domain:

1. **Check if a query already exists** — see the table above
2. **Create the query contract** in the responding domain's `application/queries/`
3. **Create the handler** in the responding domain's `application/handlers/`
4. **Write handler tests** in `application/handlers/__tests__/`
5. **Register the handler** in `QueryBusModule` (add provider + import database module if needed)
6. **Refactor the use case** — replace repository import with `QueryBus.execute()`, create local error class if needed
7. **Update use case tests** — replace InMemory repository with InMemoryQueryBus
8. **Update HTTP module** — remove the cross-domain database module import (QueryBusModule is `@Global`)
9. **Update error filter** — change import path to local error class (if error name is unchanged, no switch-case change needed)

---

## Module boundary rules

| Import | Allowed in use cases? | Allowed in subscribers? |
|--------|----------------------|------------------------|
| Repository from own domain | Yes | Yes |
| Repository from other domain | **No** — use QueryBus | Yes (subscribers bridge domains) |
| Error class from other domain | **No** — create local copy | Yes |
| Query contract from other domain | Yes (pure data, no deps) | N/A |
| `AuditLogRepository` (shared service) | Yes | Yes |

---

## Anti-patterns

```ts
// ❌ importing repository from another domain in a use case
import { HarvestsRepository } from '@/domain/crop/harvests/application/repositories/harvests-repository'

// ❌ importing error from another domain in a use case
import { HarvestNotFoundError } from '@/domain/crop/harvests/application/use-cases/errors/harvest-not-found-error'

// ❌ returning domain entity from query handler
async handle(query: FindHarvestQuery) {
  return this.harvestsRepository.findById(query.harvestId) // never return entity
}

// ❌ injecting QueryBus in constructor (it's static)
constructor(private queryBus: QueryBus) {} // no — use QueryBus.execute() directly

// ❌ putting business logic in query handler
async handle(query: FindHarvestQuery) {
  const harvest = await this.harvestsRepository.findById(query.harvestId)
  if (harvest.status !== 'ACTIVE') throw new Error('...') // handler only maps data
}

// ❌ using QueryBus for same-domain lookups
const input = await QueryBus.execute(new FindInputQuery(id)) // just inject InputsRepository
```
