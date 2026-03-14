# API Reference

Quick reference of all available endpoints. This document gives Claude a fast overview of what exists without reading every controller. For full request/response details, check the Swagger documentation.

**Why this matters:** When implementing a new feature, Claude needs to know which endpoints already exist to avoid duplicating work or creating inconsistent routes. This document is the single source of truth for "what endpoints does the API have?".

---

## How to fill this document

Add endpoints as they are implemented. Each domain has a table with:

| Column | Description |
|--------|-------------|
| Method | HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) |
| Path | Full versioned path (e.g. `/v1/animals`) |
| Description | What it does (1 short sentence) |
| Auth | `Public` or `Private` (private = JWT required, which is the default) |

```markdown
## Domain name

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/entities | Create entity | Private |
| GET | /v1/entities | List entities (paginated) | Private |
| GET | /v1/entities/:id | Find entity by ID | Private |
| PUT | /v1/entities/:id | Edit entity | Private |
| DELETE | /v1/entities/:id | Delete entity | Private |
| PATCH | /v1/entities/:id/toggle-status | Toggle active/archived | Private |
```

**Guidelines:**
- Group by domain — one section per domain, matching `product.md` core domains. For shared/cross-domain endpoints (Auth, Health, Notifications), use a "Shared" or "Platform" section
- Keep descriptions short — the detail lives in Swagger
- Always include auth — makes it clear which routes need `@Public()`
- Add query params inline when relevant (e.g. `GET /v1/entities?page=1&search=foo`)
- Update this document in Phase 4 whenever new endpoints are added

---

## How Claude uses this document

- **Before creating a new endpoint:** Check if a similar one already exists to avoid duplication
- **During frontend implementation:** Reference this to know which endpoints to call
- **After implementation:** Add new endpoints in Phase 4

---

## Real-time endpoints (optional)

If the project uses WebSockets or Server-Sent Events, document them in a separate section per domain:

```markdown
## Domain name — Real-time

### WebSocket

| Event | Direction | Description | Auth |
|-------|-----------|-------------|------|
| connection | client → server | Connect to /ws/domain | Private (JWT in query param) |
| entity:created | server → client | Notifies when entity is created | — |
| entity:updated | server → client | Notifies when entity is updated | — |

### SSE (Server-Sent Events)

| Path | Description | Auth |
|------|-------------|------|
| GET /v1/domain/events | Stream of domain events | Private |
```

**Guidelines:**
- Direction: `client → server` or `server → client`
- WebSocket events use the format `entity:action`
- SSE endpoints are standard GET routes that return `text/event-stream`
- Auth for WebSockets is typically via JWT in query parameter (not header)

---

## Platform

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| GET | /health | Application health check | Public |

## User

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/sessions | Authenticate user with email and password | Public |
| POST | /v1/sessions/refresh | Refresh access token | Public |
| POST | /v1/sessions/logout | Logout user (invalidate token) | Private |
| POST | /v1/users | Create a new user | Private |

## Field

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/fields | Create a new field | Private |
| GET | /v1/fields | List fields (paginated, searchable) | Private |
| GET | /v1/fields/:id | Find field by ID | Private |
| PUT | /v1/fields/:id | Edit a field | Private |
| DELETE | /v1/fields/:id | Delete a field | Private |
| PATCH | /v1/fields/:id/toggle-status | Toggle field active/inactive | Private |
| GET | /v1/fields/:id/audit-logs | List audit logs for a field | Private |

## Crop

### Crop Types

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/crop-types | Create a new crop type | Private |
| GET | /v1/crop-types | List crop types (paginated) | Private |
| GET | /v1/crop-types/:id | Find crop type by ID | Private |
| PUT | /v1/crop-types/:id | Edit a crop type | Private |
| DELETE | /v1/crop-types/:id | Delete a crop type | Private |
| GET | /v1/crop-types/:id/audit-logs | List audit logs for a crop type | Private |

### Varieties

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/crop-types/:cropTypeId/varieties | Create a new variety | Private |
| GET | /v1/crop-types/:cropTypeId/varieties | List varieties for a crop type | Private |
| GET | /v1/varieties/:id | Find variety by ID | Private |
| PUT | /v1/varieties/:id | Edit a variety | Private |
| DELETE | /v1/varieties/:id | Delete a variety | Private |
| GET | /v1/varieties/:id/audit-logs | List audit logs for a variety | Private |

### Harvests

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/harvests | Create a new harvest | Private |
| GET | /v1/harvests | List harvests (paginated) | Private |
| GET | /v1/harvests/:id | Find harvest by ID | Private |
| GET | /v1/harvests/active | Get active harvest by field | Private |
| PUT | /v1/harvests/:id | Edit a harvest | Private |
| DELETE | /v1/harvests/:id | Delete a harvest | Private |
| PATCH | /v1/harvests/:id/activate | Activate a harvest | Private |
| PATCH | /v1/harvests/:id/complete | Complete a harvest | Private |
| PATCH | /v1/harvests/:id/cancel | Cancel a harvest | Private |
| GET | /v1/harvests/:id/audit-logs | List audit logs for a harvest | Private |
