# API Function Pattern

API functions = typed fetch wrappers for backend endpoints. Domain layer. Shared HTTP client. Always Zod-validate responses.

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

> **Multi-entity domains:** Subdomain folders (see `domain-organization.md`) → API functions move under subdomain: `src/domains/<domain>/<subdomain>/api/`.

One function per endpoint. One file per function.

## Variants

| Operation | Pattern file |
|---|---|
| List (offset or cursor pagination) | [`list.md`](list.md) |
| Find by ID | [`find-by-id.md`](find-by-id.md) |
| Create | [`create.md`](create.md) |
| Edit | [`edit.md`](edit.md) |
| Delete | [`delete.md`](delete.md) |
| Toggle status | [`toggle-status.md`](toggle-status.md) |
| Caching (Next.js 16+) | [`caching.md`](caching.md) |
| Audit log sub-resource | [`audit-logs.md`](audit-logs.md) |

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

## Rules

> **Blueprint note:** Patterns cover common operations. New endpoints (bulk ops, file uploads, custom queries) follow same structure — one function per endpoint, Zod-parsed response, `handleHttpError` in catch.

- One function per endpoint — never group multiple API calls in one file
- Always parse with Zod — never trust raw API data
- Always catch with `handleHttpError` — handles 401 (redirect) and 404 (notFound)
- Use `URLSearchParams` + `Object.entries` for query strings — never build manually
- `find` unwraps `{ data }`, returns entity directly — `create`/`edit`/`toggle` return `MutationResponse` (`{ data, message }`) — `delete` returns `MessageResponse` (`{ message }`)
- Types/schemas always imported from domain's `schemas/` barrel — never from schema file directly
- Every `api/` directory needs `index.ts` barrel export

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
