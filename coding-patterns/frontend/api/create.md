# API — Create (POST)

```ts
// src/domains/<domain>/api/create-<entity>.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type Create<Entity>Request,
  type <Entity>MutationResponse,
  <entity>MutationResponseSchema,
} from '../schemas'

export async function create<Entity>(
  data: Create<Entity>Request,
): Promise<<Entity>MutationResponse> {
  try {
    const response = await api
      .post('v1/<entities>', { json: data })
      .json()

    return <entity>MutationResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

Returns `MutationResponse` (`{ data, message }`). Cache invalidation handled in the server action — see [`../action/create.md`](../action/create.md).
