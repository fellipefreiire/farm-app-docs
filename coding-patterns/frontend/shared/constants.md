# Shared — Constants

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
