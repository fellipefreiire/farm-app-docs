# User — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## Create user [MVP]

**Trigger:** Admin clicks "Novo usuário" on the user management page
**Actor:** Admin
**Domain:** User

**Happy path:**
1. Admin clicks "Novo usuário" → form opens
2. Admin fills: name, email, password, role (ADMIN or USER) → submits
3. System creates user → success toast → list refreshes
4. Audit log records user creation

**Error cases:**
- Email already exists → inline error: "Email já cadastrado"
- Missing required fields → inline validation
- Non-admin user → 403 Forbidden

---

## List users [MVP]

**Trigger:** Admin navigates to the user management page
**Actor:** Admin
**Domain:** User

**Happy path:**
1. Admin opens "Usuários" page → system lists users (paginated)
2. Each row shows: name, email, role, status (active/inactive)
3. Admin can filter by status or search by name/email

---

## Edit user [MVP]

**Trigger:** Admin opens a user detail and clicks edit
**Actor:** Admin
**Domain:** User

**Happy path:**
1. Admin opens user detail page → clicks edit
2. Admin modifies: name, email, role → submits
3. System validates email uniqueness (excluding current user)
4. Success toast → detail refreshes
5. Audit log records update

**Error cases:**
- Email already exists → inline error: "Email já cadastrado"
- Missing required fields → inline validation

---

## Toggle user status [MVP]

**Trigger:** Admin toggles user active/inactive
**Actor:** Admin
**Domain:** User

**Happy path:**
1. Admin clicks status toggle on user detail or list
2. System toggles active ↔ inactive
3. Success toast → UI reflects new status
4. Audit log records status change

**Notes:**
- Inactive users cannot authenticate
- Soft-deleted users are not shown in listings

---

## Delete user [MVP]

**Trigger:** Admin clicks delete on user detail
**Actor:** Admin
**Domain:** User

**Happy path:**
1. Admin clicks delete → confirmation dialog
2. Admin confirms → system soft deletes user (sets deletedAt)
3. Success toast → redirect to user list
4. Audit log records deletion

**Notes:**
- Always soft delete — user data preserved for audit trail
- Soft-deleted users cannot authenticate or appear in listings
