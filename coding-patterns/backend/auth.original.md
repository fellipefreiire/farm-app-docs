# Auth Pattern

JWT authentication (RS256) + CASL authorization (MongoAbility) + CSRF protection. All endpoints are private by default — `@Public()` is the exception, not the rule.

> Controllers use auth decorators — see `controller.md` for full endpoint examples. This file covers the auth infrastructure itself.

---

## File locations

```
src/infra/auth/jwt/jwt.strategy.ts              ← Passport JWT strategy (RS256)
src/infra/auth/jwt/jwt-auth.guard.ts            ← global guard (applied in AppModule)
src/infra/auth/jwt/jwt.module.ts                ← JwtModule config (RS256, TokenRepository, RefreshTokenRepository)
src/shared/cryptography/token-repository.ts     ← abstract TokenRepository (generateAccessToken, generateRefreshToken)
src/infra/auth/jwt/token.service.ts             ← TokenService implementation
src/shared/cryptography/refresh-token-repository.ts ← abstract RefreshTokenRepository (create, validate, revoke, revokeAllForUser, revokeAllForUserExcept)
src/infra/auth/jwt/refresh-token.service.ts     ← Redis-backed refresh token service
src/infra/auth/guards/csrf.guard.ts             ← CSRF protection guard
src/infra/auth/casl/casl-ability.factory.ts     ← defines abilities per role (MongoAbility)
src/infra/auth/casl/casl-ability.guard.ts       ← guard that checks policies
src/infra/auth/casl/check-policies.decorator.ts ← decorator for policy handlers
src/infra/auth/casl/subjects/                   ← Zod-typed subject definitions
src/infra/auth/casl/permissions/                ← role-based permission definitions
src/infra/auth/decorators/current-user.decorator.ts  ← extracts user from request
src/infra/auth/decorators/public.decorator.ts        ← marks endpoint as public
```

---

## JWT strategy

RS256 asymmetric signing. The strategy uses the **public key** to verify tokens. The private key is only used by `TokenService` when signing.

```ts
// src/infra/auth/jwt/jwt.strategy.ts
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { ExtractJwt, Strategy } from 'passport-jwt'
import { z } from 'zod'
import { EnvService } from '../../env/env.service'

const tokenPayloadSchema = z.object({
  sub: z.uuid(),
  role: z.string(),
  iat: z.number(),
  exp: z.number(),
  jti: z.uuid(),
})

export type TokenPayload = z.infer<typeof tokenPayloadSchema>

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: EnvService) {
    const publicKey = config.get('JWT_PUBLIC_KEY')

    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: Buffer.from(publicKey, 'base64'),
      algorithms: ['RS256'],
    })
  }

  async validate(payload: TokenPayload) {
    return tokenPayloadSchema.parse(payload)
  }
}
```

---

## TokenRepository

Abstract token generation contract. Use cases depend on this abstraction, never on `JwtService` directly.

```ts
// src/infra/auth/jwt/token-repository.ts
export abstract class TokenRepository {
  abstract generateAccessToken(payload: {
    sub: string
    role: string
    jti: string
  }): Promise<{ token: string; expiresIn: number }>

  abstract generateRefreshToken(payload: {
    sub: string
    role: string
    jti: string
  }): Promise<{ token: string; expiresIn: number }>
}
```

`TokenService` implements this using `@nestjs/jwt` with the RS256 private key. The implementation lives in `src/infra/auth/jwt/token.service.ts`.

---

## RefreshTokenRepository

Abstract contract for refresh token lifecycle management. Backed by Redis in production.

```ts
// src/infra/auth/jwt/refresh-token-repository.ts
export abstract class RefreshTokenRepository {
  abstract create(userId: string): Promise<string>
  abstract validate(jti: string): Promise<boolean>
  abstract revoke(jti: string): Promise<void>
  abstract revokeAllForUser(userId: string): Promise<void>
  abstract revokeAllForUserExcept(userId: string, exceptJti: string): Promise<void>
}
```

`RefreshTokenService` implements this using Redis. The implementation lives in `src/infra/auth/jwt/refresh-token.service.ts`.

---

## JwtModule

Wires RS256 config, Passport, and token abstractions together.

```ts
// src/infra/auth/jwt/jwt.module.ts
@Module({
  imports: [
    PassportModule,
    NestJwtModule.registerAsync({
      // RS256 config with private/public keys from EnvService
    }),
    CryptographyModule,
    CacheModule,
  ],
  providers: [
    JwtStrategy,
    { provide: TokenRepository, useClass: TokenService },
    { provide: RefreshTokenRepository, useClass: RefreshTokenService },
  ],
  exports: [NestJwtModule, TokenRepository, RefreshTokenRepository],
})
export class JwtModule {}
```

---

## CSRF guard

Protects state-changing endpoints that use cookies (e.g., refresh token rotation). Compares the `x-csrf-token` header against the `csrfToken` cookie.

```ts
// src/infra/auth/guards/csrf.guard.ts
import { CanActivate, ExecutionContext, ForbiddenException, Injectable } from '@nestjs/common'
import { Request } from 'express'

@Injectable()
export class CsrfGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>()
    const cookies = request.cookies ?? {}
    const csrfHeader = request.headers['x-csrf-token']
    const csrfCookie = cookies['csrfToken']

    if (!csrfHeader || !csrfCookie || csrfHeader !== csrfCookie) {
      throw new ForbiddenException('Invalid CSRF token')
    }

    return true
  }
}
```

---

## CASL ability factory

Uses `MongoAbility` for subject-based permission checks. Subjects are defined with Zod in `src/infra/auth/casl/subjects/`, and role-based permissions are defined in `src/infra/auth/casl/permissions/`.

```ts
// src/infra/auth/casl/casl-ability.factory.ts
import {
  AbilityBuilder,
  createMongoAbility,
  type CreateAbility,
  type MongoAbility,
} from '@casl/ability'
import { Injectable } from '@nestjs/common'

export type AppAbility = MongoAbility<AppAbilities>
export const createAppAbility = createMongoAbility as CreateAbility<AppAbility>

@Injectable()
export class CaslAbilityFactory {
  defineAbilityFor(user: User) {
    const builder = new AbilityBuilder(createAppAbility)

    // ... role-based permissions loaded from permissions/ files

    const ability = builder.build({
      detectSubjectType(subject) {
        return subject.__typename
      },
    })

    ability.can = ability.can.bind(ability)
    ability.cannot = ability.cannot.bind(ability)

    return ability
  }
}
```

---

## CASL guard

```ts
// src/infra/auth/casl/casl-ability.guard.ts
import { CanActivate, ExecutionContext, Injectable, ForbiddenException } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { CaslAbilityFactory, AppAbility } from './casl-ability.factory'
import { CHECK_POLICIES_KEY, PolicyHandler } from './check-policies.decorator'

@Injectable()
export class CaslAbilityGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const handlers = this.reflector.get<PolicyHandler[]>(
      CHECK_POLICIES_KEY,
      context.getHandler(),
    ) || []

    if (handlers.length === 0) return true

    const { user } = context.switchToHttp().getRequest()
    const ability = this.caslAbilityFactory.defineAbilityFor(user)

    const allowed = handlers.every((handler) =>
      typeof handler === 'function'
        ? handler(ability)
        : handler.handle(ability),
    )

    if (!allowed) throw new ForbiddenException()

    return true
  }
}
```

---

## Decorators

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

```ts
// src/infra/auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common'

export const IS_PUBLIC_KEY = 'isPublic'
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true)
```

---

## Usage in controllers

Use inline policy handlers with `@CheckPolicies`:

```ts
@Controller({ path: 'orders', version: '1' })
@UseGuards(CaslAbilityGuard)
export class CreateOrderController {
  @Post()
  @CheckPolicies((ability) => ability.can('create', 'Order'))
  async handle(
    @CurrentUser() currentUser: TokenPayload,
    @Body(new ZodValidationPipe(createOrderSchema)) body: CreateOrderDTO,
  ) {
    const result = await this.createOrder.execute({
      ...body,
      actorId: currentUser.sub,
    })
    // ...
  }
}
```

For public endpoints:

```ts
@Controller({ path: 'auth', version: '1' })
export class SignInController {
  @Post('sign-in')
  @Public()
  async handle(@Body(new ZodValidationPipe(signInSchema)) body: SignInDTO) {
    // ...
  }
}
```

For refresh token endpoints (CSRF-protected):

```ts
@Controller({ path: 'auth', version: '1' })
export class RefreshTokenController {
  @Post('refresh')
  @Public()
  @UseGuards(CsrfGuard)
  async handle(@Req() request: Request) {
    // validate refresh token, revoke old one, issue new pair
  }
}
```

---

## Resource-level authorization

When a user should only access their own resources:

```ts
// In the use case — check ownership
async execute({ entityId, actorId }: Input) {
  const entity = await this.repository.findById(entityId)

  if (!entity) return left(new EntityNotFoundError())
  if (entity.ownerId.toString() !== actorId) return left(new NotAllowedError())

  // proceed
}
```

Ownership checks live in use cases, not in guards. Guards handle role-based access; use cases handle resource-level access.

---

## Rules

- `JwtAuthGuard` is applied globally — every endpoint requires a token unless decorated with `@Public()`
- `@Public()` only when explicitly required by domain rules — never assume
- `@UseGuards(CaslAbilityGuard)` on every controller that needs action-level authorization
- `@CheckPolicies(handler)` on every endpoint that needs policy checks — prefer inline handlers: `@CheckPolicies((ability) => ability.can('create', 'Order'))`
- Always pass `actorId: currentUser.sub` to mutating use cases
- Ownership checks live in use cases, not in guards
- Token payload is validated with Zod — never trust raw JWT claims
- Roles are defined in the CASL ability factory — one place to manage permissions
- RS256 asymmetric key pair — `JWT_PRIVATE_KEY` and `JWT_PUBLIC_KEY` are base64-encoded environment variables
- `TokenPayload` includes `jti` (JWT ID) — mandatory for token revocation and refresh token rotation
- Refresh token rotation — the old refresh token is revoked every time a new token pair is issued
- CSRF guard is required on the refresh token endpoint (and any endpoint that reads tokens from cookies)
- Use cases depend on `TokenRepository` and `RefreshTokenRepository` abstractions, never on `JwtService` directly
- Subjects for CASL are defined with Zod in `src/infra/auth/casl/subjects/`
- Role-based permissions are defined in `src/infra/auth/casl/permissions/`

---

## Anti-patterns

```ts
// WRONG: missing @Public() on sign-in endpoint
@Post('sign-in')
async handle() { ... }  // will require JWT — unauthenticated users can't sign in

// WRONG: checking roles in controller instead of CASL
if (currentUser.role !== 'admin') throw new ForbiddenException()
// use: @CheckPolicies((ability) => ability.can('manage', 'all'))

// WRONG: ownership check in guard instead of use case
canActivate(context) {
  const entity = await this.repo.findById(id)
  return entity.ownerId === user.sub  // move to use case
}

// WRONG: trusting JWT payload without validation
async validate(payload: any) {
  return payload  // always validate with Zod schema
}

// WRONG: hardcoding roles in controllers
if (user.role === 'admin') { ... }  // use CASL abilities

// WRONG: missing actorId in mutating use case call
const result = await this.createOrder.execute({ name })
// always: { name, actorId: currentUser.sub }

// WRONG: using JwtService directly in a use case
constructor(private jwtService: JwtService) {}  // use TokenRepository abstraction

// WRONG: not revoking old refresh token on rotation
const newTokens = await this.tokenRepository.generateAccessToken(...)
// must revoke old refresh token first via RefreshTokenRepository.revoke()

// WRONG: refresh endpoint without CSRF protection
@Post('refresh')
@Public()
async handle() { ... }  // must add @UseGuards(CsrfGuard)

// WRONG: using HS256 symmetric secret
super({ secretOrKey: process.env.JWT_SECRET })  // use RS256 with public key

// WRONG: using PureAbility instead of MongoAbility
const { can, build } = new AbilityBuilder<AppAbility>(PureAbility)
// use: new AbilityBuilder(createAppAbility) with MongoAbility

// WRONG: calling createForUser instead of defineAbilityFor
const ability = this.caslAbilityFactory.createForUser(user)
// use: this.caslAbilityFactory.defineAbilityFor(user)
```
