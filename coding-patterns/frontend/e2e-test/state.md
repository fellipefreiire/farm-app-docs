# E2E — State management

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

MSW handler state lives in the **Next.js server process** as module-level variables. This state persists across all tests within a Playwright run.

## Key constraints

1. **No state reset between tests** — the handler state accumulates across the entire test run
2. **Tests within a file run serially** — use `test.describe.serial()` to guarantee order
3. **Test files run serially** — `workers: 1` in playwright.config.ts prevents cross-file conflicts
4. **Seed data loads once** — when the Next.js server starts, handler modules initialize with seed data
5. **Browser-side requests are NOT intercepted** — only server-side requests go through MSW

## Implications for test design

- **Order tests carefully**: list → search → create → edit → toggle/activate → delete
- **Delete tests should run last** — they remove entities from shared state
- **Don't delete seed entities needed by other test files** — e.g., crop-type delete should not delete the seed crop type used by variety tests
- **Use `first()` or `last()` with regex testid selectors** when you don't know exact IDs (entities created by previous tests)

---
