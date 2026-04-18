# Logging Pattern

Winston structured JSON logging. Domain layer never logs — logging lives exclusively in infrastructure layer. Request logging handled automatically by `RequestLoggingInterceptor`, so controllers only log errors and warnings.

---

## File location

```
src/infra/logger/winston/winston.token.ts       <- WINSTON_LOGGER symbol
src/infra/logger/winston/winston.config.ts      <- createDomainLogger factory (kleur colors)
src/infra/logger/winston/winston.provider.ts    <- NestJS provider factory
src/infra/logger/winston/logger.repository.ts   <- abstract class (info, warn, error, debug)
src/infra/logger/winston/logger.service.ts      <- implements LoggerRepository via Winston
src/infra/logger/winston/logger.interceptor.ts  <- RequestLoggingInterceptor (request logging)
src/infra/logger/logger.module.ts               <- NestJS module (@Global, APP_INTERCEPTOR)
```

---

## LoggerRepository abstract class

All logging goes through this abstract class. Concrete implementations (Winston in production, FakeLogger in tests) implement it.

```ts
export abstract class LoggerRepository {
  abstract info(msg: string, meta?: Record<string, unknown>): void
  abstract warn(msg: string, meta?: Record<string, unknown>): void
  abstract error(msg: string, meta?: Record<string, unknown>): void
  abstract debug(msg: string, meta?: Record<string, unknown>): void
}
```

---

## Logger service

`LoggerService` implements `LoggerRepository`. Receives Winston instance via `WINSTON_LOGGER` injection token — never creates logger directly.

```ts
import { Inject, Injectable } from '@nestjs/common'
import { Logger } from 'winston'

import { WINSTON_LOGGER } from './winston.token'
import { LoggerRepository } from './logger.repository'

@Injectable()
export class LoggerService extends LoggerRepository {
  constructor(@Inject(WINSTON_LOGGER) private readonly logger: Logger) {
    super()
  }

  info(message: string, meta?: Record<string, unknown>) {
    this.logger.info(message, meta)
  }

  warn(message: string, meta?: Record<string, unknown>) {
    this.logger.warn(message, meta)
  }

  error(message: string, meta?: Record<string, unknown>) {
    this.logger.error(message, meta)
  }

  debug(message: string, meta?: Record<string, unknown>) {
    this.logger.debug(message, meta)
  }
}
```

---

## Winston config — `createDomainLogger(service)`

Factory creates Winston logger instance for given service/domain.

- `kleur` for colored output in production (human-readable structured logs)
- JSON format in development
- Reads `LOG_LEVEL` from env (defaults to `'info'`)

```ts
import { createDomainLogger } from './winston.config'

const logger = createDomainLogger('order')
```

Provider (`winston.provider.ts`) calls `createDomainLogger` and binds result to `WINSTON_LOGGER` token for injection into `LoggerService`.

---

## Logger module

`@Global` — import once in `AppModule`, `LoggerService` available everywhere. `RequestLoggingInterceptor` registered as global interceptor via `APP_INTERCEPTOR`.

```ts
import { Module, Global } from '@nestjs/common'
import { WinstonLoggerProvider } from './winston/winston.provider'
import { APP_INTERCEPTOR } from '@nestjs/core'
import { LoggerService } from './winston/logger.service'
import { RequestLoggingInterceptor } from './winston/logger.interceptor'

@Global()
@Module({
  providers: [
    WinstonLoggerProvider,
    LoggerService,
    {
      provide: APP_INTERCEPTOR,
      useClass: RequestLoggingInterceptor,
    },
  ],
  exports: [LoggerService],
})
export class LoggerModule {}
```

---

## RequestLoggingInterceptor

Auto-logs all HTTP requests — controllers don't log successful requests manually.

- Uses `@ServiceTag` decorator (via `Reflector`) to identify domain/service
- Maps domain errors to HTTP status codes for error logging
- Success: logs `[status] Request handled` at `info`
- Failure: logs `[status] error message` at `error`
- Meta fields: `service`, `route`, `httpMethod`, `timeToComplete`

Controllers only need `warn` (recoverable issues like not-found) and `error` (unexpected failures). Interceptor handles rest.

---

## FakeLogger (test double)

Use `FakeLogger` in unit tests instead of real `LoggerService`. Collects log messages into arrays for assertions.

```ts
// test/infra/fake-logger.ts
export class FakeLogger {
  public logs: string[] = []
  public warnings: string[] = []
  public errors: string[] = []
  public debugs: string[] = []

  info(message: string) {
    this.logs.push(message)
  }

  warn(message: string) {
    this.warnings.push(message)
  }

  error(message: string) {
    this.errors.push(message)
  }

  debug(message: string) {
    this.debugs.push(message)
  }

  clear() {
    this.logs = []
    this.warnings = []
    this.errors = []
    this.debugs = []
  }
}
```

---

## Usage in controllers

Controllers log `error` and `warn` only. Successful request logging handled by `RequestLoggingInterceptor`.

```ts
@Controller('/v1/orders')
export class OrderController {
  constructor(
    private readonly createOrder: CreateOrderUseCase,
    private readonly logger: LoggerService,
  ) {}

  @Post()
  async create(@Body() body: CreateOrderDTO) {
    const result = await this.createOrder.execute(body)

    if (result.isLeft()) {
      const error = result.value

      if (error instanceof ResourceNotFoundError) {
        this.logger.warn('Order creation failed: resource not found', {
          domain: 'order',
          entityId: body.customerId,
        })

        throw new NotFoundException(error.message)
      }

      this.logger.error('Order creation failed unexpectedly', {
        domain: 'order',
        error: { message: error.message, stack: error.stack },
      })

      throw new InternalServerErrorException()
    }

    return OrderPresenter.toHTTP(result.value.order)
  }
}
```

---

## Usage in event subscribers

Event subscribers log `error` and `info`. Info for business events, error for failures.

```ts
@Injectable()
export class OnOrderCreated implements OnModuleInit {
  constructor(
    private readonly sendNotification: SendNotificationUseCase,
    private readonly logger: LoggerService,
  ) {}

  onModuleInit() {
    DomainEvents.register(this.handle.bind(this), OrderCreatedEvent.name)
  }

  private async handle(event: OrderCreatedEvent) {
    this.logger.info('Order created event received', {
      domain: 'order',
      entityId: event.orderId.toString(),
    })

    const result = await this.sendNotification.execute({
      recipientId: event.customerId.toString(),
      title: 'Order confirmed',
    })

    if (result.isLeft()) {
      this.logger.error('Failed to send order notification', {
        domain: 'notification',
        entityId: event.orderId.toString(),
        error: { message: result.value.message },
      })

      return
    }

    this.logger.info('Order notification sent successfully', {
      domain: 'notification',
      entityId: event.orderId.toString(),
    })
  }
}
```

---

## Log levels

| Level | When | Examples |
|-------|------|----------|
| `error` | Unrecoverable failures — needs human attention | DB connection lost, external API down, unhandled use case error |
| `warn` | Recoverable issues — retries, fallbacks, cache misses | Resource not found, validation failure, rate limit hit |
| `info` | Business events — entity created, payment processed | Order created, user registered, subscription renewed |
| `debug` | Technical details — disabled in production via `LOG_LEVEL` | Query params, cache hit/miss details, event payload |

---

## Required and optional fields

Every log entry must include:

| Field | Type | Required |
|-------|------|----------|
| `level` | string | yes (automatic from method) |
| `message` | string | yes |
| `domain` | string | yes |
| `timestamp` | string | yes (automatic from Winston) |
| `entityId` | string | no |
| `actorId` | string | no |
| `error` | `{ message, stack? }` | no |

Meta typed as `Record<string, unknown>` — include fields relevant to entry.

---

## Rules

- Controllers log `error` and `warn` only — successful request logging handled by `RequestLoggingInterceptor`
- Event subscribers log `error` and `info`
- Always include `domain` field in meta for filtering
- `LOG_LEVEL` env var controls minimum level (`debug` in dev, `info` in production)
- Error objects must be serialized as `{ message, stack? }` — never pass raw Error instances to Winston
- Always inject `LoggerService` — never create Winston logger directly in application code

---

## Anti-patterns

```ts
// WRONG: logging in domain layer
export class Order extends Entity<OrderProps> {
  complete() {
    logger.info('Order completed')  // domain must not log
    this.addDomainEvent(new OrderCompletedEvent(this))
  }
}

// WRONG: logging in use case
export class CreateOrderUseCase {
  async execute(input: Input) {
    this.logger.info('Creating order')  // use case must not log
    // ...
  }
}

// WRONG: console.log in production code
console.log('order created', order)  // use LoggerService

// WRONG: creating Winston logger directly
import { createLogger } from 'winston'
const logger = createLogger({ /* ... */ })  // inject LoggerService instead

// WRONG: logging sensitive data
this.logger.info('User login', {
  domain: 'auth',
  password: input.password,  // never log credentials
  token: jwt,                // never log tokens
})

// WRONG: logging full request body
this.logger.info('Request received', {
  domain: 'order',
  body: req.body,  // may contain PII — log specific fields only
})

// WRONG: raw Error instance
this.logger.error('Failed', {
  domain: 'order',
  error: error,  // always: { message: error.message, stack: error.stack }
})
```