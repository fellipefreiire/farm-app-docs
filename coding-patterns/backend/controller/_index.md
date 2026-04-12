# Backend Controller Pattern — Index

Navigation hub for farm-app backend controller patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree** —
layer-scoping and context-mode sandboxing only pay off if this discipline is respected.

## Files

| Variant | File | When to read |
|---|---|---|
| Create (POST with body) | [create.md](create.md) | Implementing a create endpoint (with body, CASL, full Swagger) |
| List (offset pagination) | [list.md](list.md) | Implementing a list endpoint with query params |
| List (cursor pagination) | [list-cursor.md](list-cursor.md) | Implementing a cursor-paginated list |
| Audit logs (sub-resource) | [audit-logs.md](audit-logs.md) | Implementing a per-entity audit log endpoint |
| Find by ID | [find-by-id.md](find-by-id.md) | Implementing a GET /:id endpoint with resource-level CASL |
| Edit | [edit.md](edit.md) | Implementing an edit endpoint (body + param) |
| Delete | [delete.md](delete.md) | Implementing a delete endpoint |
| Toggle status | [toggle-status.md](toggle-status.md) | Implementing a status-toggle endpoint |
| Swagger DTOs | [dtos.md](dtos.md) | Writing OpenAPI/Swagger response DTOs |
| Error filter | [error-filter.md](error-filter.md) | Wiring a domain error filter for a controller |
| Module | [module.md](module.md) | Wiring a controller into its NestJS module |
| ControllersModule aggregator | [controllers-module.md](controllers-module.md) | Registering in the top-level controllers aggregator |
| Pipes | [pipes.md](pipes.md) | Wiring Zod validation pipes for a controller |

## File locations

```
src/infra/http/controllers/<domain>/create-<entity>.controller.ts
src/infra/http/controllers/<domain>/list-<entity>s.controller.ts
src/infra/http/controllers/<domain>/find-<entity>-by-id.controller.ts
src/infra/http/controllers/<domain>/edit-<entity>.controller.ts
src/infra/http/controllers/<domain>/delete-<entity>.controller.ts
src/infra/http/controllers/<domain>/toggle-<entity>-status.controller.ts
src/infra/http/controllers/<domain>/<domain>-controllers.module.ts
src/infra/http/filters/<domain>-error.filter.ts
src/infra/http/dtos/requests/<domain>/         ← request DTOs (Swagger docs)
src/infra/http/dtos/response/<domain>/         ← response DTOs (Swagger docs)
src/infra/http/dtos/error/<domain>/            ← domain-specific error DTOs
src/infra/auth/casl/handlers/<entity>-<action>.handler.ts  ← CASL policy handlers
```

One controller per endpoint. One file per controller.

---

## Authentication — default behavior

**All endpoints require JWT authentication by default.** The `JwtAuthGuard` is applied globally.

Use `@Public()` **only when explicitly required** — e.g. sign-in, sign-up, public listings. Never assume an endpoint is public. If the domain rules file does not explicitly state an endpoint is public, it is private.

---

## Rules

- **Route ordering matters** — in NestJS modules, register more specific routes (`:id/audit-logs`) BEFORE wildcard routes (`:id`). The first matching route wins
- One controller class per endpoint — never group multiple HTTP methods in one class
- Controllers never contain business logic — only: validate input → call use case → format output
- **All endpoints are private by default** — JWT required. Use `@Public()` only when explicitly required
- Always use `ZodValidationPipe` on `@Body()` and `@Query()`
- Always use `ParseUuidPipe` on `@Param('id')`
- Always apply `@ServiceTag('<domain>')` at the class level — used by `RequestLoggingInterceptor` for log filtering
- Always apply `@UseFilters(<Domain>ErrorFilter)`
- Always add full Swagger decorators: `@ApiBody`, `@ApiOkResponse`/`@ApiCreatedResponse`, all error responses
- DTOs are for Swagger only — they describe the API shape, they do not validate
- For resource-level authorization use `@UseGuards(CaslAbilityGuard)` + `@CheckPolicies(handler)`
- For action-level authorization (e.g. role check) use `@CheckPolicies((ability) => ability.can('action', 'Subject'))`
- Pass `actorId: currentUser.sub` to every mutating use case
- Always version: `@Controller({ path: '<entities>', version: '1' })`

---

## Anti-patterns

```ts
// ❌ assuming endpoint is public without explicit requirement
@Public() // only add when explicitly stated in domain rules

// ❌ no CASL guard on protected endpoints
@Post()
async handle() { ... } // missing @UseGuards(CaslAbilityGuard) + @CheckPolicies

// ❌ no Swagger decorators
@Post()
@ApiOperation({ summary: 'Create' }) // incomplete — add all response decorators

// ❌ no validation pipe
async handle(@Body() body: any) { ... }

// ❌ throwing generic HTTP error instead of using filter
if (result.isLeft()) throw new NotFoundException()

// ❌ multiple HTTP methods in one controller class
export class <Entity>Controller {
  @Get() list() {}
  @Post() create() {} // split into separate controllers
}

// ❌ no API versioning
@Controller('<entities>') // always: { path: '<entities>', version: '1' }

// ❌ wildcard route registered before sub-resource route — shadows audit-logs
@Module({
  controllers: [
    FindEntityByIdController,        // @Get(':id') catches everything
    ListEntityAuditLogsController,   // @Get(':id/audit-logs') never reached
  ],
})
```
