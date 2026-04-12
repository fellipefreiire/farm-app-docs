# Controller — Edit

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

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
