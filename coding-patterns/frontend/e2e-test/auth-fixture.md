# E2E — Auth fixture

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
