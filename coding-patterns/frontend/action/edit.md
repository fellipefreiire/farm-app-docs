# Action — Edit

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

Invalidates two tags: list (`<entities>`) and detail (`<entities>/${id}`).
