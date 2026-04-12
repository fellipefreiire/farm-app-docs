# Controller — Toggle Status

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
