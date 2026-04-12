# E2E — Common patterns

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## Waiting for search (debounced input)

The search input has a 300ms debounce. Wait for the filtered result instead of using `waitForTimeout`:

```ts
await searchInput.fill('North')

// Wait for a non-matching item to disappear (confirms filter applied)
await expect(page.getByText('South Field')).not.toBeVisible({ timeout: 10000 })

// Then assert the matching item is visible
await expect(page.getByText('North Field')).toBeVisible()
```

## After edit/create submit

Edit and create forms call `closeAll()` on success — the sheet closes automatically. Do **not** click `stacked-sheet-close` — just verify the sheet is gone:

```ts
await page.getByTestId('entity-edit-submit').click()
await expect(page.getByText('Entidade atualizada com sucesso.')).toBeVisible()

// Sheet closes automatically via closeAll() — just verify it's gone
await expect(page.getByTestId('stacked-sheet')).not.toBeVisible()
```

## Avoiding strict mode violations

When asserting text that appears in both a table cell and a toast, use `getByRole` to be specific:

```ts
// ❌ Matches both the cell "Fornecedor Atualizado" and toast "Fornecedor atualizado com sucesso."
await expect(page.getByText('Fornecedor Atualizado')).toBeVisible()

// ✅ Targets only the table cell
await expect(page.getByRole('cell', { name: 'Fornecedor Atualizado' })).toBeVisible()
```

## Dynamic entity selection (when ID is unknown)

When entities were created or modified by previous tests:

```ts
// Use regex pattern + first()/last()
const actionsButton = page.getByTestId(/^entity-actions-/).first()
await actionsButton.click()

// For status cells — use Portuguese labels as shown in the UI
const plannedCell = page
  .locator('[data-testid^="harvest-status-"]')
  .filter({ hasText: 'Planejada' })
  .first()
const testId = await plannedCell.getAttribute('data-testid')
const harvestId = testId!.replace('harvest-status-', '')
```

## Protecting seed data from other test files

When a delete test runs, avoid deleting entities used by other test files:

```ts
// ❌ Deletes the first item — might be a seed entity used elsewhere
const first = page.getByTestId(/^crop-type-actions-/).first()

// ✅ Deletes the last item — likely a non-seed entity created by a previous test
const last = page.getByTestId(/^crop-type-actions-/).last()
```

---
