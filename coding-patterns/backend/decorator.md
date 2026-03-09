# Decorator Pattern

Custom NestJS decorators for cross-cutting concerns. Decorators are thin wrappers — they extract data or set metadata, never contain business logic.

---

## File locations

```
src/infra/auth/decorators/current-user.decorator.ts  ← extracts JWT payload from request
src/infra/auth/decorators/public.decorator.ts         ← marks endpoint as public
src/infra/decorators/                                 ← other custom decorators
```

---

## Param decorator — extract data from request

```ts
// src/infra/auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common'
import { TokenPayload } from '../jwt/jwt.strategy'

export const CurrentUser = createParamDecorator(
  (_data: unknown, context: ExecutionContext): TokenPayload => {
    const request = context.switchToHttp().getRequest()
    return request.user as TokenPayload
  },
)
```

Usage:
```ts
@Get()
async handle(@CurrentUser() user: TokenPayload) {
  // user.sub, user.role — typed and validated by JWT strategy
}
```

---

## Metadata decorator — set metadata for guards

```ts
// src/infra/auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)
```

Usage:
```ts
@Post('sign-in')
@Public()  // bypasses JwtAuthGuard
async handle() { ... }
```

---

## Service tag decorator — domain identification for logging

```ts
// src/infra/decorators/service-tag.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const SERVICE_TAG = 'SERVICE_TAG'
export const ServiceTag = (tag: string) => SetMetadata(SERVICE_TAG, tag)
```

Usage — applied at the controller class level to identify the domain for the `RequestLoggingInterceptor`:
```ts
@ServiceTag('user')
@Controller({ path: 'users', version: '1' })
export class CreateUserController { ... }
```

The interceptor reads `SERVICE_TAG` via `Reflector` and includes it as `service` in every log entry. If not set, defaults to `'api'`.

---

## Policy decorator — CASL authorization

```ts
// src/infra/auth/casl/check-policies.decorator.ts
import { SetMetadata } from '@nestjs/common'
import { AppAbility } from './casl-ability.factory'

export type PolicyHandler =
  | { handle: (ability: AppAbility) => boolean }
  | ((ability: AppAbility) => boolean)

export const CHECK_POLICIES_KEY = 'check_policies'
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers)
```

Usage:
```ts
@Post()
@UseGuards(CaslAbilityGuard)
@CheckPolicies(new CreateOrderPolicyHandler())
async handle() { ... }

// or inline:
@CheckPolicies((ability) => ability.can('create', 'Order'))
```

---

## Rules

- Decorators are infrastructure — they live in `src/infra/`
- Param decorators extract data, metadata decorators set flags — never mix both
- Never put business logic in a decorator — delegate to guards or use cases
- Always type the return value of param decorators
- Keep decorators small — one responsibility per decorator
- Auth decorators live in `src/infra/auth/decorators/`
- General decorators live in `src/infra/decorators/`

---

## Anti-patterns

```ts
// ❌ business logic in decorator
export const ValidateOrder = createParamDecorator(
  async (data, context) => {
    const order = await repo.findById(id)  // never — use a use case
    if (!order) throw new NotFoundException()
    return order
  },
)

// ❌ untyped param decorator
export const CurrentUser = createParamDecorator(
  (data, context) => {
    return context.switchToHttp().getRequest().user  // add return type
  },
)

// ❌ decorator in domain layer
// src/domain/orders/decorators/  ← decorators are infra, not domain
```
