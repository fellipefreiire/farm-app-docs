# Controller — Error Filter

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Each domain has its own error filter extending `AppErrorFilter`:

```ts
// src/infra/http/filters/<domain>-error.filter.ts
import { Catch } from '@nestjs/common'
import { AppErrorFilter } from './app-error.filter'
import { <Entity>NotFoundError } from '@/domain/<domain>/application/use-cases/errors/<entity>-not-found-error'
import { <Entity>AlreadyExistsError } from '@/domain/<domain>/application/use-cases/errors/<entity>-already-exists-error'

@Catch()
export class <Domain>ErrorFilter extends AppErrorFilter {
  protected override mapDomainErrorToStatus(name: string): number {
    switch (name) {
      case <Entity>NotFoundError.name: return 404
      case <Entity>AlreadyExistsError.name: return 409
      default: return super.mapDomainErrorToStatus(name)
    }
  }
}
```

`AppErrorFilter` base mappings: `ZodError → 422`, `BaseError → 400`, `HttpException → passthrough`, unknown → 500.

**Error flow:** `Either.left(error)` → controller `throw result.value` → `@Catch()` filter → `mapDomainErrorToStatus(error.name)` → HTTP response with status code and error message.

---
