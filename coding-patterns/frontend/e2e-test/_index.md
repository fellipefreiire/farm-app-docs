# Frontend E2E Test Pattern — Index

Navigation hub for farm-app frontend E2E test patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree**.

## Files

| Variant | File | When to read |
|---|---|---|
| MSW | [msw.md](msw.md) | Setting up Mock Service Worker for a suite |
| Handler pattern | [handler.md](handler.md) | Writing MSW handlers for API endpoints |
| State management | [state.md](state.md) | Test state lifecycle / cleanup |
| Auth fixture | [auth-fixture.md](auth-fixture.md) | Authenticating test users |
| Test structure | [test-structure.md](test-structure.md) | Organizing a test file |
| Selectors | [selectors.md](selectors.md) | Selector conventions (data-testid, role, etc.) |
| Playwright config | [config.md](config.md) | Configuring playwright.config.ts |
| Running tests | [running.md](running.md) | How to run tests locally and in CI |
| Adding tests for a new domain | [new-domain.md](new-domain.md) | Bootstrapping tests for a brand-new domain |
| Common patterns | [common-patterns.md](common-patterns.md) | Reusable test patterns (wait, retry, etc.) |

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
    supplier/
      manage-suppliers.e2e-spec.ts
    audit/
      view-audit-logs.e2e-spec.ts     ← one test per domain
    mocks/
      handlers/
        auth.handler.ts               ← MSW handlers per domain
        field.handler.ts
        crop-type.handler.ts
        variety.handler.ts
        harvest.handler.ts
        supplier.handler.ts
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
  └→ browser navigates to Next.js (localhost:4001 — dedicated E2E port, DEV_PORT + 1)
       └→ Next.js server renders page (SSR)
            └→ server action or page fetch calls API (localhost:3333)
                 └→ MSW intercepts the request inside the Node.js process
                      └→ handler returns mock response from in-memory state
                           └→ page renders with mock data
```

**Key insight:** MSW runs inside the Next.js server process via `instrumentation.ts`. It intercepts **server-side** HTTP requests (server actions, RSC data fetching). It does **not** intercept browser-side `fetch` calls from client components.

---

## Rules

- Always use `data-testid` — never select by CSS class, text content, or DOM structure
- One test file per domain CRUD flow
- File naming: `manage-<entities>.e2e-spec.ts` for CRUD, `<flow>.e2e-spec.ts` for other flows
- Use `test.describe.serial()` — tests within a file depend on shared MSW state
- AAA pattern: Arrange (navigate) → Act (interact) → Assert (verify)
- Use auth fixture for authenticated flows — never repeat sign-in logic
- MSW handler responses must match real API contracts
- Toast messages, validation errors, and status labels must be in **Portuguese** — check server action files for toast text, schema files for validation messages, and column files for status labels. Never use English text in test assertions for UI-visible strings.
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

// ❌ Clicking stacked-sheet-close after submit — sheet already closed via closeAll()
await page.getByTestId('stacked-sheet-close').click() // element detached from DOM

// ❌ Using English text for toasts or status labels — UI is in Portuguese
await expect(page.getByText('Field created successfully.')).toBeVisible()
// ✅ await expect(page.getByText('Talhão criado com sucesso.')).toBeVisible()

// ❌ Using getByText when text matches both cell and toast (strict mode violation)
await expect(page.getByText('Fornecedor Atualizado')).toBeVisible()
// ✅ await expect(page.getByRole('cell', { name: 'Fornecedor Atualizado' })).toBeVisible()

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
