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

<!-- Add endpoints below, grouped by domain. Remove this comment when the first endpoint is added. -->

<!-- Example:

## Authentication

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/auth/sign-in | Authenticate user with email and password | Public |
| POST | /v1/auth/sign-up | Register new user | Public |
| POST | /v1/auth/refresh | Refresh access token | Public |
| POST | /v1/auth/forgot-password | Send password recovery email | Public |
| POST | /v1/auth/reset-password | Reset password with token | Public |

## Livestock

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | /v1/animals | Create animal | Private |
| GET | /v1/animals | List animals (page, perPage, search, active, breed) | Private |
| GET | /v1/animals/:id | Find animal by ID | Private |
| PUT | /v1/animals/:id | Edit animal | Private |
| DELETE | /v1/animals/:id | Delete animal | Private |
| PATCH | /v1/animals/:id/toggle-status | Toggle animal active/archived | Private |

-->
