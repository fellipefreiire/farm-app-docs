# E2E Test Pattern (Playwright)

End-to-end tests that validate user flows through the browser. Always use `data-testid` selectors — never CSS selectors or text content.

---

## File locations

```
tests/
  <domain>/
    <flow>.e2e-spec.ts          ← one file per user flow
  mocks/
    handlers/
      <domain>.handler.ts       ← MSW handlers per domain
    handlers.ts                 ← aggregates all handlers
    server.ts                   ← MSW setupServer
  fixtures/
    auth.fixture.ts             ← authenticated page fixture
  helpers/
    api.helper.ts               ← seed data via API (for real backend)
  playwright.config.ts          ← Playwright config (at frontend root)
```

---

## MSW — Mock Service Worker

E2E tests run without a real backend. MSW intercepts HTTP requests at the network level inside the Next.js server process, returning mock responses.

### How it works

1. Playwright starts Next.js with `NODE_ENV=test`
2. Next.js `instrumentation.ts` detects test mode and activates MSW
3. Server actions call API functions → `ky` makes HTTP requests → MSW intercepts → returns mock response
4. The request never leaves the process — no backend needed

### instrumentation.ts

```ts
// src/instrumentation.ts
export async function register() {
  if (process.env.NODE_ENV === 'test') {
    const { server } = await import('../tests/mocks/server')
    server.listen({ onUnhandledRequest: 'warn' })
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

### Handlers

```ts
// tests/mocks/handlers/auth.handler.ts
import { http, HttpResponse } from 'msw'

const API_URL = process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3333'

export const authHandlers = [
  http.post(`${API_URL}/v1/sessions`, async ({ request }) => {
    const body = (await request.json()) as { email: string; password: string }

    if (
      body.email === 'test@example.com' &&
      body.password === 'password123'
    ) {
      return HttpResponse.json({
        accessToken: { token: 'fake-access-token', expiresIn: 3600 },
        refreshToken: { token: 'fake-refresh-token', expiresIn: 604800 },
        csrfToken: 'fake-csrf-token',
        expiresIn: 3600,
      })
    }

    return HttpResponse.json(
      { message: 'Credentials are not valid.' },
      { status: 401 },
    )
  }),
]
```

```ts
// tests/mocks/handlers.ts
import { authHandlers } from './handlers/auth.handler'

export const handlers = [...authHandlers]
```

**Rules:**
- One handler file per domain — aggregated in `handlers.ts`
- Handler responses must match the real backend API contract (same shape, same status codes)
- Use `onUnhandledRequest: 'warn'` to catch missing handlers during development
- Always use `API_URL` from env — never hardcode URLs in handlers

---

## Test structure

```ts
// tests/<domain>/<flow>.e2e-spec.ts
import { expect, test } from '@playwright/test'

test.describe('Sign In', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/sign-in')
  })

  test('should sign in with valid credentials', async ({ page }) => {
    // Act
    await page.getByTestId('auth-email-input').fill('test@example.com')
    await page.getByTestId('auth-password-input').fill('password123')
    await page.getByTestId('auth-sign-in-button').click()

    // Assert
    await page.waitForURL('/dashboard')
    await expect(page.getByTestId('dashboard-page')).toBeVisible()
  })

  test('should show error with invalid credentials', async ({ page }) => {
    await page.getByTestId('auth-email-input').fill('wrong@example.com')
    await page.getByTestId('auth-password-input').fill('wrongpass')
    await page.getByTestId('auth-sign-in-button').click()

    await expect(
      page.getByText('Credentials are not valid.'),
    ).toBeVisible()
  })
})
```

---

## Auth fixture

For authenticated flows, use the auth fixture to sign in before each test:

```ts
// tests/fixtures/auth.fixture.ts
import { test as base } from '@playwright/test'

export const test = base.extend({
  page: async ({ page }, use) => {
    await page.goto('/sign-in')
    await page.getByTestId('auth-email-input').fill(process.env.TEST_USER_EMAIL!)
    await page.getByTestId('auth-password-input').fill(process.env.TEST_USER_PASSWORD!)
    await page.getByTestId('auth-sign-in-button').click()

    await page.waitForURL('/dashboard')

    await use(page)
  },
})

export { expect } from '@playwright/test'
```

Usage:

```ts
// tests/<domain>/manage-<entity>.e2e-spec.ts
import { test, expect } from '../fixtures/auth.fixture'

test('should list entities', async ({ page }) => {
  // page is already authenticated
  await page.goto('/dashboard/<domain>')
  await expect(page.getByTestId('<domain>-page')).toBeVisible()
})
```

---

## API helper — seed test data

For tests that need pre-existing data, use the API helper to seed via the backend API. This is used when running E2E tests against a real backend.

```ts
// tests/helpers/api.helper.ts
import { request } from '@playwright/test'

export async function seedEntity(
  token: string,
  endpoint: string,
  data: Record<string, unknown>,
) {
  const context = await request.newContext({
    baseURL: process.env.API_URL,
    extraHTTPHeaders: { Authorization: `Bearer ${token}` },
  })

  const response = await context.post(endpoint, { data })
  return response.json()
}
```

---

## Selector conventions

| Element | `data-testid` pattern | Example |
|---------|----------------------|---------|
| Page container | `<domain>-page` | `orders-page` |
| Form | `<domain>-<action>-form` | `auth-sign-in-form` |
| Create button | `create-<entity>-button` | `create-order-button` |
| Form input | `<domain>-<field>-input` | `auth-email-input` |
| Form error | `<domain>-<field>-error` | `auth-email-error` |
| Submit button | `<domain>-<action>-button` | `auth-sign-in-button` |
| Table row | `<entity>-row-<identifier>` | `order-row-ORD-001` |
| Delete button | `<entity>-delete-<id>` | `order-delete-550e...` |
| Dialog | `<entity>-<action>-dialog` | `order-delete-dialog` |
| Loading skeleton | `loading-skeleton` | `loading-skeleton` |

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
  use: {
    baseURL: process.env.FRONTEND_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: 'NODE_ENV=test pnpm dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
})
```

**Key config:**
- `testMatch: '**/*.e2e-spec.ts'` — only matches E2E test files, ignores unit tests
- `NODE_ENV=test` — activates MSW via `instrumentation.ts`

---

## Rules

- Always use `data-testid` — never select by CSS class, text content, or DOM structure
- One test file per user flow, not per page
- File naming: `<flow>.e2e-spec.ts`
- AAA pattern: Arrange (navigate, seed data) → Act (interact) → Assert (verify)
- Use auth fixture for authenticated flows — never repeat sign-in logic
- Use MSW handlers to mock backend responses — tests must work without a real backend
- MSW handler responses must match real API contracts — validate with Zod schemas when in doubt
- Tests must be independent — no test should depend on another test's state
- Use `waitForURL` or `waitForSelector` — never use arbitrary `waitForTimeout`
- Screenshots on failure only — not on every test

---

## Anti-patterns

```ts
// ❌ selecting by CSS class
await page.locator('.btn-primary').click()  // use data-testid

// ❌ selecting by text for interaction
await page.getByText('Create Order').click()  // fragile — text changes with i18n

// ❌ arbitrary wait
await page.waitForTimeout(2000)  // use waitForSelector or waitForURL

// ❌ test depends on previous test
test('should edit', async ({ page }) => {
  // assumes "create" test ran first — tests must be independent
})

// ❌ seeding data through UI
test('should delete', async ({ page }) => {
  // first create via UI...  // use API helper or MSW handler instead
})

// ❌ missing data-testid on component
<button onClick={handleCreate}>Create</button>
// always: <button data-testid="create-order-button" onClick={handleCreate}>Create</button>

// ❌ hardcoding API URL in handlers
http.post('http://localhost:3333/v1/sessions', ...)  // use API_URL from env
```
