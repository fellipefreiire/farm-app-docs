# Use Case — Cross-domain data access (QueryBus)

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Use cases **never** import repositories or error classes from other domains. When a use case needs data from another domain, use the QueryBus:

```ts
import { QueryBus } from '@/core/query-bus/query-bus'
import { FindHarvestQuery, type FindHarvestQueryResult }
  from '@/domain/crop/harvests/application/queries/find-harvest.query'
import { HarvestNotFoundError } from './errors/harvest-not-found-error'  // local copy

const harvest = await QueryBus.execute<FindHarvestQueryResult>(
  new FindHarvestQuery(harvestId),
)

if (!harvest) {
  return left(new HarvestNotFoundError(harvestId))
}
```

**What's allowed to import from other domains:**
- Query contracts (`application/queries/*.query.ts`) — pure data types, no dependencies
- Nothing else — no repositories, no error classes, no entities

**What's NOT cross-domain:**
- Subdomains within the same domain (e.g., crop/harvests → crop/crop-types) — direct repository import is fine
- `AuditLogRepository` — shared service, import freely

See `query-bus.md` for the full pattern, existing queries, and how to add new ones.

---

## Testing

When a use case calls `QueryBus.execute()` instead of injecting a cross-domain repository, tests use `InMemoryQueryBus` to register static responses:

```ts
import { InMemoryQueryBus } from 'test/query-bus/in-memory-query-bus'

let queryBus: InMemoryQueryBus

describe('Use Case with QueryBus', () => {
  beforeEach(() => {
    queryBus = new InMemoryQueryBus()
    // Register default response for cross-domain query
    queryBus.register('FindHarvestQuery', {
      id: 'harvest-1',
      status: 'PLANNED',
      startDate: new Date('2026-01-01'),
      expectedEndDate: new Date('2026-06-30'),
    })

    sut = new MyUseCase(myRepository)  // no cross-domain repo in constructor
  })

  afterEach(() => {
    queryBus.clear()
  })

  it('should return error when cross-domain entity not found', async () => {
    queryBus.register('FindHarvestQuery', null)  // override for this test
    const result = await sut.execute({ ... })
    expect(result.isLeft()).toBe(true)
  })
})
```

For tests querying **multiple entities by ID** (e.g., copy-schedule with source + target harvests), register a dynamic handler:

```ts
import { QueryBus } from '@/core/query-bus/query-bus'

const harvestsMap = new Map<string, FindHarvestQueryResult>()

beforeEach(() => {
  QueryBus.register('FindHarvestQuery', async (query: any) => {
    return harvestsMap.get(query.harvestId) ?? null
  })
})

afterEach(() => {
  harvestsMap.clear()
  QueryBus.clearHandlers()
})
```

**Key differences from repository tests:**
- No `InMemory<Entity>Repository` for the cross-domain entity — use `InMemoryQueryBus` instead
- Constructor args decrease (cross-domain repos removed)
- Response data is a plain object, not a domain entity
- Error imports change to local (`./errors/`) instead of cross-domain paths

See `query-bus.md` for the full testing patterns and handler test examples.

---
