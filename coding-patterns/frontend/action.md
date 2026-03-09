# Server Action Pattern

Server actions are the bridge between UI components and API functions. They run on the server, validate input, call the API, invalidate cache, and return a standardized result.

---

## File locations

```
src/domains/<domain>/actions/create-<entity>.ts
src/domains/<domain>/actions/edit-<entity>.ts
src/domains/<domain>/actions/delete-<entity>.ts
src/domains/<domain>/actions/toggle-<entity>-status.ts
src/domains/<domain>/actions/index.ts                        ← barrel export
src/shared/constants/messages.ts                             ← DEFAULT_ERROR_MESSAGE
```

One action per operation. One file per action.

---

## Shared types and constants

The `ActionResult<T>` type is declared globally in `src/shared/types/global.d.ts` — no import needed:

```ts
type ActionResult<T> =
  | { ok: true; data: T; message: string }
  | { ok: false; data: null; message: string }
```

```ts
// src/shared/constants/messages.ts
export const DEFAULT_ERROR_MESSAGE = 'An unexpected error occurred. Please try again.'
```

---

## Cache invalidation (Next.js 16+)

Next.js 16 introduces a new cache system built around the `'use cache'` directive and three companion functions:

| Function | Where | Purpose |
|----------|-------|---------|
| `cacheTag(tag)` | Inside `'use cache'` functions/components | Labels cache entries for targeted invalidation |
| `cacheLife(profile)` | Inside `'use cache'` functions/components | Controls cache duration (stale, revalidate, expire) |
| `updateTag(tag)` | Server Actions only | Immediately expires cache — blocks until fresh data is ready (read-your-own-writes) |
| `revalidateTag(tag, profile)` | Server Actions and Route Handlers | Marks tag as stale — serves stale content while revalidating in background |

### When to use which

| Scenario | Use |
|----------|-----|
| User creates/edits/deletes — must see their own change | `updateTag('tag')` |
| Background refresh is acceptable (webhooks, cron) | `revalidateTag('tag', 'max')` |
| Route Handler needs immediate expiry | `revalidateTag('tag', { expire: 0 })` |

**In Server Actions, always use `updateTag`** — it ensures the user sees their own writes immediately after redirect.

### Tagging cache entries with `cacheTag`

Tag functions that fetch data using `'use cache'` + `cacheTag`. This enables `updateTag`/`revalidateTag` to invalidate them.

**Prerequisite:** Enable `cacheComponents: true` in `next.config.ts`:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}
export default nextConfig
```

**Tagging API functions:**

```ts
// src/domains/<domain>/api/get-<entities>.ts
import { cacheTag, cacheLife } from 'next/cache'

export async function get<Entities>(params: List<Entity>sParams) {
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>')

  const res = await api.get('v1/<entities>', { searchParams: params })
  return <entity>ListResponseSchema.parse(await res.json())
}
```

**Tagging with dynamic tags (detail pages):**

```ts
// src/domains/<domain>/api/get-<entity>.ts
import { cacheTag, cacheLife } from 'next/cache'

export async function get<Entity>(id: string) {
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>', `<entities>/${id}`)

  const res = await api.get(`v1/<entities>/${id}`)
  return <entity>ResponseSchema.parse(await res.json())
}
```

### `cacheLife` profiles

| Profile | stale | revalidate | expire |
|---------|-------|------------|--------|
| `seconds` | 30s | 1s | 1 min |
| `minutes` | 5 min | 1 min | 1 hour |
| `hours` | 5 min | 1 hour | 1 day |
| `days` | 5 min | 1 day | 1 week |
| `weeks` | 5 min | 1 week | 30 days |
| `max` | 5 min | 30 days | 1 year |

Use `'minutes'` as the default for most API data. Use `'hours'` or `'days'` for rarely-changing data. Use `'seconds'` for frequently-updated data.

---

## Create action

```ts
// src/domains/<domain>/actions/create-<entity>.ts
'use server'

import { updateTag } from 'next/cache'

import { DEFAULT_ERROR_MESSAGE } from '@/shared/constants/messages'
import { ApiError } from '@/shared/http/errors'
import { create<Entity> as create<Entity>Api } from '../api'
import { type Create<Entity>Request, create<Entity>Schema } from '../schemas'

export async function create<Entity>(
  data: Create<Entity>Request,
): Promise<ActionResult<<Entity>>> {
  const parsed = create<Entity>Schema.safeParse(data)

  if (!parsed.success) {
    return { ok: false, data: null, message: 'Invalid data.' }
  }

  try {
    const response = await create<Entity>Api(parsed.data)

    updateTag('<entities>')

    return { ok: true, data: response.data, message: response.message }
  } catch (error) {
    const message =
      error instanceof ApiError ? error.message : DEFAULT_ERROR_MESSAGE

    return { ok: false, data: null, message }
  }
}
```

---

## Edit action

```ts
// src/domains/<domain>/actions/edit-<entity>.ts
'use server'

import { updateTag } from 'next/cache'

import { DEFAULT_ERROR_MESSAGE } from '@/shared/constants/messages'
import { ApiError } from '@/shared/http/errors'
import { edit<Entity> as edit<Entity>Api } from '../api'
import { type Edit<Entity>Request, edit<Entity>Schema } from '../schemas'

export async function edit<Entity>(
  id: string,
  data: Edit<Entity>Request,
): Promise<ActionResult<<Entity>>> {
  const parsed = edit<Entity>Schema.safeParse(data)

  if (!parsed.success) {
    return { ok: false, data: null, message: 'Invalid data.' }
  }

  try {
    const response = await edit<Entity>Api(id, parsed.data)

    updateTag('<entities>')
    updateTag(`<entities>/${id}`)

    return { ok: true, data: response.data, message: response.message }
  } catch (error) {
    const message =
      error instanceof ApiError ? error.message : DEFAULT_ERROR_MESSAGE

    return { ok: false, data: null, message }
  }
}
```

Edit invalidates two tags: the list (`<entities>`) and the detail (`<entities>/${id}`).

---

## Delete action

```ts
// src/domains/<domain>/actions/delete-<entity>.ts
'use server'

import { updateTag } from 'next/cache'

import { DEFAULT_ERROR_MESSAGE } from '@/shared/constants/messages'
import { ApiError } from '@/shared/http/errors'
import { delete<Entity> } from '../api'

export async function delete<Entity>(id: string) {
  try {
    const res = await delete<Entity>(id)

    updateTag('<entities>')
    updateTag(`<entities>/${id}`)

    return {
      ok: true as const,
      data: null,
      message: res.message,
    }
  } catch (err: unknown) {
    const message =
      err instanceof ApiError
        ? err.message : DEFAULT_ERROR_MESSAGE

    return {
      ok: false as const,
      data: null,
      message,
    }
  }
}
```

---

## Toggle status action

```ts
// src/domains/<domain>/actions/toggle-<entity>-status.ts
'use server'

import { updateTag } from 'next/cache'

import { DEFAULT_ERROR_MESSAGE } from '@/shared/constants/messages'
import { ApiError } from '@/shared/http/errors'
import { toggle<Entity>Status } from '../api'

export async function toggle<Entity>Status(id: string) {
  try {
    const res = await toggle<Entity>Status(id)

    updateTag('<entities>')
    updateTag(`<entities>/${id}`)

    return {
      ok: true as const,
      data: res,
      message: res.message,
    }
  } catch (err: unknown) {
    const message =
      err instanceof ApiError
        ? err.message : DEFAULT_ERROR_MESSAGE

    return {
      ok: false as const,
      data: null,
      message,
    }
  }
}
```

---

## Barrel export

```ts
// src/domains/<domain>/actions/index.ts
export { create<Entity> } from './create-<entity>-action'
export { edit<Entity> } from './edit-<entity>-action'
export { delete<Entity> } from './delete-<entity>-action'
export { toggle<Entity>Status } from './toggle-<entity>-status-action'
```

---

## Rules

- `'use server'` directive is mandatory — actions run on the server
- Always parse input with Zod before calling the API — never trust client data
- Invalidate cache with `updateTag` after mutations in Server Actions — it ensures read-your-own-writes semantics
- Use `updateTag` for list tag + detail tag when editing/deleting — two calls, one per tag
- Use `revalidateTag(tag, profile)` only in Route Handlers (webhooks) — never use the deprecated single-argument `revalidateTag(tag)` form
- Tag API fetch functions with `cacheTag` inside `'use cache'` blocks — this is what makes `updateTag`/`revalidateTag` work
- Return `{ ok: true, data, message }` on success — `message` comes from the API response
- Return `{ ok: false, data: null, message }` on error — use `error.message` from `ApiError` (already extracted in constructor) or `DEFAULT_ERROR_MESSAGE`
- `DEFAULT_ERROR_MESSAGE` is imported from `shared/constants/messages` — never hardcode generic fallbacks
- One action per operation — never group multiple mutations in one file
- Every `actions/` directory must have an `index.ts` barrel export
- Actions never import UI code (React, components) — they are pure server functions

---

## Anti-patterns

```ts
// ❌ missing 'use server' directive
export async function createAction() { ... }  // will run on client — breaks

// ❌ no input validation
export async function createAction(body: CreateRequest) {
  const res = await createEntity(body)  // missing: schema.parse(body)
}

// ❌ no cache invalidation
export async function editAction(id: string, body: EditRequest) {
  const res = await editEntity(id, body)
  return { ok: true, data: res, message: res.message }
  // missing: updateTag('<entities>') + updateTag(`<entities>/${id}`)
}

// ❌ using deprecated single-argument revalidateTag in Server Actions
revalidateTag('<entities>')  // use updateTag('<entities>') instead

// ❌ hardcoded fallback message
catch (err) {
  return { ok: false, data: null, message: 'Something went wrong' }
  // use DEFAULT_ERROR_MESSAGE or extract from ApiError
}

// ❌ throwing instead of returning error result
catch (err) {
  throw new Error('Failed')  // never throw — return { ok: false, data: null, message }
}

// ❌ forgetting detail tag on edit/delete
updateTag('<entities>')
// missing: updateTag(`<entities>/${id}`)

// ❌ using cacheTag without 'use cache' directive
export async function getEntities() {
  cacheTag('entities')  // ERROR: cacheTag requires 'use cache' directive
  // ...
}

// ❌ using cacheTag without enabling cacheComponents in next.config.ts
// cacheComponents: true must be set in next.config.ts
```
