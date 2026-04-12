# Event Subscriber Unit Tests

> Part of the farm-app backend test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Subscriber unit tests verify that a domain event subscriber calls the correct use case when an event is dispatched. They use InMemory repositories and `DomainEvents.dispatchEventsForAggregate()` directly — no HTTP layer.

```ts
// src/domain/<subscriber-domain>/application/subscribers/<source-domain>/__tests__/on-<entity>-<action>.spec.ts
import { vi } from 'vitest'
import { waitFor } from 'test/utils/wait-for'
import { make<Entity> } from 'test/factories/make-<entity>'
import { DomainEvents } from '@/core/events/domain-events'
import { On<Entity><Action> } from '@/domain/<subscriber-domain>/application/subscribers/<source-domain>/on-<entity>-<action>'
import { InMemory<Entity>sRepository } from 'test/repositories/<source-domain>/in-memory-<entity>s-repository'
import { InMemory<SideEffect>Repository } from 'test/repositories/<subscriber-domain>/in-memory-<side-effect>-repository'
import { <SideEffect>UseCase } from '@/domain/<subscriber-domain>/application/use-cases/<side-effect-action>'

let inMemory<Entity>sRepository: InMemory<Entity>sRepository
let inMemory<SideEffect>Repository: InMemory<SideEffect>Repository
let <sideEffect>UseCase: <SideEffect>UseCase
let useCaseSpy: ReturnType<typeof vi.spyOn>

describe('On <Entity> <Action> (subscriber)', () => {
  beforeEach(() => {
    inMemory<Entity>sRepository = new InMemory<Entity>sRepository()
    inMemory<SideEffect>Repository = new InMemory<SideEffect>Repository()
    <sideEffect>UseCase = new <SideEffect>UseCase(inMemory<SideEffect>Repository)
    useCaseSpy = vi.spyOn(<sideEffect>UseCase, 'execute')

    new On<Entity><Action>(<sideEffect>UseCase)
  })

  it('should create <side-effect> when <entity> is <action>', async () => {
    // Arrange
    const entity = make<Entity>()
    inMemory<Entity>sRepository.create(entity)

    // Act — trigger the domain event
    entity.<action>(entity.id.toString())
    DomainEvents.dispatchEventsForAggregate(entity.id)

    // Assert
    await waitFor(() => {
      expect(useCaseSpy).toHaveBeenCalled()
    })

    expect(inMemory<SideEffect>Repository.items).toHaveLength(1)
    expect(inMemory<SideEffect>Repository.items[0]).toEqual(
      expect.objectContaining({
        action: '<source-domain>:<action>',
        entityId: entity.id.toString(),
        actorId: entity.id.toString(),
      }),
    )
  })
})
```

**Rules:**
- Use `vi.spyOn` on the use case's `execute` method — never mock the entire use case
- Use `waitFor()` for assertions — subscribers are async
- Trigger events via entity methods + `DomainEvents.dispatchEventsForAggregate()` (not via HTTP)
- File location: `src/domain/<subscriber-domain>/application/subscribers/<source-domain>/__tests__/`
- These are unit tests (`.spec.ts`) — InMemory repos, no database

---
