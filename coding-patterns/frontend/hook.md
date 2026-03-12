# Hook Pattern

Custom React hooks encapsulate reusable stateful logic. Hooks are client-side only — server components cannot use hooks.

---

## File locations

```
src/shared/hooks/use-<name>.ts           ← shared hooks (used across domains)
src/domains/<domain>/hooks/use-<name>.ts ← domain-specific hooks (rare)
```

Prefer shared hooks. Only create domain-specific hooks when the logic is tightly coupled to a single domain's state.

---

## Structure

```ts
// src/shared/hooks/use-debounce.ts
'use client'

import { useEffect, useState } from 'react'

export function useDebounce<T>(value: T, delayMs: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs)
    return () => clearTimeout(timer)
  }, [value, delayMs])

  return debouncedValue
}
```

---

## Pagination hook

```ts
// src/shared/hooks/use-pagination.ts
'use client'

import { useSearchParams, useRouter, usePathname } from 'next/navigation'
import { useCallback } from 'react'

interface UsePaginationOptions {
  defaultPage?: number
  defaultPerPage?: number
}

export function usePagination(options: UsePaginationOptions = {}) {
  const { defaultPage = 1, defaultPerPage = 20 } = options
  const searchParams = useSearchParams()
  const router = useRouter()
  const pathname = usePathname()

  const page = Number(searchParams.get('page')) || defaultPage
  const perPage = Number(searchParams.get('perPage')) || defaultPerPage

  const setPage = useCallback(
    (newPage: number) => {
      const params = new URLSearchParams(searchParams.toString())
      params.set('page', String(newPage))
      router.push(`${pathname}?${params.toString()}`)
    },
    [searchParams, router, pathname],
  )

  return { page, perPage, setPage }
}
```

---

## Filter navigation hook

Central hook for syncing filter state with URL searchParams. Used internally by filter components (`SearchInput`, `FilterSelect`, `SortableHeader`, `PaginationControls`).

```ts
// src/shared/hooks/use-filter-navigation.ts
'use client'

import { useCallback } from 'react'
import { useRouter, usePathname, useSearchParams } from 'next/navigation'

/** Provides URL-based filter navigation utilities. */
export function useFilterNavigation() {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()

  // Set a single param. Resets page when changing non-page params.
  const setParam = useCallback(
    (key: string, value: string | null) => {
      const params = new URLSearchParams(searchParams.toString())

      if (value === null || value === '') {
        params.delete(key)
      } else {
        params.set(key, value)
      }

      if (key !== 'page') {
        params.delete('page')
      }

      const query = params.toString()
      router.push(query ? `${pathname}?${query}` : pathname)
    },
    [searchParams, router, pathname],
  )

  // Set multiple params at once (e.g. sort + order). Single navigation.
  const setParams = useCallback(
    (entries: Record<string, string | null>) => {
      const params = new URLSearchParams(searchParams.toString())
      let resetsPage = false

      for (const [key, value] of Object.entries(entries)) {
        if (value === null || value === '') {
          params.delete(key)
        } else {
          params.set(key, value)
        }

        if (key !== 'page') {
          resetsPage = true
        }
      }

      if (resetsPage) {
        params.delete('page')
      }

      const query = params.toString()
      router.push(query ? `${pathname}?${query}` : pathname)
    },
    [searchParams, router, pathname],
  )

  // Clear all params except those in the preserve list.
  const clearAll = useCallback(
    (preserve: string[] = []) => {
      const params = new URLSearchParams()

      for (const key of preserve) {
        const value = searchParams.get(key)
        if (value) {
          params.set(key, value)
        }
      }

      const query = params.toString()
      router.push(query ? `${pathname}?${query}` : pathname)
    },
    [searchParams, router, pathname],
  )

  return { searchParams, setParam, setParams, clearAll }
}
```

**Key behaviors:**
- `setParam` and `setParams` automatically reset `page` when any non-page param changes — prevents viewing page 5 of a filtered result set that only has 1 page
- `setParams` batches multiple param changes into a single `router.push` — use for sorting (`sort` + `order` together)
- `clearAll` removes all params except those in the `preserve` list — useful for a "clear filters" button

---

## Dialog state hook

```ts
// src/shared/hooks/use-dialog.ts
'use client'

import { useState, useCallback } from 'react'

export function useDialog() {
  const [isOpen, setIsOpen] = useState(false)

  const open = useCallback(() => setIsOpen(true), [])
  const close = useCallback(() => setIsOpen(false), [])
  const toggle = useCallback(() => setIsOpen((prev) => !prev), [])

  return { isOpen, open, close, toggle }
}
```

---

## Rules

- `'use client'` directive is mandatory — hooks use React state/effects
- Prefix with `use` — required by React's rules of hooks
- One hook per file, named `use-<name>.ts`
- Return an object (not an array) when returning multiple values — easier to destructure selectively
- Keep hooks pure — no side effects outside `useEffect`
- URL state (pagination, filters, search) goes in search params via `useSearchParams` — not in React state
- Shared hooks go in `src/shared/hooks/` — domain hooks in `src/domains/<domain>/hooks/`
- Never call hooks conditionally — React rules of hooks apply

---

## Anti-patterns

```ts
// ❌ server data fetching in a hook
export function useOrders() {
  const [orders, setOrders] = useState([])
  useEffect(() => {
    fetch('/api/orders').then(...)  // use server actions or API functions instead
  }, [])
}

// ❌ missing 'use client'
import { useState } from 'react'
export function useToggle() { ... }  // will fail in server components

// ❌ storing URL state in React state
const [page, setPage] = useState(1)  // use useSearchParams

// ❌ returning array instead of object
return [value, setValue]  // hard to skip values — use { value, setValue }

// ❌ side effect outside useEffect
export function useLogger(message: string) {
  console.log(message)  // runs on every render — wrap in useEffect
}

// ❌ conditional hook call
if (isEnabled) {
  const value = useDebounce(input, 300)  // violates rules of hooks
}
```
