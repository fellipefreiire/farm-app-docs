# Unit Test Pattern (Vitest)

Unit tests validate pure logic in isolation — utilities, schemas, error classes, and stores. They live in `__tests__/` folders next to the code they test.

---

## File locations

```
src/
  shared/
    utils/__tests__/
      cn.spec.ts
      search-params.spec.ts
    http/errors/__tests__/
      error-classes.spec.ts
    http/errors/utils/__tests__/
      extract-message-from-body.spec.ts
      handle-http-error.spec.ts
  domains/
    <domain>/
      schemas/__tests__/
        <entity>.schema.spec.ts
      store/__tests__/
        <entity>-dialogs.store.spec.ts
```

Test files use `__tests__/` directory with `.spec.ts` extension.

---

## What to test

| Category | Test? | Reason |
|----------|-------|--------|
| **Pure utilities** (`cn`, `getSearchParam`, `extractMessageFromBody`) | Yes | No dependencies, fast, high value |
| **Zod schemas** (entity, request, response) | Yes | Validates data contracts with backend |
| **Error classes** (`ApiError`, `NotFoundError`, etc.) | Yes | Ensures correct status, message extraction |
| **Zustand stores** | Yes | Pure state logic, no React needed |
| **API functions** | No | Depend on HTTP client — covered by E2E |
| **Server actions** | No | Depend on cookies, cache, API — covered by E2E |
| **React components** | No | Depend on DOM, providers — covered by E2E (Playwright) |
| **Hooks** | No | Depend on React/Next.js router — covered by E2E |

---

## Test structure

Always use AAA pattern (Arrange, Act, Assert):

```ts
import { describe, expect, it } from 'vitest'

import { getSearchParam } from '../search-params'

describe('getSearchParam', () => {
  it('should return the string when value is a string', () => {
    // Arrange — input
    const value = 'hello'

    // Act — call function
    const result = getSearchParam(value)

    // Assert — verify output
    expect(result).toBe('hello')
  })
})
```

---

## Schema tests

Validate that schemas accept valid data and reject invalid data:

```ts
import { describe, expect, it } from 'vitest'

import { <entity>Schema } from '../<entity>.schema'

describe('<entity>Schema', () => {
  const validData = {
    id: '550e8400-e29b-41d4-a716-446655440000',
    name: 'Test',
    // ...all required fields
  }

  it('should parse valid data', () => {
    const result = <entity>Schema.safeParse(validData)
    expect(result.success).toBe(true)
  })

  it('should reject invalid field', () => {
    const result = <entity>Schema.safeParse({ ...validData, email: 'invalid' })
    expect(result.success).toBe(false)
  })

  it('should coerce date strings', () => {
    const result = <entity>Schema.safeParse(validData)
    expect(result.data?.createdAt).toBeInstanceOf(Date)
  })
})
```

---

## Store tests

Test state transitions using `getState()` and `setState()`:

```ts
import { beforeEach, describe, expect, it } from 'vitest'

import { use<Entity>DialogsStore } from '../<entity>-dialogs.store'

describe('use<Entity>DialogsStore', () => {
  beforeEach(() => {
    use<Entity>DialogsStore.setState({
      deleteDialogOpen: null,
    })
  })

  it('should open dialog with entity id', () => {
    use<Entity>DialogsStore.getState().setDeleteDialogOpen('some-id')
    expect(use<Entity>DialogsStore.getState().deleteDialogOpen).toBe('some-id')
  })
})
```

---

## Vitest config

```ts
// vitest.config.ts
import path from 'node:path'

import { defineConfig } from 'vitest/config'

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  test: {
    globals: true,
  },
})
```

---

## Rules

- Tests live in `__tests__/` folders next to the source code
- File naming: `<source-file>.spec.ts`
- AAA pattern: Arrange → Act → Assert
- One `describe` per module/function, one `it` per behavior
- Use `safeParse` for schema tests — avoid throwing on invalid data
- Reset store state in `beforeEach` — tests must be independent
- Mock only external dependencies (`vi.mock`) — never mock the function under test
- Only test pure logic — anything that needs HTTP, DOM, or Next.js internals belongs in E2E

---

## Anti-patterns

```ts
// ❌ testing implementation details
it('should call internal method', () => {
  const spy = vi.spyOn(module, '_internalMethod')  // test behavior, not internals
})

// ❌ testing API functions with mocked HTTP
vi.mock('@/shared/http/api-client')  // too much mocking — use E2E instead

// ❌ testing components with vitest
render(<Button />)  // use Playwright for component/integration tests

// ❌ not resetting store state
it('test 1', () => { store.getState().setOpen(true) })
it('test 2', () => { /* state leaks from test 1 */ })  // use beforeEach to reset
```
