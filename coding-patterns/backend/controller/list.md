# Controller — List (offset pagination)

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
