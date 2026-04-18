# Frontend Component Pattern — Index

Navigation hub for farm-app frontend component patterns. Read this file first,
then read ONLY the specific variant file you need. **Do not load the whole tree**.

## Files

| Variant | File | When to read |
|---|---|---|
| Listing filters | [listing-filters.md](listing-filters.md) | Building filter UI for a list page |
| Table | [table.md](table.md) | Building a data table (headers, rows, sorting, selection) |
| Forms | [forms.md](forms.md) | Building create/edit forms (with Zod + React Hook Form) |
| Dialogs | [dialogs.md](dialogs.md) | Building modal/alert dialogs |
| UI primitives | [ui-components.md](ui-components.md) | Design-system primitives wrapped for the app |
| Dynamic item lists in sheets | [dynamic-item-lists.md](dynamic-item-lists.md) | Forms with add/remove row items inside a sheet |

## File locations

```
src/domains/<domain>/components/
  <group>/                            ← subdirectory per category (see grouping rules below)
    <entity>-<name>.tsx
  index.ts                            ← barrel export
```

> **Multi-entity domains:** When the domain uses subdomain folders (see `domain-organization.md`), components move under the subdomain: `src/domains/<domain>/<subdomain>/components/`.

### Grouping rules

Components are grouped into subdirectories when **two or more components share the same category**. Groups are not fixed — they emerge from the domain's needs.

Common groups:

```
table/      ← columns + table component
forms/      ← create, edit, confirmation forms
dialogs/    ← dialog wrappers (archive, delete, cancel)
sheets/     ← sheet wrappers (create, edit)
ui/         ← small domain-specific UI (badges, popovers, cards)
```

---

## data-testid convention

All interactive elements must include `data-testid` for Playwright tests. Never use CSS selectors or text content in tests.

**Naming format:** `<entity>-<component>-<element>`

```tsx
// Buttons
<Button data-testid="<entity>-create-submit">Criar <entity></Button>
<Button data-testid="<entity>-edit-submit">Salvar alterações</Button>
<Button data-testid="<entity>-delete-confirm">Excluir <Entity></Button>
<Button data-testid="<entity>-delete-cancel">Voltar</Button>

// Form fields
<InputField data-testid="<entity>-name-input" ... />
<SelectField data-testid="<entity>-status-select" ... />

// Actions
<Button data-testid={`<entity>-actions-${entity.id}`} ... />
<Button data-testid={`<entity>-edit-btn-${entity.id}`} ... />
<Button data-testid={`<entity>-delete-btn-${entity.id}`} ... />

// Dialogs
<DialogContent data-testid="<entity>-delete-dialog" ... />

// Pages
<div data-testid="<entity>s-page" ... />
<div data-testid="<entity>-detail-page" ... />
```

---

## Barrel export

```ts
// src/domains/<domain>/components/index.ts
export * from './table/<entity>-columns'
export * from './table/<entity>-table'
export * from './forms/create-<entity>-form'
export * from './forms/edit-<entity>-form'
export * from './forms/delete-<entity>-form'
export * from './dialogs/delete-<entity>-dialog'
export * from './sheets/create-<entity>-sheet'
export * from './sheets/edit-<entity>-sheet'
export * from './ui/<entity>-actions-popover'
```

---

## Rules

- Components are grouped into subdirectories by category — groups emerge from the domain's needs
- `'use client'` directive on all components that use hooks, state, or event handlers
- Every interactive element must have a `data-testid` attribute
- Every component must have a one-line JSDoc comment describing what it does
- Forms use `react-hook-form` + `zodResolver` — validation is always schema-driven
- Forms show `toast.loading` before the action, `renderToast` after — never silent mutations
- Toast messages in Portuguese: "Criando...", "Excluindo...", "Salvando..."
- Dialogs are thin wrappers — they manage open/close via the store and render a form inside
- Destructive actions always require a confirmation dialog with `variant="destructive"` button
- Destructive option text uses `text-destructive` in popovers
- After deletion, redirect to the listing page
- Columns are defined in a separate file from the table component
- Every `components/` directory must have an `index.ts` barrel export

---

## Anti-patterns

```tsx
// ❌ missing data-testid on interactive element
<Button type="submit">Save</Button>
// ✅ <Button type="submit" data-testid="<entity>-edit-submit">Salvar alterações</Button>

// ❌ destructive action without confirmation dialog
onClick={() => deleteEntity(id)}
// ✅ onClick={() => setDeleteDialogOpen(id)}

// ❌ destructive option without red text
<Button variant="ghost">Excluir</Button>
// ✅ <Button variant="ghost" className="text-destructive">Excluir <entity></Button>

// ❌ English text in UI
<Button>Delete</Button>
// ✅ <Button>Excluir Variedade</Button>

// ❌ no redirect after deletion
if (res.ok) { closeDialog() }
// ✅ if (res.ok) { closeDialog(); router.push('/<entities>') }

// ❌ no toast feedback on mutation
await createAction(data)
// ✅ const toastId = toast.loading('Criando...'); ... renderToast(...)

// ❌ input type="number" for numeric fields
<InputField type="number" name="year" ... />
// ✅ text input with mask, inputMode, and numeric prop
<InputField name="year" inputMode="numeric" maxLength={4} mask={maskDigitsOnly} numeric ... />

// ❌ input type="number" for decimal fields
<InputField type="number" name="capacity" ... />
// ✅ text input with decimal mask
<InputField name="capacity" inputMode="decimal" mask={maskPositiveFloat} numeric ... />
```
