# Server Action Pattern

Server actions = bridge between UI components and API functions. Run on server, validate input, call API, invalidate cache, return standardized result.

## File locations

```
src/domains/<domain>/actions/create-<entity>.ts
src/domains/<domain>/actions/edit-<entity>.ts
src/domains/<domain>/actions/delete-<entity>.ts
src/domains/<domain>/actions/toggle-<entity>-status.ts
src/domains/<domain>/actions/index.ts                        ← barrel export
src/shared/constants/messages.ts                             ← DEFAULT_ERROR_MESSAGE
```

> **Multi-entity domains:** Domain uses subdomain folders (see `domain-organization.md`), actions move under subdomain: `src/domains/<domain>/<subdomain>/actions/`.

One action per operation. One file per action.

## Variants

| Operation | Pattern file |
|---|---|
| Create | [`create.md`](create.md) |
| Edit | [`edit.md`](edit.md) |
| Delete | [`delete.md`](delete.md) |
| Toggle status | [`toggle-status.md`](toggle-status.md) |

## Shared types and constants

`ActionResult<T>` declared globally in `src/shared/types/global.d.ts` — no import needed:

```ts
type ActionResult<T> =
  | { ok: true; data: T; message: string }
  | { ok: false; data: null; message: string }
```

```ts
// src/shared/constants/messages.ts
export const DEFAULT_ERROR_MESSAGE = 'An unexpected error occurred. Please try again.'
```

## Cache invalidation (Next.js 16+)

| Function | Where | Purpose |
|----------|-------|---------|
| `cacheTag(tag)` | Inside `'use cache'` functions/components | Labels cache entries for targeted invalidation |
| `cacheLife(profile)` | Inside `'use cache'` functions/components | Controls cache duration |
| `updateTag(tag)` | Server Actions only | Immediately expires cache — blocks until fresh data ready |
| `revalidateTag(tag, profile)` | Server Actions and Route Handlers | Marks tag stale — serves stale while revalidating in background |

| Scenario | Use |
|----------|-----|
| User creates/edits/deletes — must see own change | `updateTag('tag')` |
| Background refresh acceptable (webhooks, cron) | `revalidateTag('tag', 'max')` |
| Route Handler needs immediate expiry | `revalidateTag('tag', { expire: 0 })` |

**In Server Actions, always use `updateTag`** — ensures user sees own writes immediately after redirect.

**Prerequisite:** Enable `cacheComponents: true` in `next.config.ts`:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}
export default nextConfig
```

## Barrel export

```ts
// src/domains/<domain>/actions/index.ts
export { create<Entity> } from './create-<entity>-action'
export { edit<Entity> } from './edit-<entity>-action'
export { delete<Entity> } from './delete-<entity>-action'
export { toggle<Entity>Status } from './toggle-<entity>-status-action'
```

## Rules

- `'use server'` directive mandatory — actions run on server
- Parse input with Zod before calling API — never trust client data
- Invalidate cache with `updateTag` after mutations — ensures read-your-own-writes
- `updateTag` list tag + detail tag on edit/delete — two calls, one per tag
- `revalidateTag(tag, profile)` only in Route Handlers (webhooks) — never use deprecated single-arg `revalidateTag(tag)` form
- Tag API fetch functions with `cacheTag` inside `'use cache'` blocks — required for `updateTag`/`revalidateTag` to work
- Return `{ ok: true, data, message }` on success — `message` from API response
- Return `{ ok: false, data: null, message }` on error — use `error.message` from `ApiError` or `DEFAULT_ERROR_MESSAGE`
- Import `DEFAULT_ERROR_MESSAGE` from `shared/constants/messages` — never hardcode generic fallbacks
- One action per operation — never group multiple mutations in one file
- Every `actions/` directory must have `index.ts` barrel export
- Actions never import UI code (React, components) — pure server functions

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
