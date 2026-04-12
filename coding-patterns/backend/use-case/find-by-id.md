# Use Case — Find by ID

> Part of the farm-app backend use case pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Two variants depending on the use case's purpose:

```ts
type Find<Entity>ByIdUseCaseRequest = {
  id: string
}

type Find<Entity>ByIdUseCaseResponse = Either<
  <Entity>NotFoundError,
  {
    data: <Entity>
  }
>

/** Finds a <entity> by ID for display. */
@Injectable()
export class Find<Entity>ByIdUseCase {
  constructor(private <entity>Repository: <Entity>Repository) {}

  async execute({ id }: Find<Entity>ByIdUseCaseRequest): Promise<Find<Entity>ByIdUseCaseResponse> {
    const entity = await this.<entity>Repository.findActiveById(id)

    if (!entity) {
      return left(new <Entity>NotFoundError())
    }

    return right({ data: entity })
  }
}
```

> **`findById()` vs `findActiveById()`** — use `findActiveById()` for standard reads (display, listings), edit, and toggle operations. Use `findById()` only when the use case needs the entity regardless of soft-delete status: delete and audit scenarios.

---
