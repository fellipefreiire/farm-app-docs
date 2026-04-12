# Controller — Find by ID

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
