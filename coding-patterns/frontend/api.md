# API Function Pattern

API functions are typed fetch wrappers that call backend endpoints. They live in the domain layer, use the shared HTTP client, and always validate responses with Zod.

---

## File locations

```
src/domains/<domain>/api/list-<entity>s.ts           ← GET with pagination
src/domains/<domain>/api/find-<entity>-by-id.ts      ← GET by ID
src/domains/<domain>/api/create-<entity>.ts           ← POST
src/domains/<domain>/api/edit-<entity>.ts             ← PATCH
src/domains/<domain>/api/delete-<entity>.ts           ← DELETE
src/domains/<domain>/api/toggle-<entity>-status.ts    ← PATCH toggle
src/domains/<domain>/api/index.ts                     ← barrel export
```

One function per endpoint. One file per function.

---

## List (GET with offset pagination)

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

### List with cursor pagination

The function structure is identical — the only difference is in the imported schemas. Import the cursor variants from `list-<entity>s.schema.ts` (see `schema.md` — cursor list schemas) and the function works as-is.

---

## Find by ID (GET)

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

`find` unwraps `{ data }` and returns the entity directly — the caller never deals with the response wrapper.

---

## Create (POST)

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

---

## Edit (PATCH)

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

---

## Delete (DELETE)

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

---

## Toggle status (PATCH)

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

---

## Caching (Next.js 16+)

Next.js 16 uses `'use cache'` + `cacheTag` + `cacheLife` for caching. API functions that fetch data can be cached by wrapping them with the `'use cache'` directive.

**Prerequisite:** `cacheComponents: true` in `next.config.ts` (see `action.md` for full setup).

### Cached list function

```ts
// src/domains/<domain>/api/list-<entity>s.ts
import { cacheTag, cacheLife } from 'next/cache'

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
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>')

  try {
    const searchParams = new URLSearchParams()

    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        searchParams.append(key, String(value))
      }
    })

    return await api
      .get(`v1/<entities>?${searchParams.toString()}`)
      .json()
      .then((data) => <entity>ListResponseSchema.parse(data))
  } catch (error) {
    return await handleHttpError(error)
  }
}
```

### Cached find by ID

```ts
// src/domains/<domain>/api/find-<entity>-by-id.ts
import { cacheTag, cacheLife } from 'next/cache'

import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type <Entity>,
  <entity>ResponseSchema,
} from '../schemas'

export async function find<Entity>ById(id: string): Promise<<Entity>> {
  'use cache'
  cacheLife('minutes')
  cacheTag('<entities>', `<entities>/${id}`)

  try {
    return await api
      .get(`v1/<entities>/${id}`)
      .json()
      .then((data) => <entity>ResponseSchema.parse(data))
      .then((data) => data.data)
  } catch (error) {
    return await handleHttpError(error)
  }
}
```

**Tag conventions:**
- List tag: `'<entities>'` (e.g. `'users'`, `'products'`)
- Detail tag: `'<entities>/${id}'` (e.g. `'users/abc-123'`)
- Detail functions tag **both** — so `updateTag('<entities>')` invalidates detail pages too

Cache invalidation happens in server actions via `updateTag('<entities>')` — see `action.md`.

---

## Barrel export

```ts
// src/domains/<domain>/api/index.ts
export { list<Entity>s } from './list-<entity>s'
export { find<Entity>ById } from './find-<entity>-by-id'
export { create<Entity> } from './create-<entity>'
export { edit<Entity> } from './edit-<entity>'
export { delete<Entity> } from './delete-<entity>'
export { toggle<Entity>Status } from './toggle-<entity>-status'
```

---

## Rules

> **Blueprint note:** These patterns cover the most common API operations. New endpoints not listed here (e.g. bulk operations, file uploads, custom queries) should follow the same structure — one function per endpoint, Zod-parsed response, `handleHttpError` in catch.

- One function per endpoint — never group multiple API calls in one file
- Always parse responses with Zod — never trust raw API data
- Always catch errors with `handleHttpError` — it handles 401 (redirect) and 404 (notFound)
- Use `URLSearchParams` + `Object.entries` for query string serialization — never build strings manually
- `find` unwraps `{ data }` and returns the entity directly — `create`, `edit`, `toggle` return `MutationResponse` (`{ data, message }`) — `delete` returns `MessageResponse` (`{ message }`)
- Types and schemas are always imported from the domain's `schemas/` barrel — never from the schema file directly
- Every `api/` directory must have an `index.ts` barrel export

---

## Anti-patterns

```ts
// ❌ manual searchParams building
const searchParams: Record<string, string> = {}
if (params.page) searchParams.page = String(params.page)
if (params.perPage) searchParams.perPage = String(params.perPage)
// use URLSearchParams + Object.entries instead

// ❌ no Zod parse on response
const response = await api.get('v1/entities').json()
return response  // always parse: schema.parse(response)

// ❌ no error handling
export async function listEntities() {
  return await api.get('v1/entities').json()  // missing try/catch + handleHttpError
}

// ❌ importing schema from deep path
import { entitySchema } from '../schemas/entity.schema'  // import from '../schemas'

// ❌ multiple endpoints in one file
export async function createEntity() { ... }
export async function editEntity() { ... }  // split into separate files

// ❌ returning raw response from find
return await api.get(`v1/entities/${id}`).json()
  .then((data) => schema.parse(data))
// missing: .then((data) => data.data) — find should unwrap { data }
```
