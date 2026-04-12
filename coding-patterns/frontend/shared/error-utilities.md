# Shared — Error utilities

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## handleHttpError

Used in API functions (`api/`) to log and re-throw errors from the HTTP client. This is a pure utility — it does NOT call `notFound()` or `redirect()`. Navigation concerns are handled by the caller (pages or server actions).

```ts
// src/shared/http/errors/utils/handle-http-error.ts
import { logError } from './log-error'

export function handleHttpError(error: unknown): never {
  logError(error)
  throw error
}
```

**Rules:**
- Return type is `never` — it always throws
- No `'use server'` directive — this is a utility, not a server action
- No Next.js navigation imports — API functions must be framework-agnostic
- Error classes already have human-readable messages (set via `extractMessageFromBody` in their constructors) — callers use `error.message` directly

**Error handling by context:**

| Context | How to handle errors |
|---------|---------------------|
| **Pages (Server Components)** | Catch the error at page level: `NotFoundError` → `notFound()`, `UnauthorizedError` → `redirect('/sign-in')`, others → let bubble to `error.tsx` |
| **Server Actions** | Catch the error and return `ActionResult`: `error instanceof ApiError ? error.message : DEFAULT_ERROR_MESSAGE` |

## extractMessageFromBody

Used internally by error class constructors to extract a user-friendly message from the API error body. Server actions do not need to call this directly — use `error.message` instead.

```ts
// src/shared/http/errors/utils/extract-message-from-body.ts
export function extractMessageFromBody(
  body: unknown,
  fallback: string,
): string {
  if (
    typeof body === 'object' &&
    body !== null &&
    Object.prototype.hasOwnProperty.call(body, 'message')
  ) {
    const message = (body as Record<string, unknown>).message
    if (typeof message === 'string') {
      return message
    }
  }

  return fallback
}
```

## logError

```ts
// src/shared/http/errors/utils/log-error.ts
export function logError(error: unknown): void {
  if (error instanceof Error) {
    console.error(`[${error.name}] ${error.message}`, error)
  } else {
    console.error('[Unknown error]', error)
  }
}
```

---
