# Shared — HTTP client

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
