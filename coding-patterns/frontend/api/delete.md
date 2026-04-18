# API — Delete (DELETE)

```ts
// src/domains/<domain>/api/delete-<entity>.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type MessageResponse,
  messageResponseSchema,
} from '@/shared/http/schemas/message-response.schema'

export async function delete<Entity>(id: string): Promise<MessageResponse> {
  try {
    const response = await api
      .delete(`v1/<entities>/${id}`)
      .json()

    return messageResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

Returns `MessageResponse` (`{ message }`). Cache invalidation handled in server action — see [`../action/delete.md`](../action/delete.md).
