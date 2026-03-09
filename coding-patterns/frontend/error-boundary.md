# Error Boundary

Error boundaries catch unhandled runtime errors in React components and prevent the entire app from crashing. Next.js App Router provides built-in support via `error.tsx` files.

---

## Route-level error boundary (`error.tsx`)

Place `error.tsx` in any route segment to catch errors thrown by that segment's `page.tsx` or child components.

```tsx
'use client'

interface ErrorProps {
  error: Error & { digest?: string }
  reset: () => void
}

export default function Error({ error, reset }: ErrorProps) {
  return (
    <div data-testid="error-boundary">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button data-testid="error-reset-button" onClick={reset}>
        Try again
      </button>
    </div>
  )
}
```

**Rules:**
- Must be a Client Component (`'use client'`)
- `reset` re-renders the route segment — use it for transient errors
- `error.digest` is a server-generated hash — never expose full stack traces to users
- Always include `data-testid` on the container and the reset button
- Style consistently with the rest of the app — use shared UI components

---

## Global error boundary (`global-error.tsx`)

Catches errors in the root layout itself. Place at `src/app/global-error.tsx`.

```tsx
'use client'

interface GlobalErrorProps {
  error: Error & { digest?: string }
  reset: () => void
}

export default function GlobalError({ error, reset }: GlobalErrorProps) {
  return (
    <html>
      <body>
        <div data-testid="global-error-boundary">
          <h2>Something went wrong</h2>
          <button data-testid="global-error-reset-button" onClick={reset}>
            Try again
          </button>
        </div>
      </body>
    </html>
  )
}
```

**Rules:**
- Must include `<html>` and `<body>` tags — it replaces the root layout when triggered
- Keep it minimal — no shared components or providers (they might be the source of the error)
- Only catches errors in the root layout — route-level errors are caught by `error.tsx`

---

## When to use each

| Scenario | File |
|----------|------|
| Error in a page or its components | `error.tsx` in the route segment |
| Error in the root layout | `global-error.tsx` at app root |
| Error in a nested layout | `error.tsx` in the parent segment |
| 404 not found | `not-found.tsx` (not an error boundary) |

---

## Placement

```
src/app/
  global-error.tsx              ← root layout errors
  (private)/
    dashboard/
      error.tsx                 ← dashboard errors
      <domain>/
        error.tsx               ← domain page errors
        [id]/
          error.tsx             ← detail page errors
```

Every route group that represents a distinct user context should have its own `error.tsx`. At minimum: one per domain route and one at the dashboard level.

---

## What NOT to do

- Never use `try/catch` in Server Components to swallow errors — let them bubble to the error boundary
- Never show raw error messages to users in production — use generic messages
- Never use class-based React error boundaries — Next.js `error.tsx` replaces them
- Never import heavy dependencies in `global-error.tsx` — keep it self-contained
