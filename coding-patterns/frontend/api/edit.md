# API — Edit (PATCH)

```ts
// src/domains/<domain>/api/edit-<entity>.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type Edit<Entity>Request,
  type <Entity>MutationResponse,
  <entity>MutationResponseSchema,
} from '../schemas'

export async function edit<Entity>(
  id: string,
  data: Edit<Entity>Request,
): Promise<<Entity>MutationResponse> {
  try {
    const response = await api
      .patch(`v1/<entities>/${id}`, { json: data })
      .json()

    return <entity>MutationResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

Returns `MutationResponse` (`{ data, message }`). Cache invalidation (list + detail tags) handled in server action — see [`../action/edit.md`](../action/edit.md).
