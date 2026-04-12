# Frontend Shared Pattern — Index

Navigation hub for farm-app frontend shared patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree**.

## Files

| Variant | File | When to read |
|---|---|---|
| HTTP client | [http-client.md](http-client.md) | The shared fetch wrapper (typed, authed) |
| Error classes | [error-classes.md](error-classes.md) | Typed error classes (ApiError, NetworkError, etc.) |
| Error utilities | [error-utilities.md](error-utilities.md) | Helpers for mapping / formatting errors |
| Global types | [global-types.md](global-types.md) | Types shared across domains |
| Utility functions | [utility-functions.md](utility-functions.md) | Small reusable helpers (formatters, parsers) |
| Auth + token refresh | [auth.md](auth.md) | Auth flow, cookies, token refresh, proxy, CSRF |
| Constants | [constants.md](constants.md) | Hard-coded values / enums used app-wide |

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
