# Controller — Sub-resource Audit Logs

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Each entity owns its own audit log endpoint as a sub-resource. The controller filters by entity type (hardcoded) and delegates ID resolution to the use case.

```ts
import { Controller, Get, HttpCode, Param, ParseUuidPipe, Query, UseGuards } from '@nestjs/common'
import {
  ApiOperation, ApiQuery, ApiTags, ApiOkResponse,
  ApiForbiddenResponse, ApiUnauthorizedResponse, ApiInternalServerErrorResponse,
} from '@nestjs/swagger'
import { z } from 'zod'
import { ZodValidationPipe } from '../../pipes/zod-validation.pipe'
import { CaslAbilityGuard } from '@/infra/auth/casl/casl-ability.guard'
import { CheckPolicies } from '@/infra/auth/casl/check-policies.decorator'
import { List<Entity>AuditLogsUseCase } from '@/domain/<domain>/application/use-cases/list-<entity>-audit-logs'
import { AuditLogPresenter } from '../../presenters/audit-log.presenter'

const cursorQuerySchema = z.object({
  cursor: z.string().uuid().optional(),
  limit: z.coerce.number().min(1).max(100).optional().default(20),
})

type CursorQuerySchema = z.infer<typeof cursorQuerySchema>

@ApiTags('<Entities>')
@Controller({ path: '<entities>', version: '1' })
export class List<Entity>AuditLogsController {
  constructor(private list<Entity>AuditLogsUseCase: List<Entity>AuditLogsUseCase) {}

  @Get(':id/audit-logs')
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('list', 'AuditLog'))
  @HttpCode(200)
  @ApiOperation({ summary: 'List audit logs for <entity>' })
  @ApiQuery({ name: 'cursor', required: false, type: String })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  async handle(
    @Param('id', ParseUuidPipe) id: string,
    @Query(new ZodValidationPipe(cursorQuerySchema)) query: CursorQuerySchema,
  ) {
    const result = await this.list<Entity>AuditLogsUseCase.execute({
      entityId: id,
      ...query,
    })

    if (result.isLeft()) throw result.value

    const { data, meta, actorNames, resolvedNames } = result.value
    return {
      data: data.map((log) =>
        AuditLogPresenter.toHTTP(log, actorNames.get(log.actorId), resolvedNames),
      ),
      ...meta,
    }
  }
}
```

**Route ordering rule:** In the NestJS module, audit-log controllers (`@Get(':id/audit-logs')`) MUST be registered BEFORE find-by-id controllers (`@Get(':id')`). NestJS matches routes in registration order — a wildcard `:id` route registered first will shadow `:id/audit-logs`:

```ts
@Module({
  controllers: [
    // ✅ audit-logs controllers FIRST — more specific routes
    List<Entity>AuditLogsController,
    // then generic :id routes
    Find<Entity>ByIdController,
  ],
})
```

---
