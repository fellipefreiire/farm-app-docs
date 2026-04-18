# API — Caching (Next.js 16+)

Next.js 16 uses `'use cache'` + `cacheTag` + `cacheLife`. Wrap data-fetching API functions with `'use cache'` directive.

**Prerequisite:** `cacheComponents: true` in `next.config.ts` (see `action.md` for full setup).

## Cached list function

```ts
// src/domains/<domain>/api/list-<entity>s.ts
import { cacheTag, cacheLife } from 'next/cache'

import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type List<Entity>sParams,
  type <Entity>ListResponse,
  <entity>ListResponseSchema,
} from '../schemas'

export async function list<Entity>s(
  params: List<Entity>sParams,
): Promise<<Entity>ListResponse> {
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>')

  try {
    const searchParams = new URLSearchParams()

    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        searchParams.append(key, String(value))
      }
    })

    return await api
      .get(`v1/<entities>?${searchParams.toString()}`)
      .json()
      .then((data) => <entity>ListResponseSchema.parse(data))
  } catch (error) {
    return await handleHttpError(error)
  }
}
```

## Cached find by ID

```ts
// src/domains/<domain>/api/find-<entity>-by-id.ts
import { cacheTag, cacheLife } from 'next/cache'

import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type <Entity>,
  <entity>ResponseSchema,
} from '../schemas'

export async function find<Entity>ById(id: string): Promise<<Entity>> {
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>', `<entities>/${id}`)

  try {
    return await api
      .get(`v1/<entities>/${id}`)
      .json()
      .then((data) => <entity>ResponseSchema.parse(data))
      .then((data) => data.data)
  } catch (error) {
    return await handleHttpError(error)
  }
}
```

## Tag conventions

- List tag: `'<entities>'` (e.g. `'users'`, `'products'`)
- Detail tag: `'<entities>/${id}'` (e.g. `'users/abc-123'`)
- Detail functions tag **both** — so `updateTag('<entities>')` invalidates detail pages too

Cache invalidation in server actions via `updateTag('<entities>')` — see `../action/_index.md`.
