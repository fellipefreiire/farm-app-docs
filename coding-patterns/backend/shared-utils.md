# Shared Utils Pattern

Small, pure utility functions used across the application. No NestJS dependencies, no side effects.

---

## File locations

```
src/shared/utils/with-timeout.ts         ← race a promise against a timeout
src/shared/utils/retry-with-backoff.ts   ← retry with exponential backoff
src/shared/utils/sanitize-html.ts        ← strip all HTML tags from input
```

---

## with-timeout

Wraps a promise with a timeout. Rejects with `Error('Timeout')` if the promise doesn't resolve in time.

```ts
// src/shared/utils/with-timeout.ts
export async function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_resolve, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms),
    ),
  ])
}
```

Usage:
```ts
const result = await withTimeout(fetch(url), 5000)
```

---

## retry-with-backoff

Retries a function with exponential backoff. Defaults: 3 retries, 300ms initial delay, factor 2.

```ts
// src/shared/utils/retry-with-backoff.ts
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: {
    retries?: number       // default 3
    initialDelayMs?: number // default 300
    factor?: number         // default 2
    onRetry?: (err: unknown, attempt: number) => void
  },
): Promise<T>
```

Usage:
```ts
const data = await retryWithBackoff(() => externalApi.call(), {
  retries: 3,
  initialDelayMs: 500,
  onRetry: (err, attempt) => logger.warn(`Retry ${attempt}`, { error: err }),
})
```

---

## sanitize-html

Strips all HTML tags and attributes from a string. Used in Zod schema transforms to prevent XSS.

```ts
// src/shared/utils/sanitize-html.ts
import sanitizeHtml from 'sanitize-html'

export function sanitize(input: string) {
  return sanitizeHtml(input, {
    allowedTags: [],
    allowedAttributes: {},
  })
}
```

Usage in Zod schemas:
```ts
const schema = z.object({
  name: z.string().min(2).transform(sanitize),
  email: z.email().transform(sanitize),
})
```

---

## Rules

- Utils are pure functions — no NestJS decorators, no DI, no side effects
- Located in `src/shared/utils/` — accessible from any layer
- One function per file — keeps imports clean
- Always generic — never domain-specific logic in utils
- `sanitize()` must be applied to all user-facing string inputs in Zod schemas (via `.transform(sanitize)`)

---

## Anti-patterns

```ts
// ❌ domain logic in utils
export function calculateOrderTotal(items) { ... } // belongs in domain layer

// ❌ NestJS dependency in utils
@Injectable()
export class TimeoutService { ... } // utils are plain functions

// ❌ missing sanitize on user input
name: z.string().min(2) // always: z.string().min(2).transform(sanitize)
```
