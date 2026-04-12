# Controller — Pipes

> Part of the farm-app backend controller pattern collection. Read `_index.md` first for shared rules and anti-patterns.

| Pipe | Usage |
|------|-------|
| `ZodValidationPipe(schema)` | Validates `@Body()` and `@Query()` — pass schema inline |
| `ParseUuidPipe` | Validates `@Param('id')` is a valid UUID v4 |

Boolean query params require a string transform:
```ts
active: z.string().optional().transform((val) => {
  if (val === 'true') return true
  if (val === 'false') return false
  return undefined
})
```

---
