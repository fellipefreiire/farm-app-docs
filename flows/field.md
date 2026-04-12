# Field — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Create field [MVP]

**Trigger:** User clicks "Novo talhão" on the fields list page
**Actor:** Farm owner, Farm manager
**Domain:** Field

**Happy path:**
1. User clicks "Novo talhão" → form opens
2. User fills: name, area (hectares), location/description → submits
3. System creates field → success toast → list refreshes

**Error cases:**
- Duplicate name → inline error: "Talhão já cadastrado com esse nome"
- Missing required fields → inline validation

---

## Manage field [MVP]

**Trigger:** User interacts with an existing field (edit, archive, delete)
**Actor:** Farm owner, Farm manager
**Domain:** Field

**Edit:**
1. User clicks field in the list → detail page opens
2. User edits name, area, or description → submits
3. System updates field → success toast → audit log records change

**Toggle status (active/inactive):**
1. User clicks toggle on a field → confirmation dialog
2. System toggles status → list updates → audit log records change

**Delete:**
1. User clicks delete on a field → confirmation dialog
2. System soft-deletes field → success toast → list refreshes → audit log records deletion

**Error cases:**
- Duplicate name on edit → inline error: "Talhão já cadastrado com esse nome"
- Field not found → 404

---
