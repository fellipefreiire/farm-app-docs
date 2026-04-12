# Controller — Delete

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
