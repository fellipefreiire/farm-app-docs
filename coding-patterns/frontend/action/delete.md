# Action — Delete

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

Invalidates two tags: list (`<entities>`) and detail (`<entities>/${id}`).
