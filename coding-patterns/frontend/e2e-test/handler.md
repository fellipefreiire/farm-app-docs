# E2E — Handler pattern

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Every domain handler must follow this pattern:

```ts
// tests/mocks/handlers/<domain>.handler.ts
import { http, HttpResponse } from 'msw'

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3333'

// 1. Interfaces — match the real API response shape
interface Entity {
  id: string
  name: string
  createdAt: string
  updatedAt: string | null
}

// 2. Audit log interface — must match the shared audit log schema
interface AuditLog {
  id: string
  actorId: string            // required string, NOT nullable
  actorName: string | null
  action: string
  entity: string
  entityId: string
  changes: unknown | null
  createdAt: string
}

// 3. Exported seed IDs — used by test files to reference specific entities
export const SEED_ENTITY_ID_1 = '00000000-0000-4000-8000-000000000001'

// 4. Seed data — initial state, always valid UUIDs
const seedEntities: Entity[] = [
  {
    id: SEED_ENTITY_ID_1,
    name: 'Seed Entity',
    createdAt: new Date('2025-01-01T00:00:00.000Z').toISOString(),
    updatedAt: null,
  },
]

// 5. Pre-seeded audit logs — so audit page tests work without mutations
function seedAuditLogs(): AuditLog[] {
  return [
    {
      id: 'a0000000-0000-4000-8000-f00000000001',
      actorId: 'a0000000-0000-4000-8000-000000000001',
      actorName: 'Test User',
      action: 'CREATE',
      entity: 'Entity',
      entityId: SEED_ENTITY_ID_1,
      changes: { name: 'Seed Entity' },
      createdAt: new Date('2025-01-01T00:00:00.000Z').toISOString(),
    },
  ]
}

// 6. In-memory state — mutated by handlers, initialized with seed data
let entities: Entity[] = structuredClone(seedEntities)
let auditLogs: AuditLog[] = seedAuditLogs()

// 7. Reset function — exported but NOT called between tests (see State Management)
export function resetEntityState(): void {
  entities = structuredClone(seedEntities)
  auditLogs = seedAuditLogs()
}

// 8. Helper — record audit logs on mutations
function addAuditLog(entityId: string, action: string, changes: unknown | null = null): void {
  auditLogs.push({
    id: crypto.randomUUID(),
    actorId: 'a0000000-0000-4000-8000-000000000001',
    actorName: 'Test User',
    action,
    entity: 'Entity',
    entityId,
    changes,
    createdAt: new Date().toISOString(),
  })
}

// 9. Handlers — full CRUD + audit logs
export const entityHandlers = [
  // POST — create
  http.post(`${API_URL}/v1/entities`, async ({ request }) => { /* ... */ }),

  // GET — list (paginated)
  http.get(`${API_URL}/v1/entities`, ({ request }) => {
    const url = new URL(request.url)
    const page = Number(url.searchParams.get('page') ?? '1')
    const perPage = Number(url.searchParams.get('perPage') ?? '10')
    const query = url.searchParams.get('query') ?? ''

    // Filter, paginate, return { data, meta: { total, page, perPage } }
  }),

  // GET :id — find by ID
  // PUT :id — edit
  // DELETE :id — delete (return 204)
  // GET :id/audit-logs — cursor-based pagination
  http.get(`${API_URL}/v1/entities/:id/audit-logs`, ({ params, request }) => {
    // Return { data, meta: { count, hasNextPage, nextCursor } }
  }),
]
```

## Handler checklist

| Requirement | Details |
|---|---|
| All IDs must be valid UUIDs | Use pattern `XXXXXXXX-0000-4000-8000-XXXXXXXXXXXX`. Each domain has a unique prefix: Field=`00`, CropType=`10`, Variety=`20`, Harvest=`30`, Supplier=`40`. Next domain uses `50`. |
| Response shape must match backend | Validate against Zod schemas in `src/domains/<domain>/schemas/` |
| Audit logs must use shared schema | `{ id, actorId, actorName, action, entity, entityId, changes, createdAt }` |
| List endpoints return `meta` | `{ data, meta: { total, page, perPage } }` |
| Audit log endpoints return cursor pagination | `{ data, meta: { count, hasNextPage, nextCursor } }` |
| Pre-seed audit logs | At least one audit log per seed entity for audit page tests |
| Export seed IDs | `export const SEED_ENTITY_ID = '...'` for test file references |
| Always use `API_URL` from env | Never hardcode URLs |

---
