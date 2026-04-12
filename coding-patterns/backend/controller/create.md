# Controller — Create

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
