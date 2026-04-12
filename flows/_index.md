# Flows — Index

Per-domain catalog of user and system flows. Each file corresponds to one
farm-app domain and lists all the flows belonging to it. Mirrors the
`rules/<domain>.md` structure — a subagent working on a domain reads both.

## Files

| Domain | File | Flows |
|---|---|---|
| Auth | [auth.md](auth.md) | Sign in, Sign up, Password recovery, Token refresh |
| User | [user.md](user.md) | Create user, List users, Edit user, Toggle user status, Delete user |
| Field | [field.md](field.md) | Create field, Manage field |
| Crop | [crop.md](crop.md) | Manage crop types, Manage varieties, Harvest lifecycle |
| Audit | [audit.md](audit.md) | View audit logs |
| Inventory | [inventory.md](inventory.md) | Manage categories, Manage inputs (insumos), Register purchase (entrada), Register stock movement (saída) |
| Schedule | [schedule.md](schedule.md) | Schedule auto-created with Harvest, Edit schedule, Copy schedule (wizard), Review completed schedule, Cancel schedule with PRINTED resolution |
| Fleet | [fleet.md](fleet.md) | Manage vehicles, Manage implements |
| Employee | [employee.md](employee.md) | Manage positions, Create employee, Edit employee, Toggle employee status, Delete employee, View employee audit logs |
| Supplier | [supplier.md](supplier.md) | Manage suppliers, Toggle supplier status, Delete supplier, View supplier audit logs |
| FieldTicket | [field-ticket.md](field-ticket.md) | Add operation to schedule, Review field ticket, Print field ticket, Finalize field ticket, Re-evaluate field ticket |

## How to add a new flow

Each flow follows this format:

```markdown
### Flow name [MVP]

**Trigger:** What initiates the flow
**Actor:** Who performs the action
**Domain:** Which domain(s) are involved

**Happy path:**
1. Step-by-step description of the normal flow

**Error cases:**
- Error condition → expected behavior (toast, inline error, redirect)
```

**Guidelines:**
- Group flows by domain
- Include both happy path and error cases
- Error messages should be in Portuguese (frontend i18n responsibility)
- Use `[MVP]` tag for features in the minimum viable product
- Update this document in Phase 4 whenever user flows are added or changed

---
