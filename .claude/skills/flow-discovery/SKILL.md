---
name: flow-discovery
description: Structured conversation to capture a user/system flow end-to-end. Output is appended to docs/flows/<domain>.md (per-domain file).
---

# Flow Discovery

Captures a farm-app flow end-to-end. Output goes into `docs/flows/<domain>.md` (per-domain file) and
serves as reference for frontend implementation and E2E test scenarios.

One flow per session. Ask questions **one at a time**.

## Process

### 1. Flow Identification
- Flow name (e.g. "User Registration", "Place Order", "Approve Budget")
- Actor (user role that initiates it)
- Starting state
- End state

### 2. Happy Path
Walk through each step:
- User action
- Frontend response (local validation, optimistic UI)
- Backend request (method, path, body shape)
- Backend validation + processing (use case, business rules)
- Database effect (what persists or changes)
- Response to frontend
- User-visible result

Repeat until end state.

### 3. Alternative Paths
- Validation failure (frontend + backend)
- External service unavailable
- Unauthorized
- Optional branches (e.g. "if user already has address, skip that step")

### 4. Downstream Effects
- Domain events emitted that trigger other flows
- Downstream flow impact

## Output

Append to `docs/flows/<domain>.md` (where `<domain>` is the domain the flow belongs to — create a new file if none exists, using the template in `docs/flows/_index.md`):

~~~markdown
## <Flow Name>

**Actor:** <user role>
**Start:** <starting state>
**End:** <end state>

### Steps
1. User does X
2. Frontend validates locally, sends POST /endpoint with { field: value }
3. Backend validates input (Zod), checks authorization
4. Use case enforces business rules
5. Repository persists
6. Domain event <EventName> emitted
7. Backend returns { id, status }
8. Frontend updates UI / redirects to /path

### Alternative Paths
- **Validation error:** frontend shows inline error, no request sent
- **Business rule violation:** backend returns 422, typed error code, frontend shows message
- **Unauthorized:** backend returns 401, frontend redirects to login

### Downstream Effects
- <EventName> triggers <OtherFlow> in <Domain>

\`\`\`mermaid
sequenceDiagram
  actor User
  participant Frontend
  participant Backend
  participant DB
  User->>Frontend: action
  Frontend->>Backend: POST /endpoint
  Backend->>DB: query/persist
  DB-->>Backend: result
  Backend-->>Frontend: response
  Frontend-->>User: feedback
\`\`\`
~~~

## Completion

Present the flow to the user. Ask:
1. Is the happy path correct?
2. Missing alternative paths?
3. Missing downstream effects?

If confirmed, inform which `docs/flows/<domain>.md` file was updated. Ask if there are more flows to define.
