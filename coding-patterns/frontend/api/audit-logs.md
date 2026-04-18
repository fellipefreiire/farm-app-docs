# API — Audit Log Sub-resource

Each domain owns its audit log API function. Endpoint is sub-resource of entity (`/v1/<entities>/:id/audit-logs`), function lives in domain's `api/` folder.

```ts
// src/domains/<domain>/api/list-<entity>-audit-logs.ts
import { api } from '@/shared/http/api-client'
import { handleHttpError } from '@/shared/http/errors/utils/handle-http-error'

import {
  type ListAuditLogsResponse,
  listAuditLogsResponseSchema,
} from '@/domains/audit-log/schemas'

type ListAuditLogsParams = {
  cursor?: string
  limit?: number
}

export async function list<Entity>AuditLogs(
  id: string,
  params: ListAuditLogsParams = {},
): Promise<ListAuditLogsResponse> {
  try {
    const searchParams = new URLSearchParams()

    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        searchParams.append(key, String(value))
      }
    })

    const response = await api
      .get(`v1/<entities>/${id}/audit-logs?${searchParams.toString()}`)
      .json()

    return listAuditLogsResponseSchema.parse(response)
  } catch (error) {
    handleHttpError(error)
  }
}
```

**Key patterns:**
- Function lives in **entity's domain** (e.g. `domains/crop/crop-types/api/`), not `domains/audit-log/api/`
- Schemas (`ListAuditLogsResponse`, `listAuditLogsResponseSchema`) imported from shared `domains/audit-log/schemas/` — reused by all domains
- Shared `domains/audit-log/` keeps components (`AuditDiffModal`, `LimitedAuditLogs`, `AuditLogTable`), schemas, utils — only API functions are per-domain
