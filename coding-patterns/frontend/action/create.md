# Action — Create

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

Invalidates list tag only (`<entities>`). Detail tag not needed — entity didn't exist yet.
