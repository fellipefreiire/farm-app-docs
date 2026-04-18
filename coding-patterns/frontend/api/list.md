# API — List (GET with pagination)

## Offset pagination

```ts
// src/domains/<domain>/api/list-<entity>s.ts
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
  try {
    const searchParams = new URLSearchParams()

    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        searchParams.append(key, String(value))
      }
    })

    const response = await api
      .get(`v1/<entities>?${searchParams.toString()}`)
      .json()

    return <entity>ListResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

## Cursor pagination

Structure identical — only difference: import cursor variants from `list-<entity>s.schema.ts` (see `schema.md` — cursor list schemas). Function works as-is.

## Cached variant

See [`caching.md`](caching.md) — cached list wraps this function with `'use cache'` + `cacheTag` + `cacheLife`.
