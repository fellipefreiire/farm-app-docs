---
name: flow-discovery
description: Guides a structured conversation to define user flows and system interactions. Output is added to docs/flows.md.
---

# Flow Discovery

Facilitates a structured conversation to define user flows end-to-end. The output is appended to `docs/flows.md` and serves as reference for frontend implementation and E2E test scenarios.

---

## Process

Ask questions **one at a time**. Focus on one flow per session.

### 1. Flow Identification

- What is the name of this flow? (e.g., "User Registration", "Place Order", "Approve Budget")
- Who initiates it? (which user role)
- What is the starting state? (e.g., "user is on the login page", "order is in draft")
- What is the end state? (e.g., "user is authenticated", "order is confirmed and payment initiated")

### 2. Happy Path Steps

Walk through the flow step by step:
- What does the user do?
- What does the frontend do in response?
- What request is sent to the backend?
- What does the backend validate and process?
- What is stored or changed in the database?
- What is returned to the frontend?
- What does the user see?

Repeat for each step until the end state is reached.

### 3. Alternative Paths

- What happens if validation fails? (frontend and backend)
- What happens if an external service is unavailable?
- What happens if the user is unauthorized?
- Are there optional branches? (e.g., "if the user already has an address, skip that step")

### 4. Domain Events

- Does this flow emit domain events that trigger other flows?
- If yes, describe the downstream effect

---

## Output

Write the flow to `docs/flows.md`:

```markdown
## <Flow Name>

**Actor:** <user role>
**Start:** <starting state>
**End:** <end state>

### Steps

1. User does X
2. Frontend validates locally and sends POST /endpoint with { field: value }
3. Backend validates input (Zod), checks authorization
4. Use case enforces business rules
5. Repository persists the result
6. Domain event <EventName> is emitted
7. Backend returns { id, status }
8. Frontend updates UI / redirects to /path

### Alternative Paths

- **Validation error:** frontend shows inline error, no request sent
- **Business rule violation:** backend returns 422 with typed error code, frontend shows message
- **Unauthorized:** backend returns 401, frontend redirects to login

### Downstream Effects

- <EventName> triggers <OtherFlow> in <Domain>

---

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
```

---

## Completion

Present the flow to the user and ask:
1. Is the happy path correct?
2. Are there missing alternative paths?
3. Are there missing downstream effects?

If confirmed → inform the user that `docs/flows.md` has been updated.
Ask if there are more flows to define.
