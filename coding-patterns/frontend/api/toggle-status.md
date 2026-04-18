# API — Toggle Status (PATCH)

```ts
// src/domains/<domain>/api/toggle-<entity>-status.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type <Entity>MutationResponse,
  <entity>MutationResponseSchema,
} from '../schemas'

export async function toggle<Entity>Status(id: string): Promise<<Entity>MutationResponse> {
  try {
    const response = await api
      .patch(`v1/<entities>/${id}/toggle-status`)
      .json()

    return <entity>MutationResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

Returns `MutationResponse` (`{ data, message }`). Cache invalidation handled in server action — see [`../action/toggle-status.md`](../action/toggle-status.md).
