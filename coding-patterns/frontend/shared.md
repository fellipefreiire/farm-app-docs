# Shared Utilities Pattern

Shared utilities are reusable across all domains. They live in `src/shared/` and provide HTTP infrastructure, types, form helpers, and UI primitives.

---

## File locations

```
src/shared/
  http/
    api-client.ts                              ← ky HTTP client instance
    errors/
      api.error.ts                             ← base API error class
      bad-request.error.ts                     ← 400 error
      unauthorized.error.ts                    ← 401 error
      forbidden.error.ts                       ← 403 error
      not-found.error.ts                       ← 404 error
      index.ts                                 ← barrel export
      utils/
        handle-http-error.ts                   ← error handler for API functions
        extract-message-from-body.ts           ← extracts message from error body
        log-error.ts                           ← error logging
    schemas/
      pagination.schema.ts                     ← offset + cursor pagination schemas
      message-response.schema.ts               ← message-only response ({ message })
  types/
    global.d.ts                                ← global types (NextPageProps, SearchParams)
  utils/
    search-params.ts                           ← getSearchParam helper
    toast.ts                                   ← renderToast helper
  components/
    ui/                                        ← design system primitives (shadcn/ui)
    fields/                                    ← form field components (InputField, SelectField, etc.)
  constants/
    messages.ts                                ← DEFAULT_ERROR_MESSAGE
    cookies.ts                                 ← cookie name constants
  hooks/                                       ← shared React hooks
```

---

## HTTP client

The HTTP client is a pre-configured `ky` instance with JWT injection and error class mapping.

```ts
// src/shared/http/api-client.ts
import { type CookiesFn, getCookie } from 'cookies-next'
import ky from 'ky'

import { COOKIE_NAMES } from '@/shared/constants/cookies'

import {
  ApiError,
  BadRequestError,
  ForbiddenError,
  NotFoundError,
  UnauthorizedError,
} from './errors'

export const api = ky.create({
  prefixUrl: process.env.NEXT_PUBLIC_API_URL,
  hooks: {
    beforeRequest: [
      async (request) => {
        let cookieStore: CookiesFn | undefined

        if (typeof window === 'undefined') {
          const { cookies: serverCookies } = await import('next/headers')
          cookieStore = serverCookies
        }

        const accessToken = await getCookie(COOKIE_NAMES.ACCESS_TOKEN, {
          cookies: cookieStore,
        })

        if (accessToken) {
          request.headers.set('Authorization', `Bearer ${accessToken}`)
        }
      },
    ],
    afterResponse: [
      async (_request, _options, response) => {
        if (!response.ok) {
          let body: unknown = {}

          try {
            body = await response.clone().json()
          } catch {}

          if (response.status === 400) throw new BadRequestError(body)
          if (response.status === 401) throw new UnauthorizedError(body)
          if (response.status === 403) throw new ForbiddenError(body)
          if (response.status === 404) throw new NotFoundError(body)

          throw new ApiError(response.status, body)
        }

        return response
      },
    ],
  },
})
```

**Rules:**
- `beforeRequest` injects the JWT from cookies — works in both server and client components
- `afterResponse` maps HTTP status codes to custom error classes — API functions catch these via `handleHttpError`
- Cookie names are centralized in `shared/constants/cookies.ts` — replace the project prefix when setting up a new project

---

## Error classes

Custom error hierarchy for typed error handling in API functions and server actions.

```ts
// src/shared/http/errors/api.error.ts
export class ApiError extends Error {
  constructor(
    public status: number,
    public body: unknown,
    message: string = 'Request error',
  ) {
    super(message)
    this.name = 'ApiError'
  }
}
```

```ts
// src/shared/http/errors/bad-request.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class BadRequestError extends ApiError {
  constructor(body: unknown) {
    super(400, body, extractMessageFromBody(body, 'Invalid parameters'))
    this.name = 'BadRequestError'
  }
}
```

```ts
// src/shared/http/errors/unauthorized.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class UnauthorizedError extends ApiError {
  constructor(body: unknown) {
    super(401, body, extractMessageFromBody(body, 'Unauthorized'))
    this.name = 'UnauthorizedError'
  }
}
```

```ts
// src/shared/http/errors/forbidden.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class ForbiddenError extends ApiError {
  constructor(body: unknown) {
    super(403, body, extractMessageFromBody(body, 'Forbidden'))
    this.name = 'ForbiddenError'
  }
}
```

```ts
// src/shared/http/errors/not-found.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class NotFoundError extends ApiError {
  constructor(body: unknown) {
    super(404, body, extractMessageFromBody(body, 'Resource not found'))
    this.name = 'NotFoundError'
  }
}
```

```ts
// src/shared/http/errors/index.ts
export { ApiError } from './api.error'
export { BadRequestError } from './bad-request.error'
export { UnauthorizedError } from './unauthorized.error'
export { ForbiddenError } from './forbidden.error'
export { NotFoundError } from './not-found.error'
```

---

## Error utilities

### handleHttpError

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

### extractMessageFromBody

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

### logError

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

## Global types

```ts
// src/shared/types/global.d.ts
type SearchParams = Promise<Record<string, string | string[] | undefined>>

type NextPageProps<TParams extends Record<string, string> = never> = {
  searchParams: SearchParams
} & ([TParams] extends [never]
  ? Record<never, never>
  : { params: Promise<TParams> })
```

**Usage:**

```tsx
// List page — only searchParams
export default async function EntitiesPage(props: NextPageProps) { ... }

// Detail page — searchParams + route params
export default async function EntityDetailPage(
  props: NextPageProps<{ id: string }>,
) { ... }
```

`NextPageProps` wraps Next.js App Router page props with proper typing for `searchParams` and optional `params`.

---

## Utility functions

### getSearchParam

Extracts a single string value from Next.js searchParams, handling the `string | string[] | undefined` union safely.

```ts
// src/shared/utils/search-params.ts
export function getSearchParam(
  value: string | string[] | undefined,
): string | undefined {
  return typeof value === 'string' ? value : undefined
}
```

**Usage:**

```tsx
const searchParams = await props.searchParams
const page = getSearchParam(searchParams.page)
const search = getSearchParam(searchParams.search)
```

### renderToast

Dismisses a loading toast and shows a success or error toast based on the action result.

```ts
// src/shared/utils/toast.ts
import { toast } from 'sonner'

export function renderToast(
  ok: boolean,
  message: string,
  toastId: string | number,
) {
  if (ok) {
    toast.success(message, { id: toastId })
  } else {
    toast.error(message, { id: toastId })
  }
}
```

**Usage pattern (in form components):**

```tsx
const toastId = toast.loading('Creating entity...')
const res = await createEntityAction(data)
renderToast(res.ok, res.message, toastId)
```

The `id` parameter replaces the loading toast in-place — the user sees a smooth transition from loading to success/error.

---

## Constants

```ts
// src/shared/constants/messages.ts
export const DEFAULT_ERROR_MESSAGE = 'An unexpected error occurred. Please try again.'
```

Used in server actions as the fallback error message when the error is not an `ApiError`.

---

## Rules

- Shared utilities are project-wide — they never import from `domains/`
- HTTP client (`api-client.ts`) is the single point for all API calls — never use `fetch` or `ky` directly in domain code
- Error classes map 1:1 to HTTP status codes — the `afterResponse` hook in `api-client` handles the mapping
- `handleHttpError` is used in API functions (try/catch) — it logs and re-throws, no navigation logic
- Server actions catch errors and use `error.message` directly (error classes already extract the message in their constructors via `extractMessageFromBody`)
- `NextPageProps` and `getSearchParam` are used in pages — they handle Next.js App Router typing quirks
- `renderToast` is used in form components — always paired with `toast.loading` before the action call
- `DEFAULT_ERROR_MESSAGE` is used in server actions — never hardcode fallback messages

---

## Anti-patterns

```ts
// ❌ using fetch or ky directly in domain code
import ky from 'ky'
const data = await ky.get('v1/entities').json()  // use api from shared/http/api-client

// ❌ importing error classes from deep path instead of barrel
import { ApiError } from '@/shared/http/errors/api.error'  // use barrel: '@/shared/http/errors'

// ✅ correct — utils are imported by full path (no barrel for utils/)
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'
import { extractMessageFromBody } from '@/shared/http/errors/utils/extract-message-from-body'

// ❌ handling HTTP errors manually in API functions
if (response.status === 404) { notFound() }  // use handleHttpError — it centralizes all error handling

// ❌ hardcoding fallback error messages in actions
catch (err) { return { ok: false, data: null, message: 'Something went wrong' } }
// use DEFAULT_ERROR_MESSAGE from shared/constants/messages

// ❌ accessing searchParams without getSearchParam
const page = searchParams.page  // might be string[] — use getSearchParam(searchParams.page)

// ❌ silent mutations without toast feedback
const res = await createAction(data)  // missing toast.loading + renderToast
```
