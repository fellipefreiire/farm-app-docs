# E2E — Running tests

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

```bash
# Run all E2E tests
cd frontend && pnpm test:e2e

# Run a specific test file
cd frontend && npx playwright test tests/field/manage-fields.e2e-spec.ts

# Run with UI mode (interactive debugging)
cd frontend && npx playwright test --ui

# Run with headed browser (see the browser)
cd frontend && npx playwright test --headed

# Show HTML report after run
cd frontend && npx playwright show-report
```

## Pre-run cleanup

If tests fail on `page.goto()` or the dev server doesn't start, check for stale processes:

```bash
# Remove stale lock file (left by crashed dev server)
rm -f frontend/.next/dev/lock

# Kill residual Next.js processes on E2E port
lsof -i :4001 -sTCP:LISTEN   # check if something is listening
pkill -f "next dev"           # kill if needed
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| All auth tests timeout on `waitForURL('/dashboard')` | Dev server running on dev port without MSW, Playwright reusing it | Use dedicated port (DEV_PORT + 1, currently 4001). Ensure no stale server on that port. |
| `page.goto()` fails with connection refused | `.next/dev/lock` file prevents server startup | `rm -f frontend/.next/dev/lock` |
| ZodError on page load | MSW handler response shape doesn't match Zod schema | Compare handler interface with `src/domains/<domain>/schemas/`. Common: missing new field added to entity. |
| `strict mode violation: resolved to 2 elements` | `getByText` matches both table cell and toast | Use `getByRole('cell', { name: '...' })` for table data |
| Edit test fails on `stacked-sheet-close` click | Sheet already closed via `closeAll()` after success | Don't click close — just verify `not.toBeVisible()` |

---
