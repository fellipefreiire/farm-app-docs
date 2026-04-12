# Supplier — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Manage suppliers [MVP]

**Trigger:** User navigates to "Fornecedores" page
**Actor:** Owner, Manager
**Domain:** Supplier

**Happy path:**
1. User opens "Fornecedores" page → system lists suppliers (paginated)
2. User clicks "Novo Fornecedor" → form opens
3. User fills: name, phone (optional), email (optional) → submits
4. System validates name uniqueness (accent-insensitive) → creates supplier with status ACTIVE
5. Success toast → list refreshes
6. Audit log records supplier creation

**Edit flow:**
1. User opens supplier detail → clicks edit
2. User modifies fields → submits
3. System validates (name uniqueness excludes current supplier)
4. Success toast → detail refreshes

**Error cases:**
- Name already exists (accent-insensitive, e.g. "Agroquimica" vs "Agroquímica") → inline error: "Fornecedor com este nome já existe"
- Invalid email format → inline validation
- Missing name → inline validation

---

## Toggle supplier status [MVP]

**Trigger:** User toggles supplier active/inactive
**Actor:** Owner, Manager
**Domain:** Supplier

**Happy path:**
1. User clicks status toggle on supplier detail or list
2. System toggles ACTIVE ↔ INACTIVE
3. Success toast → UI reflects new status
4. Audit log records status change

**Notes:**
- Inactive supplier cannot be selected in new Purchase forms
- Inactive supplier still visible in existing Purchase records (historical data preserved)

---

## Delete supplier [MVP]

**Trigger:** User clicks delete on supplier detail
**Actor:** Owner only
**Domain:** Supplier

**Happy path (no references):**
1. Owner clicks delete → confirmation dialog
2. Owner confirms → system checks for Purchase references
3. No purchases reference this supplier → hard delete → success toast → redirect to list

**Happy path (has references):**
1. Owner clicks delete → confirmation dialog
2. Owner confirms → system detects Purchase references → soft delete (sets deletedAt)
3. Success toast → redirect to list
4. Supplier no longer appears in listings but remains visible in existing Purchase records

**Error cases:**
- Non-owner user → 403 Forbidden

---

## View supplier audit logs [MVP]

**Trigger:** User clicks "Histórico" on supplier detail page
**Actor:** Owner, Manager
**Domain:** Supplier, Audit

**Happy path:**
1. User opens supplier detail page
2. User clicks "Histórico" → system lists audit log entries (paginated, newest first)
3. Each entry shows: action, actor, timestamp, changes

**Endpoint:** `GET /v1/suppliers/:id/audit-logs`

---
