# E2E — Playwright config

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

const DEV_PORT = 4000
const E2E_PORT = Number(process.env.E2E_PORT ?? DEV_PORT + 1)

export default defineConfig({
  testDir: './tests',
  testMatch: '**/*.e2e-spec.ts',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  workers: 1,                    // serial execution — MSW state is shared
  use: {
    baseURL: process.env.FRONTEND_URL ?? `http://localhost:${E2E_PORT}`,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: `ENABLE_MSW=true pnpm dev --port ${E2E_PORT}`,
    port: E2E_PORT,
    reuseExistingServer: !process.env.CI,
  },
})
```

**Key settings:**
- `E2E_PORT = DEV_PORT + 1` — derived from the frontend dev port (currently 4000 → E2E on 4001). Always uses dev port + 1 to avoid conflicts with other projects. Never hardcode a fixed port. If tests reuse an existing dev server that was started **without** `ENABLE_MSW=true`, MSW won't be active and all auth-dependent tests will fail silently.
- `workers: 1` — mandatory because MSW state is shared across all tests
- `ENABLE_MSW=true` — activates MSW via `instrumentation.ts`
- `reuseExistingServer` — reuses running dev server locally, starts fresh in CI

---
