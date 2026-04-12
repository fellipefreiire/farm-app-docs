# E2E — MSW (Mock Service Worker)

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## How it works

1. Playwright starts Next.js with `ENABLE_MSW=true`
2. `instrumentation.ts` detects `ENABLE_MSW=true` and starts MSW server
3. Server components and server actions make API calls → MSW intercepts → returns mock data
4. No real backend is needed

## instrumentation.ts

```ts
// src/instrumentation.ts
export async function register() {
  if (process.env.ENABLE_MSW === 'true' && process.env.NEXT_RUNTIME === 'nodejs') {
    const { server } = await import('../tests/mocks/server')
    server.listen({ onUnhandledRequest: 'bypass' })
  }
}
```

## MSW server

```ts
// tests/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

## Handler aggregator

```ts
// tests/mocks/handlers.ts — add new domain handlers here
import { authHandlers } from './handlers/auth.handler'
import { fieldHandlers } from './handlers/field.handler'
import { cropTypeHandlers } from './handlers/crop-type.handler'
import { varietyHandlers } from './handlers/variety.handler'
import { harvestHandlers } from './handlers/harvest.handler'
import { supplierHandlers } from './handlers/supplier.handler'

export const handlers = [
  ...authHandlers,
  ...fieldHandlers,
  ...cropTypeHandlers,
  ...varietyHandlers,
  ...harvestHandlers,
  ...supplierHandlers,
]
```

---
