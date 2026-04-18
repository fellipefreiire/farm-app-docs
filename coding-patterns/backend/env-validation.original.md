# Env Validation Pattern

Zod-based environment variable validation at application startup. Fails fast if required variables are missing or malformed.

---

## File locations

```
src/infra/env/env.ts             ← Zod schema + type export
src/infra/env/env.module.ts      ← NestJS module
src/infra/env/env.service.ts     ← injectable service
test/infra/fake-env.ts           ← fake EnvService for unit tests
.env.example                     ← development env template
.env.test.example                ← E2E test env template (overrides for test runs)
```

---

## Env schema

```ts
// src/infra/env/env.ts
import { z } from 'zod'

export const envSchema = z.object({
  // Server
  PORT: z.coerce.number().default(3333),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),

  // Database
  DATABASE_URL: z.url(),

  // Auth (RS256 key pair)
  JWT_PRIVATE_KEY: z.string(),
  JWT_PUBLIC_KEY: z.string(),

  // Redis
  REDIS_HOST: z.string().optional().default('127.0.0.1'),
  REDIS_PORT: z.coerce.number().optional().default(6379),
  REDIS_DB: z.coerce.number().optional().default(0),
  REDIS_COMMAND_TIMEOUT: z.coerce.number().optional().default(5000),

  // Rate limiting
  RATE_LIMIT_POINTS: z.coerce.number().optional().default(100),
  RATE_LIMIT_DURATION: z.coerce.number().optional().default(60),

  // Logging
  LOG_LEVEL: z.enum(['error', 'warn', 'info', 'debug']).default('info'),

  // CORS
  CORS_ORIGINS: z.string().optional().default(''),
})

export type Env = z.infer<typeof envSchema>
```

---

## Env service

```ts
// src/infra/env/env.service.ts
import { Injectable } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import { Env } from './env'

@Injectable()
export class EnvService {
  constructor(private configService: ConfigService<Env, true>) {}

  get<T extends keyof Env>(key: T): Env[T] {
    return this.configService.get(key, { infer: true })
  }
}
```

---

## Env module

```ts
// src/infra/env/env.module.ts
import { Global, Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'
import { envSchema } from './env'
import { EnvService } from './env.service'

@Global()
@Module({
  imports: [
    ConfigModule.forRoot({
      validate: (env) => envSchema.parse(env),
      isGlobal: true,
    }),
  ],
  providers: [EnvService],
  exports: [EnvService],
})
export class EnvModule {}
```

---

## Usage

```ts
// In any injectable service
@Injectable()
export class SomeService {
  constructor(private env: EnvService) {}

  doSomething() {
    const dbUrl = this.env.get('DATABASE_URL')  // typed, validated
    const port = this.env.get('PORT')            // number, not string
  }
}
```

---

## Test helper — fake env

```ts
// test/infra/fake-env.ts
import type { Env } from '@/infra/env/env'
import { EnvService } from '@/infra/env/env.service'
import type { ConfigService } from '@nestjs/config'

const fakeValues: Env = {
  NODE_ENV: 'test',
  PORT: 3333,
  DATABASE_URL: 'postgres://user:pass@localhost:5432/db',
  JWT_PRIVATE_KEY: 'private-key',
  JWT_PUBLIC_KEY: 'public-key',
  REDIS_HOST: '127.0.0.1',
  REDIS_PORT: 6379,
  REDIS_DB: 0,
  REDIS_COMMAND_TIMEOUT: 1000,
  RATE_LIMIT_POINTS: 10,
  RATE_LIMIT_DURATION: 60,
  LOG_LEVEL: 'debug',
  CORS_ORIGINS: '',
}

const fakeConfigService = {
  get: <T extends keyof Env>(key: T) => fakeValues[key],
} as ConfigService<Env, true>

export const makeFakeEnvService = () => {
  return new EnvService(fakeConfigService)
}
```

Use `makeFakeEnvService()` in unit tests instead of mocking `ConfigService` or instantiating `EnvService` manually. The fake values are deterministic and match the schema defaults where appropriate.

---

## Environment files

Two env files, both with `.example` templates checked into git:

| File | Purpose | Loaded by |
|------|---------|-----------|
| `.env` | Development defaults | `ConfigModule.forRoot()` (app startup) and `setup-e2e.ts` (first) |
| `.env.test` | E2E test overrides | `setup-e2e.ts` (second, with `override: true`) |

`.env.test` only needs variables that **differ** from `.env` — typically `DATABASE_URL` (separate test database) and `REDIS_DB` (dedicated index to avoid flushing dev data). All other values are inherited from `.env`.

```
# .env.test.example
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/project_test?schema=public"
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_DB=1
```

The E2E setup (`test/setup-e2e.ts`) loads both files in order:

```ts
config({ path: '.env', override: true })
config({ path: '.env.test', override: true })
```

This means `.env.test` values win over `.env` values. Each E2E run creates an isolated Postgres schema (random UUID) and flushes the Redis test DB before starting.

**Setup for a new project:**
```bash
cp .env.example .env
cp .env.test.example .env.test
# Edit values as needed
```

---

## Rules

- Every environment variable must be declared in `envSchema` — never read `process.env` directly in application code
- Use `z.coerce.number()` for numeric env vars — they arrive as strings
- Use `.default()` for optional vars with sensible defaults
- Use `.optional()` only for truly optional vars (e.g., Redis in dev)
- `EnvModule` is `@Global()` — available everywhere without importing
- App fails fast on startup if validation fails — never silently use undefined values
- When adding a new env var: add to schema, add to `.env.example` with placeholder, and to `.env.test.example` if it needs a different test value
- `.env` and `.env.test` are in `.gitignore` — only `.example` files are committed

---

## Anti-patterns

```ts
// ❌ reading process.env directly
const key = process.env.JWT_PRIVATE_KEY  // use EnvService.get('JWT_PRIVATE_KEY')

// ❌ no validation on startup
ConfigModule.forRoot()  // always pass validate function

// ❌ missing from .env.example
// new var added to schema but not documented in .env.example

// ❌ string where number expected
PORT: z.string()  // use z.coerce.number()

// ❌ optional without default for required infra
DATABASE_URL: z.string().optional()  // DB is required — don't make it optional

// ❌ instantiating EnvService manually in tests
const env = new EnvService(configService)  // use makeFakeEnvService() from test/infra/fake-env.ts
```
