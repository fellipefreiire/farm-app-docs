# Shared Utilities Pattern

Shared utilities are reusable across all domains. They live in `src/shared/` and provide HTTP infrastructure, types, form helpers, and UI primitives.

---

## File locations

```
src/shared/
  http/
    api-client.ts                              ← ky HTTP client instance
    auth.ts                                    ← auth helpers (token check, refresh, redirect)
    decode-jwt.ts                              ← JWT payload decoder (base64url → JSON)
    refresh-tokens.ts                          ← client-side refresh via /api/auth/refresh
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
      refresh-response.schema.ts               ← refresh endpoint response schema
  types/
    global.d.ts                                ← global types (NextPageProps, SearchParams)
  utils/
    search-params.ts                           ← getSearchParam helper
    toast.ts                                   ← renderToast helper
  components/
    ui/                                        ← design system primitives (shadcn/ui)
    fields/                                    ← form field components (InputField, SelectField, etc.)
    search-input.tsx                           ← debounced search synced with URL
    filter-select.tsx                          ← select dropdown synced with URL
    sortable-header.tsx                        ← clickable column header for sorting
    pagination-controls.tsx                    ← prev/next buttons with page indicator
    table-toolbar.tsx                          ← layout container for filters above tables
    empty-state.tsx                            ← empty state with icon and message
    table-tabs.tsx                             ← tab navigation for filter presets
    preserve-params-link.tsx                   ← Link that preserves URL search params
    view-toggle.tsx                            ← list/card view toggle synced with URL
  constants/
    messages.ts                                ← DEFAULT_ERROR_MESSAGE
    cookies.ts                                 ← cookie name constants
    auth.ts                                    ← REFRESH_THRESHOLD_SECONDS
  hooks/                                       ← shared React hooks
```

---

## HTTP client

The HTTP client is a pre-configured `ky` instance with JWT injection, automatic token refresh, and error class mapping.

```ts
// src/shared/http/api-client.ts
import ky from 'ky'

import {
  ensureRefresh,
  getAccessToken,
  getCookiesFn,
  isTokenExpiringSoon,
  redirectToLogin,
} from './auth'
import {
  ApiError,
  BadRequestError,
  ForbiddenError,
  NotFoundError,
  UnauthorizedError,
} from './errors'

const isServer = typeof window === 'undefined'

export const api = ky.create({
  prefixUrl: process.env.NEXT_PUBLIC_API_URL,
  retry: {
    limit: isServer ? 0 : 1,
    statusCodes: [401],
  },
  hooks: {
    beforeRequest: [
      async (request) => {
        const cookiesFn = await getCookiesFn()
        let accessToken = await getAccessToken(cookiesFn)

        if (!isServer && accessToken && isTokenExpiringSoon(accessToken)) {
          const success = await ensureRefresh()

          if (!success) {
            redirectToLogin()
            return
          }

          accessToken = await getAccessToken(cookiesFn)
        }

        if (accessToken) {
          request.headers.set('Authorization', `Bearer ${accessToken}`)
        }
      },
    ],
    beforeRetry: [
      async ({ request }) => {
        const success = await ensureRefresh()

        if (!success) {
          redirectToLogin()
          throw new UnauthorizedError({})
        }

        const cookiesFn = await getCookiesFn()
        const accessToken = await getAccessToken(cookiesFn)

        if (accessToken) {
          request.headers.set('Authorization', `Bearer ${accessToken}`)
        }
      },
    ],
    afterResponse: [
      async (_request, _options, response, { retryCount }) => {
        if (!response.ok) {
          let body: unknown = {}

          try {
            body = await response.clone().json()
          } catch {}

          if (response.status === 401 && retryCount === 0 && !isServer) {
            return ky.retry()
          }

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
- `beforeRequest` injects the JWT from cookies — works in both server and client components. On client-side, also checks if token is expiring soon and triggers preventive refresh
- `afterResponse` maps HTTP status codes to custom error classes. On first 401 (client-side only), returns `ky.retry()` to trigger a retry with fresh tokens
- `beforeRetry` calls `ensureRefresh()` to get new tokens before retrying the request
- Server-side has `retry.limit: 0` — no retries, no refresh attempts. Server-side refresh is handled by the proxy
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

### Input masks

Mask functions for numeric form fields. **Never use `type="number"`** — always use `type="text"` with a mask and the `numeric` prop on `InputField`.

```ts
// src/shared/utils/masks.ts

/** Allows only digits (integers). Use for year, quantity, etc. */
export function maskDigitsOnly(value: string): string {
  return value.replace(/\D/g, '')
}

/** Allows digits and a single decimal point (positive floats). Use for capacity, width, etc. */
export function maskPositiveFloat(value: string): string {
  const cleaned = value.replace(/[^\d.]/g, '')
  const parts = cleaned.split('.')
  if (parts.length <= 2) return cleaned
  return parts[0] + '.' + parts.slice(1).join('')
}
```

**Usage with InputField:**

```tsx
import { maskDigitsOnly, maskPositiveFloat } from '@/shared/utils/masks'

// Integer field (year)
<InputField name="year" inputMode="numeric" maxLength={4} mask={maskDigitsOnly} numeric data-testid="..." />

// Decimal field (capacity in liters)
<InputField name="tankCapacityL" inputMode="decimal" mask={maskPositiveFloat} numeric data-testid="..." />
```

The `numeric` prop on `InputField` converts the masked string to `Number` before passing to react-hook-form, so Zod schemas can keep `z.number()` validation.

---

## Authentication and token refresh

The auth system uses three cookies and two refresh mechanisms (proxy + HTTP client). All auth helpers live in `shared/http/auth.ts`.

### Cookies

```ts
// src/shared/constants/cookies.ts
const PROJECT_PREFIX = 'farm-app'

export const COOKIE_NAMES = {
  ACCESS_TOKEN: `${PROJECT_PREFIX}_access-token`,
  REFRESH_TOKEN: `${PROJECT_PREFIX}_refresh-token`,
  CSRF_TOKEN: `${PROJECT_PREFIX}_csrf-token`,
} as const
```

| Cookie | Duration | httpOnly | Purpose |
|--------|----------|----------|---------|
| `farm-app_access-token` | 6 min | no | JWT sent as Bearer token in API requests |
| `farm-app_refresh-token` | 7 days | yes | Used to obtain new access + refresh tokens |
| `farm-app_csrf-token` | session | no | Double-submit CSRF protection for refresh endpoint |

### Auth helpers

```ts
// src/shared/http/auth.ts
```

| Function | Purpose |
|----------|---------|
| `getCookiesFn()` | Returns the cookie store — `next/headers` cookies on server, `undefined` on client (falls back to `document.cookie`) |
| `getAccessToken(cookiesFn)` | Reads access token from cookies |
| `isTokenExpiringSoon(token)` | Decodes JWT and checks if `exp` is within `REFRESH_THRESHOLD_SECONDS` (300s = 5 min) |
| `ensureRefresh()` | Calls `refreshTokens()` with a module-level mutex to prevent concurrent refreshes |
| `redirectToLogin()` | `window.location.href = '/sign-in'` (client-side only) |
| `refreshAccessToken(refreshToken, csrfToken?)` | Calls backend `POST /v1/sessions/refresh` directly, validates response with `refreshResponseSchema`, returns parsed result or `null` |

### JWT decoder

```ts
// src/shared/http/decode-jwt.ts
export function decodeJwt(token: string): { exp?: number; [key: string]: unknown } | null
```

Decodes the JWT payload (base64url → JSON). Returns `null` on malformed input. Used by `isTokenExpiringSoon` and the proxy to check token expiry without verifying the signature.

### Client-side refresh

```ts
// src/shared/http/refresh-tokens.ts
export async function refreshTokens(): Promise<boolean>
```

Client-side only (returns `false` on server). Calls `POST /api/auth/refresh` which proxies to the backend and updates cookies server-side.

### Refresh response schema

```ts
// src/shared/http/schemas/refresh-response.schema.ts
export const refreshResponseSchema = z.object({
  accessToken: z.object({
    token: z.string(),
    expiresIn: z.number(),
  }),
  refreshToken: z.object({
    token: z.string(),
    expiresIn: z.number(),
  }),
  csrfToken: z.string(),
  expiresIn: z.number(),
})
```

Used by `refreshAccessToken()` in `auth.ts` to validate the backend response. Both the proxy and the API route use `refreshAccessToken()`, so all refresh responses are validated through this schema.

### Proxy (`src/proxy.ts`)

Protects private routes and handles preventive token refresh on navigation.

```ts
// src/proxy.ts
import { COOKIE_NAMES } from '@/shared/constants/cookies'
import { isTokenExpiringSoon, refreshAccessToken } from '@/shared/http/auth'
```

Logic:
1. Public routes (`/sign-in`) → pass through
2. No access token and no refresh token → redirect to `/sign-in`
3. Access token missing or expiring → call `refreshAccessToken()` → set new cookies on response
4. Refresh fails → delete cookies, redirect to `/sign-in`
5. Access token valid → pass through

### API route (`src/app/api/auth/refresh/route.ts`)

Server-side endpoint called by the client-side HTTP client when it needs to refresh tokens. Reads refresh + CSRF cookies, calls `refreshAccessToken()`, updates cookies.

### Sign-in action

The `signIn` server action sets all three cookies after successful authentication:
- `COOKIE_NAMES.ACCESS_TOKEN` — `maxAge: response.expiresIn`
- `COOKIE_NAMES.REFRESH_TOKEN` — `httpOnly: true`, `maxAge: response.refreshToken.expiresIn`
- `COOKIE_NAMES.CSRF_TOKEN` — `httpOnly: false`, `sameSite: 'strict'`

### CSRF protection

The backend refresh endpoint (`POST /v1/sessions/refresh`) is protected by `CsrfGuard`. Both the proxy and the API route must send:
- `x-csrf-token` header with the CSRF token value
- `Cookie` header with `farm-app_csrf-token={value}`

The backend validates that the header and cookie values match. After each refresh, the backend returns a new `csrfToken` which must be saved as a cookie for the next refresh.

**Rules:**
- Cookie names are centralized in `COOKIE_NAMES` — never hardcode cookie names
- `REFRESH_THRESHOLD_SECONDS` lives in `shared/constants/auth.ts` — the threshold must always be less than the access token duration
- `refreshAccessToken()` is the single function for calling the backend refresh endpoint — used by both proxy and API route
- All refresh responses are validated with `refreshResponseSchema` — never trust unvalidated backend responses
- `ensureRefresh()` uses a module-level mutex (`refreshPromise`) to prevent concurrent refresh calls — only effective on client-side
- Server-side refresh only happens in the proxy — the HTTP client does not retry on server-side (prevents infinite loops in Server Components)

---

## Constants

```ts
// src/shared/constants/messages.ts
export const DEFAULT_ERROR_MESSAGE = 'An unexpected error occurred. Please try again.'
```

Used in server actions as the fallback error message when the error is not an `ApiError`.

```ts
// src/shared/constants/auth.ts
export const REFRESH_THRESHOLD_SECONDS = 300
```

Time in seconds before token expiry to trigger a preventive refresh (5 minutes). Used by `isTokenExpiringSoon()` in the HTTP client and proxy.

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
- Cookie names are always referenced via `COOKIE_NAMES` constant — never hardcode cookie name strings
- Auth constants (`REFRESH_THRESHOLD_SECONDS`) live in `shared/constants/auth.ts`
- `refreshAccessToken()` in `shared/http/auth.ts` is the single point for refresh calls — proxy and API route both use it
- Sign-in action must set all three cookies (access, refresh, CSRF) — missing any breaks the refresh flow

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

// ❌ hardcoding cookie names
request.cookies.get('farm-app_access-token')  // use COOKIE_NAMES.ACCESS_TOKEN

// ❌ calling backend refresh endpoint directly in proxy or API route
const response = await fetch(`${process.env.API_URL}/v1/sessions/refresh`, ...)
// use refreshAccessToken() from shared/http/auth — it validates with Zod schema

// ❌ setting only access and refresh cookies on sign-in (missing CSRF)
await setCookie(COOKIE_NAMES.ACCESS_TOKEN, ...)
await setCookie(COOKIE_NAMES.REFRESH_TOKEN, ...)
// must also set COOKIE_NAMES.CSRF_TOKEN — without it, refresh flow breaks

// ❌ refreshing tokens on server-side in the HTTP client
if (isServer) { await ensureRefresh() }  // server-side refresh is handled by proxy only
```
