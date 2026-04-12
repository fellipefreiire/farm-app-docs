# Controller — Swagger DTOs

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

DTOs are **only for Swagger documentation** — they do not validate input (Zod does that).

```ts
// src/infra/http/dtos/requests/<domain>/<action>-<entity>-request.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger'

export class <Action><Entity>RequestDto {
  @ApiProperty({ example: 'Name value' })
  name!: string

  @ApiPropertyOptional({ example: 'optional value' })
  optionalField?: string
}
```

```ts
// src/infra/http/dtos/response/<domain>/<entity>-response.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger'

export class <Entity>ResponseDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440000' })
  id!: string

  @ApiProperty({ example: 'Name value' })
  name!: string

  @ApiProperty({ example: true })
  active!: boolean

  @ApiProperty({ example: '2024-01-01T00:00:00.000Z' })
  createdAt!: string

  @ApiPropertyOptional({ example: '2024-01-01T00:00:00.000Z' })
  updatedAt?: string | null
}
```

```ts
// src/infra/http/dtos/error/<domain>/<entity>-not-found.dto.ts
import { ApiProperty } from '@nestjs/swagger'

export class <Entity>NotFoundDto {
  @ApiProperty({ example: 404 })
  statusCode!: number

  @ApiProperty({ example: 'Not Found' })
  error!: string

  @ApiProperty({ example: '<Entity> not found' })
  message!: string
}
```

---
