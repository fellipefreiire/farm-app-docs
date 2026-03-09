# Schema Pattern

Schemas define the shape of API responses, request payloads, and form validation using Zod. They are the source of truth for TypeScript types in the frontend — never define types manually when a schema can infer them.

---

## File locations

```
src/domains/<domain>/schemas/<entity>.schema.ts              ← entity shape + response schemas
src/domains/<domain>/schemas/create-<entity>.schema.ts       ← create request validation
src/domains/<domain>/schemas/edit-<entity>.schema.ts         ← edit request validation
src/domains/<domain>/schemas/list-<entity>s.schema.ts        ← list params + list response
src/domains/<domain>/schemas/index.ts                        ← barrel export
src/shared/http/schemas/pagination.schema.ts                 ← offset + cursor pagination (shared)
src/shared/http/schemas/message-response.schema.ts           ← delete/action-only response ({ message })
```

One schema per functionality. One file per schema.

---

## Shared pagination schemas

Defined once in `shared/`, imported by every domain that lists data.

```ts
// src/shared/http/schemas/pagination.schema.ts
import { z } from 'zod'

// Offset pagination — admin panels, tables with page numbers
export const paginationSchema = z.object({
  currentPage: z.coerce.number(),
  nextPage: z.coerce.number().nullable(),
  perPage: z.coerce.number(),
  previousPage: z.coerce.number().nullable(),
  total: z.coerce.number(),
  totalPages: z.coerce.number(),
})

export type PaginationMeta = z.infer<typeof paginationSchema>

// Cursor pagination — feeds, logs, infinite scroll
export const cursorPaginationSchema = z.object({
  nextCursor: z.uuid().nullable(),
  count: z.coerce.number(),
})

export type CursorPaginationMeta = z.infer<typeof cursorPaginationSchema>
```

### Message response schema

Used for operations that return only a message (e.g. delete).

```ts
// src/shared/http/schemas/message-response.schema.ts
import { z } from 'zod'

export const messageResponseSchema = z.object({
  message: z.string(),
})

export type MessageResponse = z.infer<typeof messageResponseSchema>
```

---

## Entity schema

Describes the shape returned by the API for a single entity. This is the canonical type used across the frontend.

```ts
// src/domains/<domain>/schemas/<entity>.schema.ts
import { z } from 'zod'

export const <entity>Schema = z.object({
  id: z.uuid(),
  name: z.string(),
  description: z.string().nullable(),
  active: z.boolean(),
  createdAt: z.coerce.date(),
  updatedAt: z.coerce.date(),
})

export type <Entity> = z.infer<typeof <entity>Schema>

export const <entity>ResponseSchema = z.object({
  data: <entity>Schema,
})

export type <Entity>Response = z.infer<typeof <entity>ResponseSchema>

export const <entity>MutationResponseSchema = z.object({
  data: <entity>Schema,
  message: z.string(),
})

export type <Entity>MutationResponse = z.infer<typeof <entity>MutationResponseSchema>
```

**Rules:**
- Optional API fields use `.nullable()` — matches the backend contract (`string | null`)
- Dates always use `z.coerce.date()` — the API returns ISO strings, Zod coerces to `Date`
- The entity type is inferred from the schema — never define it manually
- Read responses use `<entity>ResponseSchema` (`{ data }`) — find-by-id
- Mutation responses use `<entity>MutationResponseSchema` (`{ data, message }`) — create, edit, toggle
- Delete uses `messageResponseSchema` from shared (`{ message }`) — no data returned

---

## Create request schema

Validates the create form before sending to the API.

```ts
// src/domains/<domain>/schemas/create-<entity>.schema.ts
import { z } from 'zod'

export const create<Entity>Schema = z.object({
  name: z.string().min(1, 'Name is required'),
  description: z.string().optional(),
})

export type Create<Entity>Request = z.infer<typeof create<Entity>Schema>
```

**Rules:**
- Required fields use `.min(1, 'message')` for strings — not just `.string()` (empty string would pass)
- Optional fields that can be omitted from the request use `.optional()`
- Optional fields that the API expects as `null` use `.nullable()`
- Validation messages match the UI language

---

## Edit request schema

Validates the edit form. Often identical to create, but split into its own file — edit schemas diverge as the domain grows (e.g. some fields become read-only after creation).

```ts
// src/domains/<domain>/schemas/edit-<entity>.schema.ts
import { z } from 'zod'

export const edit<Entity>Schema = z.object({
  name: z.string().min(1, 'Name is required'),
  description: z.string().optional(),
})

export type Edit<Entity>Request = z.infer<typeof edit<Entity>Schema>
```

---

## List schemas

List params (query string) and list response (API response with pagination metadata).

### Offset pagination (default)

```ts
// src/domains/<domain>/schemas/list-<entity>s.schema.ts
import { z } from 'zod'

import { paginationSchema } from '@/shared/http/schemas/pagination.schema'

import { <entity>Schema } from './<entity>.schema'

export const list<Entity>sParamsSchema = z.object({
  page: z.coerce.number().optional(),
  perPage: z.coerce.number().min(1).max(100).optional(),
  search: z.string().optional(),
  active: z.boolean().optional(),
  // add domain-specific filters as needed
})

export type List<Entity>sParams = z.infer<typeof list<Entity>sParamsSchema>

export const <entity>ListResponseSchema = z.object({
  data: z.array(<entity>Schema),
  meta: paginationSchema,
})

export type <Entity>ListResponse = z.infer<typeof <entity>ListResponseSchema>
```

### Cursor pagination

When the domain uses cursor-based pagination (feeds, logs, infinite scroll):

```ts
// src/domains/<domain>/schemas/list-<entity>s.schema.ts
import { z } from 'zod'

import { cursorPaginationSchema } from '@/shared/http/schemas/pagination.schema'

import { <entity>Schema } from './<entity>.schema'

export const list<Entity>sParamsSchema = z.object({
  cursor: z.uuid().optional(),
  limit: z.coerce.number().min(1).max(100).optional(),
  // add domain-specific filters as needed
})

export type List<Entity>sParams = z.infer<typeof list<Entity>sParamsSchema>

export const <entity>ListResponseSchema = z.object({
  data: z.array(<entity>Schema),
  nextCursor: cursorPaginationSchema.shape.nextCursor,
  count: cursorPaginationSchema.shape.count,
})

export type <Entity>ListResponse = z.infer<typeof <entity>ListResponseSchema>
```

---

## Barrel export

Every `schemas/` directory has an `index.ts` that re-exports all schemas and types.

```ts
// src/domains/<domain>/schemas/index.ts
export {
  <entity>Schema,
  <entity>ResponseSchema,
  <entity>MutationResponseSchema,
  type <Entity>,
  type <Entity>Response,
  type <Entity>MutationResponse,
} from './<entity>.schema'

export {
  create<Entity>Schema,
  type Create<Entity>Request,
} from './create-<entity>.schema'

export {
  edit<Entity>Schema,
  type Edit<Entity>Request,
} from './edit-<entity>.schema'

export {
  list<Entity>sParamsSchema,
  <entity>ListResponseSchema,
  type List<Entity>sParams,
  type <Entity>ListResponse,
} from './list-<entity>s.schema'
```

---

## Rules

- One schema file per functionality — schemas grow with the domain, splitting prevents large files
- Types are always inferred from schemas (`z.infer<typeof schema>`) — never manually defined
- Entity schema matches the API response shape exactly — it is the contract
- Optional API fields: `.nullable()` when the API returns `null`, `.optional()` when the field can be omitted
- Dates: always `z.coerce.date()` — the API returns ISO strings
- Shared schemas (`paginationSchema`, `cursorPaginationSchema`) live in `shared/http/schemas/` — domains import and compose
- Every `schemas/` directory must have an `index.ts` barrel export
- Request schemas (create, edit) validate form input — they may differ from the entity schema

---

## Anti-patterns

```ts
// ❌ manually defined type instead of inferred
type Entity = {
  id: string
  name: string
}
// always: type Entity = z.infer<typeof entitySchema>

// ❌ all schemas in one file
// domains/product/schemas/product.schema.ts with 200+ lines
// split: product.schema.ts, create-product.schema.ts, edit-product.schema.ts, list-products.schema.ts

// ❌ using .nullish() when the API contract is .nullable()
description: z.string().nullish()  // use .nullable() — match the API

// ❌ missing barrel export
// importing from deep path: import { Entity } from '../schemas/entity.schema'
// always: import { Entity } from '../schemas'

// ❌ duplicating pagination schema in domain
const meta = z.object({ total: z.number(), ... })  // import from shared

// ❌ no validation message on required fields
name: z.string()  // empty string passes — use .min(1, 'Name is required')
```
