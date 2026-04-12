# Shared — Error classes

> Part of the farm-app frontend shared pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Custom error hierarchy for typed error handling in API functions and server actions.

```ts
// src/shared/http/errors/api.error.ts
export class ApiError extends Error {
  constructor(
    public status: number,
    public body: unknown,
    message: string = 'Request error',
  ) {
    super(message)
    this.name = 'ApiError'
  }
}
```

```ts
// src/shared/http/errors/bad-request.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class BadRequestError extends ApiError {
  constructor(body: unknown) {
    super(400, body, extractMessageFromBody(body, 'Invalid parameters'))
    this.name = 'BadRequestError'
  }
}
```

```ts
// src/shared/http/errors/unauthorized.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class UnauthorizedError extends ApiError {
  constructor(body: unknown) {
    super(401, body, extractMessageFromBody(body, 'Unauthorized'))
    this.name = 'UnauthorizedError'
  }
}
```

```ts
// src/shared/http/errors/forbidden.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class ForbiddenError extends ApiError {
  constructor(body: unknown) {
    super(403, body, extractMessageFromBody(body, 'Forbidden'))
    this.name = 'ForbiddenError'
  }
}
```

```ts
// src/shared/http/errors/not-found.error.ts
import { extractMessageFromBody } from './utils/extract-message-from-body'
import { ApiError } from './api.error'

export class NotFoundError extends ApiError {
  constructor(body: unknown) {
    super(404, body, extractMessageFromBody(body, 'Resource not found'))
    this.name = 'NotFoundError'
  }
}
```

```ts
// src/shared/http/errors/index.ts
export { ApiError } from './api.error'
export { BadRequestError } from './bad-request.error'
export { UnauthorizedError } from './unauthorized.error'
export { ForbiddenError } from './forbidden.error'
export { NotFoundError } from './not-found.error'
```

---
