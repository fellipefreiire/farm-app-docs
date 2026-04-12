# Shared — Authentication and token refresh

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

The auth system uses three cookies and two refresh mechanisms (proxy + HTTP client). All auth helpers live in `shared/http/auth.ts`.

## Cookies

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

## Auth helpers

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

## JWT decoder

```ts
// src/shared/http/decode-jwt.ts
export function decodeJwt(token: string): { exp?: number; [key: string]: unknown } | null
```

Decodes the JWT payload (base64url → JSON). Returns `null` on malformed input. Used by `isTokenExpiringSoon` and the proxy to check token expiry without verifying the signature.

## Client-side refresh

```ts
// src/shared/http/refresh-tokens.ts
export async function refreshTokens(): Promise<boolean>
```

Client-side only (returns `false` on server). Calls `POST /api/auth/refresh` which proxies to the backend and updates cookies server-side.

## Refresh response schema

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

## Proxy (`src/proxy.ts`)

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

## API route (`src/app/api/auth/refresh/route.ts`)

Server-side endpoint called by the client-side HTTP client when it needs to refresh tokens. Reads refresh + CSRF cookies, calls `refreshAccessToken()`, updates cookies.

## Sign-in action

The `signIn` server action sets all three cookies after successful authentication:
- `COOKIE_NAMES.ACCESS_TOKEN` — `maxAge: response.expiresIn`
- `COOKIE_NAMES.REFRESH_TOKEN` — `httpOnly: true`, `maxAge: response.refreshToken.expiresIn`
- `COOKIE_NAMES.CSRF_TOKEN` — `httpOnly: false`, `sameSite: 'strict'`

## CSRF protection

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
