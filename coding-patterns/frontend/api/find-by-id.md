# API — Find by ID (GET)

```ts
// src/domains/<domain>/api/find-<entity>-by-id.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type <Entity>,
  <entity>ResponseSchema,
} from '../schemas'

export async function find<Entity>ById(id: string): Promise<<Entity>> {
  try {
    const response = await api
      .get(`v1/<entities>/${id}`)
      .json()

    const parsed = <entity>ResponseSchema.parse(response)
    return parsed.data
  } catch (error) {
    handleHttpError(error)
  }
}
```

`find` unwraps `{ data }`, returns entity directly — caller never sees response wrapper.

## Cached variant

See [`caching.md`](caching.md) — cached find wraps this with `'use cache'` + `cacheTag('<entities>', '<entities>/${id}')`.
