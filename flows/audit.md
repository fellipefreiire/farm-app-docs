# Audit — Flows

> Per-domain flow catalog. See `_index.md` for format template and conventions.

## View audit logs [MVP]

**Trigger:** User clicks "Histórico" on any entity detail page
**Actor:** Farm owner, Farm manager, Admin
**Domain:** Cross-domain (all domains with mutations)

**Happy path:**
1. User opens entity detail page (field, crop type, variety, or harvest)
2. User clicks "Histórico" or scrolls to audit section
3. System lists audit log entries (paginated, newest first)
4. Each entry shows: action, actor, timestamp, changes

**Supported entities:**
- Fields → `GET /v1/fields/:id/audit-logs`
- Crop types → `GET /v1/crop-types/:id/audit-logs`
- Varieties → `GET /v1/varieties/:id/audit-logs`
- Harvests → `GET /v1/harvests/:id/audit-logs`
- Employees → `GET /v1/employees/:id/audit-logs`
- Positions → `GET /v1/positions/:id/audit-logs`
- Suppliers → `GET /v1/suppliers/:id/audit-logs`
- Vehicles → `GET /v1/vehicles/:id/audit-logs`
- Implements → `GET /v1/implements/:id/audit-logs`
- Schedules → `GET /v1/schedules/:id/audit-logs`
- Field tickets → `GET /v1/field-tickets/:id/audit-logs`
- Categories → `GET /v1/categories/:id/audit-logs`
- Inputs → `GET /v1/inputs/:id/audit-logs`
- Purchases → `GET /v1/purchases/:id/audit-logs`
- Stock movements → `GET /v1/stock-movements/:id/audit-logs`

---
