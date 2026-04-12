# Employee — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Manage positions [MVP]

**Trigger:** Owner navigates to "Cargos" page
**Actor:** Owner
**Domain:** Employee

**Happy path:**
1. Owner opens "Cargos" page → system lists positions (sorted by name)
2. Owner clicks "Novo Cargo" → form opens (name field only)
3. Owner fills name → submits
4. System creates position → success toast → list refreshes
5. Owner can edit a position name inline or via detail → same uniqueness check
6. Owner can delete a position only when no employees are linked (button disabled otherwise)

**Error cases:**
- Name already exists (accent-insensitive) → inline error: "Cargo com este nome já existe"
- Delete position with linked employees → button disabled in UI; if forced: "Cargo possui funcionários vinculados"

---

## Create employee [MVP]

**Trigger:** User clicks "Novo Funcionário" on the employees page
**Actor:** Owner, Manager
**Domain:** Employee

**Happy path:**
1. User clicks "Novo Funcionário" → form opens
2. User fills: name, CPF, phone (optional), hire date, position (dropdown), user (optional dropdown) → submits
3. System validates CPF (format + check digits + uniqueness) → creates employee with status ACTIVE
4. Success toast → list refreshes
5. Audit log records employee creation

**Error cases:**
- Invalid CPF format → inline error: "CPF inválido"
- CPF already exists (active or inactive) → inline error: "CPF já cadastrado"
- Position not found → should not happen (dropdown), but returns error if forced
- User already linked to another employee → inline error: "Usuário já vinculado a outro funcionário"
- Missing required fields → inline validation

---

## Edit employee [MVP]

**Trigger:** User opens employee detail and clicks edit
**Actor:** Owner, Manager
**Domain:** Employee

**Happy path:**
1. User opens employee detail page → clicks edit
2. User modifies fields → submits
3. System validates (same rules as create, excluding self for uniqueness checks)
4. Success toast → detail refreshes
5. Audit log records update

**Error cases:**
- Same as create (CPF uniqueness excludes current employee, userId uniqueness excludes current employee)

---

## Toggle employee status [MVP]

**Trigger:** User toggles employee active/inactive
**Actor:** Owner, Manager
**Domain:** Employee

**Happy path:**
1. User clicks status toggle on employee detail or list row
2. System toggles ACTIVE ↔ INACTIVE
3. Success toast → UI reflects new status
4. Audit log records status change

**Error cases:**
- Employee already in target status → toast: "Funcionário já está ativo/inativo"

---

## Delete employee [MVP]

**Trigger:** Owner clicks delete on employee detail
**Actor:** Owner only
**Domain:** Employee

**Happy path:**
1. Owner clicks delete → confirmation dialog
2. Owner confirms → system checks for cross-domain references (FieldTickets)
3. No references → hard delete → success toast → redirect to list
4. Audit log records deletion

**Error cases:**
- Employee has FieldTicket references → error: "Funcionário possui registros vinculados. Utilize a inativação."
- Non-owner user → 403 Forbidden

---

## View employee audit logs [MVP]

**Trigger:** User clicks "Histórico" on employee or position detail page
**Actor:** Owner, Manager
**Domain:** Employee, Audit

**Happy path:**
1. User opens employee (or position) detail page
2. User clicks "Histórico" → system lists audit log entries (paginated, newest first)
3. Each entry shows: action, actor, timestamp, changes

**Supported entities:**
- Employees → `GET /v1/employees/:id/audit-logs`
- Positions → `GET /v1/positions/:id/audit-logs`

---
