# Action — Toggle Status

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

Invalidates two tags: list (`<entities>`) and detail (`<entities>/${id}`).
