# Controller Pattern

Controllers handle HTTP routing and input validation. They delegate all logic to use cases — no business logic lives here.

> This pattern also covers DTOs (`dtos/`), Error Filters (`filters/`), and NestJS Modules — see sections below.

---

## File locations

```
src/infra/http/controllers/<domain>/create-<entity>.controller.ts
src/infra/http/controllers/<domain>/list-<entity>s.controller.ts
src/infra/http/controllers/<domain>/find-<entity>-by-id.controller.ts
src/infra/http/controllers/<domain>/edit-<entity>.controller.ts
src/infra/http/controllers/<domain>/delete-<entity>.controller.ts
src/infra/http/controllers/<domain>/toggle-<entity>-status.controller.ts
src/infra/http/controllers/<domain>/<domain>-controllers.module.ts
src/infra/http/filters/<domain>-error.filter.ts
src/infra/http/dtos/requests/<domain>/         ← request DTOs (Swagger docs)
src/infra/http/dtos/response/<domain>/         ← response DTOs (Swagger docs)
src/infra/http/dtos/error/<domain>/            ← domain-specific error DTOs
src/infra/auth/casl/handlers/<entity>-<action>.handler.ts  ← CASL policy handlers
```

One controller per endpoint. One file per controller.

---

## Authentication — default behavior

**All endpoints require JWT authentication by default.** The `JwtAuthGuard` is applied globally.

Use `@Public()` **only when explicitly required** — e.g. sign-in, sign-up, public listings. Never assume an endpoint is public. If the domain rules file does not explicitly state an endpoint is public, it is private.

---

## Structure — create (with body, CASL, full Swagger)

```ts
import {
  Body, Controller, HttpCode, Post, UseFilters, UseGuards,
} from '@nestjs/common'
import {
  ApiBody, ApiCreatedResponse, ApiBadRequestResponse,
  ApiForbiddenResponse, ApiUnauthorizedResponse,
  ApiUnprocessableEntityResponse, ApiInternalServerErrorResponse,
  ApiOperation, ApiTags,
} from '@nestjs/swagger'
import { z } from 'zod'
import { ZodValidationPipe } from '../../pipes/zod-validation.pipe'
import { CaslAbilityGuard } from '@/infra/auth/casl/casl-ability.guard'
import { CheckPolicies } from '@/infra/auth/casl/check-policies.decorator'
import { CurrentUser } from '@/infra/auth/current-user.decorator'
import type { UserPayload } from '@/infra/auth/jwt.strategy'
import { <Action><Entity>UseCase } from '@/domain/<domain>/application/use-cases/<action>-<entity>'
import { <Entity>Presenter } from '../../presenters/<entity>.presenter'
import { <Domain>ErrorFilter } from '../../filters/<domain>-error.filter'
import { <Action><Entity>RequestDto } from '../../dtos/requests/<domain>'
import { <Entity>ResponseDto } from '../../dtos/response/<domain>'
import {
  BadRequestDto, UnprocessableEntityDto, InternalServerErrorDto,
  WrongCredentialsDto,
} from '../../dtos/error/generic'
import { <Entity>ForbiddenDto } from '../../dtos/error/<domain>'

const <action><Entity>BodySchema = z.object({
  name: z.string().min(1),
  optionalField: z.string().optional(),
})

type <Action><Entity>BodySchema = z.infer<typeof <action><Entity>BodySchema>

@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@ApiBearerAuth()
@ServiceTag('<domain>')
@Controller({ path: '<entities>', version: '1' })
export class <Action><Entity>Controller {
  constructor(private <action><Entity>UseCase: <Action><Entity>UseCase) {}

  @Post()
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('create', '<Entity>'))
  @HttpCode(201)
  @ApiOperation({ summary: 'Create <entity>' })
  @ApiBody({ type: <Action><Entity>RequestDto })
  @ApiCreatedResponse({ type: <Entity>ResponseDto })
  @ApiBadRequestResponse({ type: BadRequestDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiUnprocessableEntityResponse({ type: UnprocessableEntityDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @CurrentUser() currentUser: UserPayload,
    @Body(new ZodValidationPipe(<action><Entity>BodySchema))
    body: <Action><Entity>BodySchema,
  ) {
    const result = await this.<action><Entity>UseCase.execute({
      ...body,
      actorId: currentUser.sub,
    })

    if (result.isLeft()) throw result.value

    return {
      data: <Entity>Presenter.toHTTP(result.value.data),
      message: '<Entity> created successfully',
    }
  }
}
```

---

## Structure — list (query params, CASL or Public)

Each domain defines its own filter fields — there is no fixed template. Declare only what the domain needs.

```ts
import { Controller, Get, HttpCode, Query, UseFilters, UseGuards } from '@nestjs/common'
import {
  ApiOperation, ApiQuery, ApiTags, ApiOkResponse,
  ApiForbiddenResponse, ApiUnauthorizedResponse, ApiInternalServerErrorResponse,
} from '@nestjs/swagger'
import { z } from 'zod'
import { ZodValidationPipe } from '../../pipes/zod-validation.pipe'
import { CaslAbilityGuard } from '@/infra/auth/casl/casl-ability.guard'
import { CheckPolicies } from '@/infra/auth/casl/check-policies.decorator'
import { List<Entity>sUseCase } from '@/domain/<domain>/application/use-cases/list-<entity>s'
import { <Entity>Presenter } from '../../presenters/<entity>.presenter'
import { <Domain>ErrorFilter } from '../../filters/<domain>-error.filter'
import { <Entity>ResponseDto } from '../../dtos/response/<domain>'
import {
  WrongCredentialsDto, InternalServerErrorDto,
} from '../../dtos/error/generic'
import { <Entity>ForbiddenDto } from '../../dtos/error/<domain>'

const querySchema = z.object({
  // --- pagination (always present) ---
  page: z.coerce.number().min(1).optional().default(1),
  perPage: z.coerce.number().min(1).max(100).optional().default(20),

  // --- sorting (optional — domain defines allowed fields) ---
  sort: z.enum(['name', 'createdAt']).optional(),
  order: z.enum(['asc', 'desc']).optional().default('asc'),

  // --- text search across multiple columns (optional) ---
  search: z.string().optional(),

  // --- boolean filter ---
  active: z.string().optional().transform((val) => {
    if (val === 'true') return true
    if (val === 'false') return false
    return undefined
  }),

  // --- enum filter (single value) ---
  status: z.nativeEnum(<StatusEnum>).optional(),

  // --- multi-value enum (single value OR array from query string) ---
  type: z.union([
    z.nativeEnum(<TypeEnum>),
    z.array(z.nativeEnum(<TypeEnum>)),
  ]).optional(),

  // --- date range ---
  startDate: z.coerce.date().optional(),
  endDate: z.coerce.date().optional(),
})

type QuerySchema = z.infer<typeof querySchema>

@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@Controller({ path: '<entities>', version: '1' })
export class List<Entity>sController {
  constructor(private list<Entity>sUseCase: List<Entity>sUseCase) {}

  @Get()
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('list', '<Entity>'))
  @HttpCode(200)
  @ApiOperation({ summary: 'List <entities>' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'perPage', required: false, type: Number })
  @ApiQuery({ name: 'sort', required: false, enum: ['name', 'createdAt'] })
  @ApiQuery({ name: 'order', required: false, enum: ['asc', 'desc'] })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiQuery({ name: 'active', required: false, type: Boolean })
  @ApiQuery({ name: 'status', required: false, enum: <StatusEnum> })
  @ApiQuery({ name: 'type', required: false, isArray: true, enum: <TypeEnum> })
  @ApiQuery({ name: 'startDate', required: false, type: String, example: '2024-01-01T00:00:00.000Z' })
  @ApiQuery({ name: 'endDate', required: false, type: String, example: '2024-12-31T23:59:59.000Z' })
  @ApiOkResponse({ type: <Entity>ResponseDto, isArray: true })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @Query(new ZodValidationPipe(querySchema)) query: QuerySchema,
  ) {
    const result = await this.list<Entity>sUseCase.execute({
      ...query,
      // normalize multi-value enum to always be array
      type: Array.isArray(query.type) ? query.type : query.type ? [query.type] : undefined,
    })

    if (result.isLeft()) throw result.value

    const { data, meta } = result.value
    return { data: data.map(<Entity>Presenter.toHTTP), meta }
  }
}
```

**Filter field naming conventions:**
- `sort` — column to sort by, validated with `z.enum([...])` using the domain's allowed sort fields
- `order` — sort direction (`asc` or `desc`), defaults to `asc`
- `search` — free text across multiple columns (domain defines which columns in the repository)
- `active` — boolean toggle, always string-transformed
- `status`, `type`, etc. — enum fields, use `z.nativeEnum()`
- Multi-value enums — use `z.union([z.nativeEnum(), z.array(z.nativeEnum())])`, normalize to array before passing to use case
- `startDate` / `endDate` — date range, use `z.coerce.date()` for automatic string → Date conversion
- Specific field filters (`name`, `reference`, `email`) — use `z.string().optional()` directly

---

## Structure — list with cursor pagination

For feeds, logs, timelines, and infinite scroll — see decision rule in `repository.md`.

```ts
import { Controller, Get, HttpCode, Query, UseFilters, UseGuards } from '@nestjs/common'
import {
  ApiOperation, ApiQuery, ApiTags, ApiOkResponse,
  ApiForbiddenResponse, ApiUnauthorizedResponse, ApiInternalServerErrorResponse,
} from '@nestjs/swagger'
import { z } from 'zod'
import { ZodValidationPipe } from '../../pipes/zod-validation.pipe'
import { CaslAbilityGuard } from '@/infra/auth/casl/casl-ability.guard'
import { CheckPolicies } from '@/infra/auth/casl/check-policies.decorator'
import { List<Entity>sCursorUseCase } from '@/domain/<domain>/application/use-cases/list-<entity>s-cursor'
import { <Entity>Presenter } from '../../presenters/<entity>.presenter'
import { <Domain>ErrorFilter } from '../../filters/<domain>-error.filter'
import { <Entity>ResponseDto } from '../../dtos/response/<domain>'
import {
  WrongCredentialsDto, InternalServerErrorDto,
} from '../../dtos/error/generic'
import { <Entity>ForbiddenDto } from '../../dtos/error/<domain>'

const cursorQuerySchema = z.object({
  cursor: z.uuid().optional(),
  limit: z.coerce.number().min(1).max(100).optional().default(20),
})

type CursorQuerySchema = z.infer<typeof cursorQuerySchema>

@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@Controller({ path: '<entities>/feed', version: '1' })
export class List<Entity>sCursorController {
  constructor(private list<Entity>sCursorUseCase: List<Entity>sCursorUseCase) {}

  @Get()
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('list', '<Entity>'))
  @HttpCode(200)
  @ApiOperation({ summary: 'List <entities> (cursor pagination)' })
  @ApiQuery({ name: 'cursor', required: false, type: String })
  @ApiQuery({ name: 'limit', required: false, type: Number })
  @ApiOkResponse({ type: <Entity>ResponseDto, isArray: true })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @Query(new ZodValidationPipe(cursorQuerySchema)) query: CursorQuerySchema,
  ) {
    const result = await this.list<Entity>sCursorUseCase.execute(query)

    if (result.isLeft()) throw result.value

    const { items, nextCursor, count } = result.value
    return {
      data: items.map(<Entity>Presenter.toHTTP),
      nextCursor,
      count,
    }
  }
}
```

Response shape: `{ data: [...], nextCursor: string | null, count: number }`. The client sends back `nextCursor` as the `cursor` query param to fetch the next page.

---

## Structure — sub-resource audit logs (per-entity)

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

## Structure — find by ID (resource-level CASL)

When authorization depends on the resource (e.g. user can only read their own record), use a policy handler:

```ts
// src/infra/auth/casl/handlers/<entity>-can-read.handler.ts
import type { ExecutionContext } from '@nestjs/common'
import type { PolicyHandler } from '../check-policies.decorator'

export const <entity>CanReadHandler: PolicyHandler = (ability, context: ExecutionContext) => {
  const req = context.switchToHttp().getRequest()
  const targetId = req.params.id as string
  return ability.can('read', { __typename: '<Entity>' as const, id: targetId })
}
```

```ts
@Get(':id')
@UseGuards(CaslAbilityGuard)
@CheckPolicies(<entity>CanReadHandler)
@HttpCode(200)
@ApiOperation({ summary: 'Find <entity> by ID' })
@ApiOkResponse({ type: <Entity>ResponseDto })
@ApiNotFoundResponse({ type: <Entity>NotFoundDto })
@ApiUnauthorizedResponse({ type: WrongCredentialsDto })
@ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
@ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
async handle(@Param('id', ParseUuidPipe) id: string) {
  const result = await this.find<Entity>ByIdUseCase.execute({ id })
  if (result.isLeft()) throw result.value
  return { data: <Entity>Presenter.toHTTP(result.value.data) }
}
```

---

## Structure — edit (body + param, CASL)

```ts
const edit<Entity>BodySchema = z.object({
  name: z.string().min(1),
  optionalField: z.string().optional(),
})

type Edit<Entity>BodySchema = z.infer<typeof edit<Entity>BodySchema>

@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@Controller({ path: '<entities>', version: '1' })
export class Edit<Entity>Controller {
  constructor(private edit<Entity>UseCase: Edit<Entity>UseCase) {}

  @Put(':id')
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('update', '<Entity>'))
  @HttpCode(200)
  @ApiOperation({ summary: 'Edit <entity>' })
  @ApiBody({ type: Edit<Entity>RequestDto })
  @ApiOkResponse({ type: <Entity>ResponseDto })
  @ApiNotFoundResponse({ type: <Entity>NotFoundDto })
  @ApiBadRequestResponse({ type: BadRequestDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiUnprocessableEntityResponse({ type: UnprocessableEntityDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @CurrentUser() currentUser: UserPayload,
    @Param('id', ParseUuidPipe) id: string,
    @Body(new ZodValidationPipe(edit<Entity>BodySchema)) body: Edit<Entity>BodySchema,
  ) {
    const result = await this.edit<Entity>UseCase.execute({
      id,
      ...body,
      actorId: currentUser.sub,
    })

    if (result.isLeft()) throw result.value

    return {
      data: <Entity>Presenter.toHTTP(result.value.data),
      message: '<Entity> updated successfully',
    }
  }
}
```

---

## Structure — delete (param only, CASL)

The controller does not know whether a hard or soft delete occurred — the use case decides based on domain rules. Always returns 200 with a message.

```ts
@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@Controller({ path: '<entities>', version: '1' })
export class Delete<Entity>Controller {
  constructor(private delete<Entity>UseCase: Delete<Entity>UseCase) {}

  @Delete(':id')
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('delete', '<Entity>'))
  @HttpCode(200)
  @ApiOperation({ summary: 'Delete <entity>' })
  @ApiOkResponse({ description: '<Entity> deleted successfully' })
  @ApiNotFoundResponse({ type: <Entity>NotFoundDto })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @CurrentUser() currentUser: UserPayload,
    @Param('id', ParseUuidPipe) id: string,
  ) {
    const result = await this.delete<Entity>UseCase.execute({
      id,
      actorId: currentUser.sub,
    })

    if (result.isLeft()) throw result.value

    return { message: '<Entity> deleted successfully' }
  }
}
```

---

## Structure — toggle status (param only, CASL)

```ts
@UseFilters(<Domain>ErrorFilter)
@ApiTags('<Entities>')
@Controller({ path: '<entities>', version: '1' })
export class Toggle<Entity>StatusController {
  constructor(private toggle<Entity>StatusUseCase: Toggle<Entity>StatusUseCase) {}

  @Patch(':id/toggle-status')
  @UseGuards(CaslAbilityGuard)
  @CheckPolicies((ability) => ability.can('update', '<Entity>'))
  @HttpCode(200)
  @ApiOperation({ summary: 'Toggle <entity> active status' })
  @ApiOkResponse({ type: <Entity>ResponseDto })
  @ApiNotFoundResponse({ type: <Entity>NotFoundDto })
  @ApiUnauthorizedResponse({ type: WrongCredentialsDto })
  @ApiForbiddenResponse({ type: <Entity>ForbiddenDto })
  @ApiInternalServerErrorResponse({ type: InternalServerErrorDto })
  async handle(
    @CurrentUser() currentUser: UserPayload,
    @Param('id', ParseUuidPipe) id: string,
  ) {
    const result = await this.toggle<Entity>StatusUseCase.execute({
      id,
      actorId: currentUser.sub,
    })

    if (result.isLeft()) throw result.value

    return {
      data: <Entity>Presenter.toHTTP(result.value.data),
      message: '<Entity> status updated successfully',
    }
  }
}
```

---

## DTOs (Swagger documentation)

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

## Error filter

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

## Module

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

## ControllersModule (aggregator)

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

## Pipes

| Pipe | Usage |
|------|-------|
| `ZodValidationPipe(schema)` | Validates `@Body()` and `@Query()` — pass schema inline |
| `ParseUuidPipe` | Validates `@Param('id')` is a valid UUID v4 |

Boolean query params require a string transform:
```ts
active: z.string().optional().transform((val) => {
  if (val === 'true') return true
  if (val === 'false') return false
  return undefined
})
```

---

## Rules

- **Route ordering matters** — in NestJS modules, register more specific routes (`:id/audit-logs`) BEFORE wildcard routes (`:id`). The first matching route wins
- One controller class per endpoint — never group multiple HTTP methods in one class
- Controllers never contain business logic — only: validate input → call use case → format output
- **All endpoints are private by default** — JWT required. Use `@Public()` only when explicitly required
- Always use `ZodValidationPipe` on `@Body()` and `@Query()`
- Always use `ParseUuidPipe` on `@Param('id')`
- Always apply `@ServiceTag('<domain>')` at the class level — used by `RequestLoggingInterceptor` for log filtering
- Always apply `@UseFilters(<Domain>ErrorFilter)`
- Always add full Swagger decorators: `@ApiBody`, `@ApiOkResponse`/`@ApiCreatedResponse`, all error responses
- DTOs are for Swagger only — they describe the API shape, they do not validate
- For resource-level authorization use `@UseGuards(CaslAbilityGuard)` + `@CheckPolicies(handler)`
- For action-level authorization (e.g. role check) use `@CheckPolicies((ability) => ability.can('action', 'Subject'))`
- Pass `actorId: currentUser.sub` to every mutating use case
- Always version: `@Controller({ path: '<entities>', version: '1' })`

---

## Anti-patterns

```ts
// ❌ assuming endpoint is public without explicit requirement
@Public() // only add when explicitly stated in domain rules

// ❌ no CASL guard on protected endpoints
@Post()
async handle() { ... } // missing @UseGuards(CaslAbilityGuard) + @CheckPolicies

// ❌ no Swagger decorators
@Post()
@ApiOperation({ summary: 'Create' }) // incomplete — add all response decorators

// ❌ no validation pipe
async handle(@Body() body: any) { ... }

// ❌ throwing generic HTTP error instead of using filter
if (result.isLeft()) throw new NotFoundException()

// ❌ multiple HTTP methods in one controller class
export class <Entity>Controller {
  @Get() list() {}
  @Post() create() {} // split into separate controllers
}

// ❌ no API versioning
@Controller('<entities>') // always: { path: '<entities>', version: '1' }

// ❌ wildcard route registered before sub-resource route — shadows audit-logs
@Module({
  controllers: [
    FindEntityByIdController,        // @Get(':id') catches everything
    ListEntityAuditLogsController,   // @Get(':id/audit-logs') never reached
  ],
})
```
