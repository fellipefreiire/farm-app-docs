# Controller — ControllersModule (aggregator)

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

All domain controller modules are aggregated in a single `ControllersModule`, which is imported by `HttpModule`:

```ts
// src/infra/http/controllers/controllers.module.ts
import { Module } from '@nestjs/common'
import { UserHttpModule } from './user/user-http.module'

@Module({
  imports: [UserHttpModule],
})
export class ControllersModule {}
```

When adding a new domain, add its controller module to `ControllersModule.imports` — never add it directly to `HttpModule`.

---
