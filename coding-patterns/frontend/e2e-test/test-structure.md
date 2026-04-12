# E2E — Test structure

> Part of the farm-app frontend E2E test pattern collection. Read `_index.md` first for shared rules and anti-patterns.

## Unauthenticated flow

```ts
import { expect, test } from '@playwright/test'

test.describe('Sign In', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/sign-in')
  })

  test('should sign in with valid credentials', async ({ page }) => {
    await page.getByTestId('auth-email-input').fill('test@example.com')
    await page.getByTestId('auth-password-input').fill('password123')
    await page.getByTestId('auth-sign-in-button').click()

    await page.waitForURL('/dashboard')
    await expect(page.getByTestId('dashboard-page')).toBeVisible()
  })
})
```

## Authenticated CRUD flow

```ts
import { test, expect } from '../fixtures/auth.fixture'

test.describe.serial('Manage Entities', () => {
  test('should list entities', async ({ page }) => {
    await page.goto('/entities')
    await expect(page.getByTestId('entities-page')).toBeVisible()
    await expect(page.getByText('Seed Entity')).toBeVisible()
  })

  test('should create an entity', async ({ page }) => {
    await page.goto('/entities')
    await page.getByTestId('entity-create-button').click()
    await expect(page.getByTestId('stacked-sheet')).toBeVisible()

    await page.getByTestId('entity-name-input').fill('Nova Entidade')
    await page.getByTestId('entity-create-submit').click()

    // Toast messages are always in Portuguese (set in server actions)
    await expect(page.getByText('Entidade criada com sucesso.')).toBeVisible()
  })

  test('should edit an entity', async ({ page }) => {
    await page.goto('/entities')
    // Use regex + first() when exact ID is unknown
    const actionsButton = page.getByTestId(/^entity-actions-/).first()
    await actionsButton.click()

    const editButton = page.getByTestId(/^entity-edit-btn-/).first()
    await editButton.click()

    await expect(page.getByTestId('entity-edit-form')).toBeVisible()
    // ... fill and submit ...

    await expect(page.getByText('Entidade atualizada com sucesso.')).toBeVisible()

    // Edit forms call closeAll() on success — sheet closes automatically.
    // Do NOT click stacked-sheet-close — just verify it's gone.
    await expect(page.getByTestId('stacked-sheet')).not.toBeVisible()
  })

  test('should delete an entity', async ({ page }) => {
    await page.goto('/entities')
    const actionsButton = page.getByTestId(/^entity-actions-/).first()
    await actionsButton.click()

    const deleteButton = page.getByTestId(/^entity-delete-btn-/).first()
    await deleteButton.click()

    await expect(page.getByTestId('entity-delete-dialog')).toBeVisible()
    await page.getByTestId('entity-delete-confirm').click()
    await expect(page.getByTestId('entity-delete-dialog')).not.toBeVisible()
  })
})
```

---
