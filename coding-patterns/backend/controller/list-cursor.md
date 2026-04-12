# Controller — List (cursor pagination)

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
