# Use Case — Per-domain audit list (with ID resolution)

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Each domain owns its audit log listing. The use case filters by entity type, fetches logs with cursor pagination, then batch-resolves actor IDs and any domain-specific foreign key IDs to human-readable names.

```ts
import { Injectable } from '@nestjs/common'
import { right, type Either } from '@/core/either'
import type { CursorPaginationMeta } from '@/core/repositories/pagination-params'
import type { AuditLog } from '@/domain/audit-log/enterprise/entities/audit-log'
import type { AuditLogRepository } from '@/domain/audit-log/application/repositories/audit-log-repository'
import { QueryBus } from '@/core/query-bus/query-bus'
import { FindUsersByIdsQuery, type FindUsersByIdsQueryResult }
  from '@/domain/user/application/queries/find-users-by-ids.query'
import type { <Related>Repository } from '@/domain/<domain>/application/repositories/<related>-repository'
import { extractChangeIds } from '@/shared/utils/extract-change-ids'

type List<Entity>AuditLogsUseCaseRequest = {
  entityId: string
  cursor?: string
  limit?: number
}

type List<Entity>AuditLogsUseCaseResponse = Either<
  null,
  {
    data: AuditLog[]
    meta: CursorPaginationMeta
    actorNames: Map<string, string>
    resolvedNames: Map<string, string>
  }
>

/** Lists audit logs for a specific <entity>, resolving actor and related entity names. */
@Injectable()
export class List<Entity>AuditLogsUseCase {
  constructor(
    private auditLogRepository: AuditLogRepository,
    private <related>Repository: <Related>Repository,  // only if entity has foreign IDs to resolve (same domain)
  ) {}

  async execute({
    entityId,
    cursor,
    limit = 20,
  }: List<Entity>AuditLogsUseCaseRequest): Promise<List<Entity>AuditLogsUseCaseResponse> {
    const result = await this.auditLogRepository.listWithCursor({
      entity: '<ENTITY_CONSTANT>',
      entityId,
      cursor,
      limit,
    })

    // 1. Resolve actor names via QueryBus (User is a different domain)
    const actorIds = [...new Set(result.items.map((log) => log.actorId))]
    const actors = await QueryBus.execute<FindUsersByIdsQueryResult>(
      new FindUsersByIdsQuery(actorIds),
    )
    const actorNames = new Map(actors.map((a) => [a.id, a.name]))

    // 2. Resolve domain-specific foreign IDs from changes (same-domain repos only)
    const resolvedNames = new Map<string, string>()

    const relatedIds = extractChangeIds(result.items, '<relatedField>Id')
    if (relatedIds.length > 0) {
      const relatedEntities = await this.<related>Repository.findManyByIds(relatedIds)
      for (const entity of relatedEntities) {
        resolvedNames.set(entity.id.toString(), entity.name)
      }
    }

    return right({
      data: result.items,
      meta: { nextCursor: result.nextCursor, count: result.count },
      actorNames,
      resolvedNames,
    })
  }
}
```

**Key patterns:**
- Actor name resolution uses **QueryBus** (`FindUsersByIdsQuery`) — User is a different domain, never import `UsersRepository` directly
- Domain-specific foreign ID resolution (e.g. variety → cropType) uses **direct repository injection** — these are within the same domain
- Each domain use case knows which foreign IDs exist in its changes (e.g. variety knows `cropTypeId`, harvest knows `cropTypeId` + `varietyId`)
- Use `extractChangeIds()` from `@/shared/utils/extract-change-ids` to extract IDs from `{ from, to }` change fields
- Batch-load with `findManyByIds()` — never loop with individual `findById` calls
- Simple domains (e.g. crop types with only `name` changes) skip the foreign ID resolution step

---
