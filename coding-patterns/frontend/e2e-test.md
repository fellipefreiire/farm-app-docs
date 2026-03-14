# E2E Test Pattern (Playwright + MSW)

End-to-end tests that validate user flows through the browser using MSW to mock the backend. Always use `data-testid` selectors — never CSS selectors or text content.

---

## File locations

```
frontend/
  tests/
    auth/
      sign-in.e2e-spec.ts            ← unauthenticated flow
    field/
      manage-fields.e2e-spec.ts       ← authenticated CRUD flow
    crop/
      manage-crop-types.e2e-spec.ts
      manage-varieties.e2e-spec.ts
      manage-harvests.e2e-spec.ts
    audit/
      view-audit-logs.e2e-spec.ts
    mocks/
      handlers/
        auth.handler.ts               ← MSW handlers per domain
        field.handler.ts
        crop-type.handler.ts
        variety.handler.ts
        harvest.handler.ts
      handlers.ts                     ← aggregates all handlers
      server.ts                       ← MSW setupServer (Node.js)
    fixtures/
      auth.fixture.ts                 ← authenticated page fixture
  playwright.config.ts                ← at frontend root
```

---

## Architecture overview

```
Playwright (test process)
  └→ browser navigates to Next.js (localhost:3000)
       └→ Next.js server renders page (SSR)
            └→ server action or page fetch calls API (localhost:3333)
                 └→ MSW intercepts the request inside the Node.js process
                      └→ handler returns mock response from in-memory state
                           └→ page renders with mock data
```

**Key insight:** MSW runs inside the Next.js server process via `instrumentation.ts`. It intercepts **server-side** HTTP requests (server actions, RSC data fetching). It does **not** intercept browser-side `fetch` calls from client components.

---

## MSW — Mock Service Worker

### How it works

1. Playwright starts Next.js with `ENABLE_MSW=true`
2. `instrumentation.ts` detects `ENABLE_MSW=true` and starts MSW server
3. Server components and server actions make API calls → MSW intercepts → returns mock data
4. No real backend is needed

### instrumentation.ts

```ts
// src/instrumentation.ts
export async function register() {
  if (process.env.ENABLE_MSW === 'true' && process.env.NEXT_RUNTIME === 'nodejs') {
    const { server } = await import('../tests/mocks/server')
    server.listen({ onUnhandledRequest: 'bypass' })
  }
}
```

### MSW server

```ts
// tests/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

### Handler aggregator

```ts
// tests/mocks/handlers.ts
import { authHandlers } from './handlers/auth.handler'
import { fieldHandlers } from './handlers/field.handler'
import { cropTypeHandlers } from './handlers/crop-type.handler'
import { varietyHandlers } from './handlers/variety.handler'
import { harvestHandlers } from './handlers/harvest.handler'

export const handlers = [
  ...authHandlers,
  ...fieldHandlers,
  ...cropTypeHandlers,
  ...varietyHandlers,
  ...harvestHandlers,
]
```

---

## Handler pattern

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

### Handler checklist

| Requirement | Details |
|---|---|
| All IDs must be valid UUIDs | Use pattern `XXXXXXXX-0000-4000-8000-XXXXXXXXXXXX` |
| Response shape must match backend | Validate against Zod schemas in `src/domains/<domain>/schemas/` |
| Audit logs must use shared schema | `{ id, actorId, actorName, action, entity, entityId, changes, createdAt }` |
| List endpoints return `meta` | `{ data, meta: { total, page, perPage } }` |
| Audit log endpoints return cursor pagination | `{ data, meta: { count, hasNextPage, nextCursor } }` |
| Pre-seed audit logs | At least one audit log per seed entity for audit page tests |
| Export seed IDs | `export const SEED_ENTITY_ID = '...'` for test file references |
| Always use `API_URL` from env | Never hardcode URLs |

---

## State management

MSW handler state lives in the **Next.js server process** as module-level variables. This state persists across all tests within a Playwright run.

### Key constraints

1. **No state reset between tests** — the handler state accumulates across the entire test run
2. **Tests within a file run serially** — use `test.describe.serial()` to guarantee order
3. **Test files run serially** — `workers: 1` in playwright.config.ts prevents cross-file conflicts
4. **Seed data loads once** — when the Next.js server starts, handler modules initialize with seed data
5. **Browser-side requests are NOT intercepted** — only server-side requests go through MSW

### Implications for test design

- **Order tests carefully**: list → search → create → edit → toggle/activate → delete
- **Delete tests should run last** — they remove entities from shared state
- **Don't delete seed entities needed by other test files** — e.g., crop-type delete should not delete the seed crop type used by variety tests
- **Use `first()` or `last()` with regex testid selectors** when you don't know exact IDs (entities created by previous tests)

---

## Auth fixture

For authenticated flows, use the auth fixture. It signs in before each test:

```ts
// tests/fixtures/auth.fixture.ts
import { test as base } from '@playwright/test'

export const test = base.extend({
  page: async ({ page }, use) => {
    await page.goto('/sign-in')
    await page.getByTestId('auth-email-input').fill('test@example.com')
    await page.getByTestId('auth-password-input').fill('password123')
    await page.getByTestId('auth-sign-in-button').click()

    await page.waitForURL('/dashboard')

    await use(page)
  },
})

export { expect } from '@playwright/test'
```

**Usage:**
- Unauthenticated tests (sign-in): import from `@playwright/test`
- Authenticated tests (all CRUD flows): import from `../fixtures/auth.fixture`

---

## Test structure

### Unauthenticated flow

```ts
import { expect, test } from '@playwright/test'

test.describe('Sign In', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/sign-in')
  })

  test('should sign in with valid credentials', async ({ page }) => {
    await page.getByTestId('auth-email-input').fill('test@example.com')
    await page.getByTestId('auth-password-input').fill('password123')
    await page.getByTestId('auth-sign-in-button').click()

    await page.waitForURL('/dashboard')
    await expect(page.getByTestId('dashboard-page')).toBeVisible()
  })
})
```

### Authenticated CRUD flow

```ts
import { test, expect } from '../fixtures/auth.fixture'

test.describe.serial('Manage Entities', () => {
  test('should list entities', async ({ page }) => {
    await page.goto('/entities')
    await expect(page.getByTestId('entities-page')).toBeVisible()
    await expect(page.getByText('Seed Entity')).toBeVisible()
  })

  test('should create an entity', async ({ page }) => {
    await page.goto('/entities')
    await page.getByTestId('entity-create-button').click()
    await expect(page.getByTestId('stacked-sheet')).toBeVisible()

    await page.getByTestId('entity-name-input').fill('New Entity')
    await page.getByTestId('entity-create-submit').click()

    await expect(page.getByText('Entity created successfully.')).toBeVisible()
  })

  test('should edit an entity', async ({ page }) => {
    await page.goto('/entities')
    // Use regex + first() when exact ID is unknown
    const actionsButton = page.getByTestId(/^entity-actions-/).first()
    await actionsButton.click()

    const editButton = page.getByTestId(/^entity-edit-btn-/).first()
    await editButton.click()

    await expect(page.getByTestId('entity-edit-form')).toBeVisible()
    // ... fill and submit ...

    await expect(page.getByText('Entity updated successfully.')).toBeVisible()

    // Edit sheets do NOT auto-close — close manually
    await page.getByTestId('stacked-sheet-close').click()
    await expect(page.getByTestId('stacked-sheet')).not.toBeVisible()
  })

  test('should delete an entity', async ({ page }) => {
    await page.goto('/entities')
    const actionsButton = page.getByTestId(/^entity-actions-/).first()
    await actionsButton.click()

    const deleteButton = page.getByTestId(/^entity-delete-btn-/).first()
    await deleteButton.click()

    await expect(page.getByTestId('entity-delete-dialog')).toBeVisible()
    await page.getByTestId('entity-delete-confirm').click()
    await expect(page.getByTestId('entity-delete-dialog')).not.toBeVisible()
  })
})
```

---

## Selector conventions

Actual `data-testid` patterns used in the codebase:

| Element | Pattern | Example |
|---|---|---|
| Page container | `<domain>-page` | `fields-page`, `harvests-page` |
| Data table | `data-table` | `data-table` |
| Empty state | `empty-state` | `empty-state` |
| Search input | `query-search-input` | `query-search-input` |
| Create button | `<entity>-create-button` | `field-create-button` |
| Create form | `<entity>-create-form` | `field-create-form` |
| Create submit | `<entity>-create-submit` | `field-create-submit` |
| Edit form | `<entity>-edit-form` | `field-edit-form` |
| Edit submit | `<entity>-edit-submit` | `field-edit-submit` |
| Delete dialog | `<entity>-delete-dialog` | `field-delete-dialog` |
| Delete form | `<entity>-delete-form` | `field-delete-form` |
| Delete confirm | `<entity>-delete-confirm` | `field-delete-confirm` |
| Actions popover | `<entity>-actions-<id>` | `field-actions-00000...` |
| Edit button | `<entity>-edit-btn-<id>` | `field-edit-btn-00000...` |
| Delete button | `<entity>-delete-btn-<id>` | `field-delete-btn-00000...` |
| Toggle status | `<entity>-toggle-status-btn-<id>` | `field-toggle-status-btn-00000...` |
| Stacked sheet | `stacked-sheet` | `stacked-sheet` |
| Sheet close | `stacked-sheet-close` | `stacked-sheet-close` |
| Status cell | `<entity>-status-<id>` | `harvest-status-30000...` |
| Image cell | `<entity>-image` | `field-image`, `crop-type-image` |
| Detail page | `<entity>-detail-page` | `variety-detail-page` |
| Detail edit btn | `<entity>-edit-detail-btn` | `variety-edit-detail-btn` |

---

## Playwright config

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  testMatch: '**/*.e2e-spec.ts',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: 1,                    // serial execution — MSW state is shared
  use: {
    baseURL: process.env.FRONTEND_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: 'ENABLE_MSW=true pnpm dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
})
```

**Key settings:**
- `workers: 1` — mandatory because MSW state is shared across all tests
- `ENABLE_MSW=true` — activates MSW via `instrumentation.ts`
- `reuseExistingServer` — reuses running dev server locally, starts fresh in CI

---

## Running tests

```bash
# Run all E2E tests
cd frontend && pnpm test:e2e

# Run a specific test file
cd frontend && pnpm playwright test tests/field/manage-fields.e2e-spec.ts

# Run with UI mode (interactive debugging)
cd frontend && pnpm playwright test --ui

# Run with headed browser (see the browser)
cd frontend && pnpm playwright test --headed

# Show HTML report after run
cd frontend && pnpm playwright show-report
```

---

## Adding tests for a new domain

1. **Create the handler** at `tests/mocks/handlers/<domain>.handler.ts`:
   - Define interfaces matching the real API response
   - Add seed data with valid UUIDs
   - Pre-seed audit logs for at least one entity
   - Implement all CRUD endpoints + audit log endpoint
   - Export seed IDs and reset function

2. **Register the handler** in `tests/mocks/handlers.ts`

3. **Create the test file** at `tests/<domain>/manage-<entities>.e2e-spec.ts`:
   - Import from `../fixtures/auth.fixture`
   - Use `test.describe.serial()`
   - Order: list → search → create → validation → edit → status changes → delete
   - Delete tests LAST (removes entities from shared state)

4. **Verify components have `data-testid`** — every interactive element in the frontend must have a `data-testid` attribute following the conventions above

---

## Common patterns

### Waiting for search (debounced input)

The search input has a 300ms debounce. Wait for the filtered result instead of using `waitForTimeout`:

```ts
await searchInput.fill('North')

// Wait for a non-matching item to disappear (confirms filter applied)
await expect(page.getByText('South Field')).not.toBeVisible({ timeout: 10000 })

// Then assert the matching item is visible
await expect(page.getByText('North Field')).toBeVisible()
```

### Closing edit sheets

Edit sheets do NOT auto-close after submit. Close them manually:

```ts
await page.getByTestId('entity-edit-submit').click()
await expect(page.getByText('Entity updated successfully.')).toBeVisible()

// Close the sheet manually
await page.getByTestId('stacked-sheet-close').click()
await expect(page.getByTestId('stacked-sheet')).not.toBeVisible()
```

### Dynamic entity selection (when ID is unknown)

When entities were created or modified by previous tests:

```ts
// Use regex pattern + first()/last()
const actionsButton = page.getByTestId(/^entity-actions-/).first()
await actionsButton.click()

// For status cells
const plannedCell = page
  .locator('[data-testid^="harvest-status-"]')
  .filter({ hasText: 'Planned' })
  .first()
const testId = await plannedCell.getAttribute('data-testid')
const harvestId = testId!.replace('harvest-status-', '')
```

### Protecting seed data from other test files

When a delete test runs, avoid deleting entities used by other test files:

```ts
// ❌ Deletes the first item — might be a seed entity used elsewhere
const first = page.getByTestId(/^crop-type-actions-/).first()

// ✅ Deletes the last item — likely a non-seed entity created by a previous test
const last = page.getByTestId(/^crop-type-actions-/).last()
```

---

## Rules

- Always use `data-testid` — never select by CSS class, text content, or DOM structure
- One test file per domain CRUD flow
- File naming: `manage-<entities>.e2e-spec.ts` for CRUD, `<flow>.e2e-spec.ts` for other flows
- Use `test.describe.serial()` — tests within a file depend on shared MSW state
- AAA pattern: Arrange (navigate) → Act (interact) → Assert (verify)
- Use auth fixture for authenticated flows — never repeat sign-in logic
- MSW handler responses must match real API contracts
- Toast and validation messages must match the actual frontend text (check schemas and action files)
- Use `waitForSelector` or assertion timeouts — never use arbitrary `waitForTimeout`
- Screenshots on failure only

---

## Anti-patterns

```ts
// ❌ Selecting by CSS class
await page.locator('.btn-primary').click()

// ❌ Selecting by text for interaction (fragile with i18n)
await page.getByText('Create Order').click()

// ❌ Arbitrary wait
await page.waitForTimeout(2000)

// ❌ Assuming stacked sheet auto-closes after edit
await expect(page.getByTestId('stacked-sheet')).not.toBeVisible() // will timeout

// ❌ Deleting seed entities needed by other test files
page.getByTestId(/^crop-type-actions-/).first() // might delete shared seed data

// ❌ Missing data-testid on component
<button onClick={handleCreate}>Create</button>
// ✅ <button data-testid="entity-create-button" onClick={handleCreate}>Create</button>

// ❌ Hardcoding API URL in handlers
http.post('http://localhost:3333/v1/sessions', ...)
// ✅ http.post(`${API_URL}/v1/sessions`, ...)

// ❌ Non-UUID seed IDs
{ id: 'field-seed-001' }  // Zod schema rejects non-UUID
// ✅ { id: '00000000-0000-4000-8000-000000000001' }

// ❌ Audit log with nullable actorId
{ actorId: null }  // schema expects string
// ✅ { actorId: 'a0000000-0000-4000-8000-000000000001' }
```

---

## Known limitations

| Limitation | Impact | Workaround |
|---|---|---|
| MSW only intercepts server-side requests | Client-side `fetch` in hooks/components bypasses MSW | Use server actions for data mutations; async selects may need browser MSW in future |
| No state reset between tests | Tests share accumulated state | Use `test.describe.serial()` and order tests carefully |
| `workers: 1` required | Slower test execution | Acceptable tradeoff for state consistency |
| Edit sheets don't auto-close | Tests must close sheets manually | Click `stacked-sheet-close` after verifying toast |
