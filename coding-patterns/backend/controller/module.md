# Controller — Module

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```ts
import { Module } from '@nestjs/common'
import { <Entity>DatabaseModule } from '@/infra/database/prisma/repositories/<domain>/<entity>-database.module'
import { Create<Entity>Controller } from './create-<entity>.controller'
import { List<Entity>sController } from './list-<entity>s.controller'
import { Find<Entity>ByIdController } from './find-<entity>-by-id.controller'
import { Edit<Entity>Controller } from './edit-<entity>.controller'
import { Delete<Entity>Controller } from './delete-<entity>.controller'
import { Toggle<Entity>StatusController } from './toggle-<entity>-status.controller'
import { Create<Entity>UseCase } from '@/domain/<domain>/application/use-cases/create-<entity>'
import { List<Entity>sUseCase } from '@/domain/<domain>/application/use-cases/list-<entity>s'
import { Find<Entity>ByIdUseCase } from '@/domain/<domain>/application/use-cases/find-<entity>-by-id'
import { Edit<Entity>UseCase } from '@/domain/<domain>/application/use-cases/edit-<entity>'
import { Delete<Entity>UseCase } from '@/domain/<domain>/application/use-cases/delete-<entity>'
import { Toggle<Entity>StatusUseCase } from '@/domain/<domain>/application/use-cases/toggle-<entity>-status'

@Module({
  imports: [<Entity>DatabaseModule],
  controllers: [
    Create<Entity>Controller,
    List<Entity>sController,
    Find<Entity>ByIdController,
    Edit<Entity>Controller,
    Delete<Entity>Controller,
    Toggle<Entity>StatusController,
  ],
  providers: [
    Create<Entity>UseCase,
    List<Entity>sUseCase,
    Find<Entity>ByIdUseCase,
    Edit<Entity>UseCase,
    Delete<Entity>UseCase,
    Toggle<Entity>StatusUseCase,
  ],
})
export class <Entity>ControllersModule {}
```

---
