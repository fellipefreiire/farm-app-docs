# Core Primitive Unit Tests

> Part of the farm-app backend test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

Core primitives (`Either`, `DomainEvents`, `WatchedList`, `UniqueEntityID`) are foundational building blocks. Each must have a dedicated test file in `src/core/__tests__/`.

## Either test

```ts
// src/core/__tests__/either.spec.ts
import { type Either, left, right } from '../either'

function doSomething(shouldSuccess: boolean): Either<string, number> {
  if (shouldSuccess) {
    return right(10)
  }

  return left('error')
}

test('success', () => {
  const result = doSomething(true)

  expect(result.isRight()).toBe(true)
  expect(result.isLeft()).toBe(false)
})

test('error', () => {
  const result = doSomething(false)

  expect(result.isRight()).toBe(false)
  expect(result.isLeft()).toBe(true)
})
```

## DomainEvents test

Tests the register → dispatch → callback cycle. Creates inline aggregate and event classes for isolation.

```ts
// src/core/__tests__/domain-events.spec.ts
import { DomainEvent } from '../events/domain-event'
import { UniqueEntityID } from '../entities/unique-entity-id'
import { AggregateRoot } from '../entities/aggregate-root'
import { DomainEvents } from '@/core/events/domain-events'
import { vi } from 'vitest'

class CustomAggregateCreated implements DomainEvent {
  public occurredAt: Date
  private aggregate: CustomAggregate

  constructor(aggregate: CustomAggregate) {
    this.aggregate = aggregate
    this.occurredAt = new Date()
  }

  public getAggregateId(): UniqueEntityID {
    return this.aggregate.id
  }
}

class CustomAggregate extends AggregateRoot<null> {
  static create() {
    const aggregate = new CustomAggregate(null)
    aggregate.addDomainEvent(new CustomAggregateCreated(aggregate))
    return aggregate
  }
}

describe('domain events', () => {
  it('should be able to dispatch and listen to events', async () => {
    const callbackSpy = vi.fn()

    // Register subscriber
    DomainEvents.register(callbackSpy, CustomAggregateCreated.name)

    // Create aggregate (event queued, not dispatched)
    const aggregate = CustomAggregate.create()
    expect(aggregate.domainEvents).toHaveLength(1)

    // Dispatch (simulates repository save)
    DomainEvents.dispatchEventsForAggregate(aggregate.id)

    expect(callbackSpy).toHaveBeenCalled()
    expect(aggregate.domainEvents).toHaveLength(0)
  })
})
```

## WatchedList test

Tests add, remove, update tracking, and re-add/re-remove edge cases.

```ts
// src/core/__tests__/watched-list.spec.ts
import { WatchedList } from '@/core/entities/watched-list'

class NumberWatchedList extends WatchedList<number> {
  compareItems(a: number, b: number): boolean {
    return a === b
  }
}

describe('watched list', () => {
  it('should be able to create a watched list with initial items', () => {
    const list = new NumberWatchedList([1, 2, 3])
    expect(list.currentItems).toHaveLength(3)
  })

  it('should be able to add new items to the list', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.add(4)
    expect(list.currentItems).toHaveLength(4)
    expect(list.getNewItems()).toEqual([4])
  })

  it('should be able to remove items from the list', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.remove(2)
    expect(list.currentItems).toHaveLength(2)
    expect(list.getRemovedItems()).toEqual([2])
  })

  it('should be able to add an item even if it was removed before', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.remove(2)
    list.add(2)
    expect(list.currentItems).toHaveLength(3)
    expect(list.getRemovedItems()).toEqual([])
    expect(list.getNewItems()).toEqual([])
  })

  it('should be able to remove an item even if it was added before', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.add(4)
    list.remove(4)
    expect(list.currentItems).toHaveLength(3)
    expect(list.getRemovedItems()).toEqual([])
    expect(list.getNewItems()).toEqual([])
  })

  it('should be able to update watched list items', () => {
    const list = new NumberWatchedList([1, 2, 3])
    list.update([1, 3, 5])
    expect(list.getRemovedItems()).toEqual([2])
    expect(list.getNewItems()).toEqual([5])
  })
})
```

**Rules:**
- One test file per core primitive
- Core primitive tests are pure unit tests — no database, no DI
- DomainEvents test must use inline aggregate/event classes (never import real domain entities)
- WatchedList test must use a concrete subclass (e.g. `NumberWatchedList`)
- Every new core primitive must have a corresponding test

---
