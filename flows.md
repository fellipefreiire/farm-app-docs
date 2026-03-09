# Flows

User flows describe how features work step by step — what the user does, what the system does, and which technical pieces are involved. This document connects product requirements to implementation.

**Why this matters:** Without documented flows, Claude implements features based on assumptions. With them, Claude knows the exact sequence of interactions, which screens are involved, and how errors are handled.

---

## How to fill this document

Each flow follows this structure:

```markdown
### Flow name

**Trigger:** What starts this flow (e.g. user clicks "Create", page loads, cron job runs)
**Actor:** Who performs this flow (e.g. Farm owner, Admin, System)
**Domain:** Which domain this flow belongs to

**Happy path:**
1. User does X → sees Y
2. User fills form → submits
3. System calls `create<Entity>Action` → backend validates → returns success
4. Toast: "<Entity> created successfully" → redirects to list page

**Error cases:**
- Validation fails → form shows inline errors (Zod)
- API returns 409 (conflict) → toast: "Entity already exists"
- API returns 500 → toast: DEFAULT_ERROR_MESSAGE

**Technical references:**
- Page: `src/app/(private)/dashboard/<domain>/page.tsx`
- Action: `src/domains/<domain>/actions/create-<entity>-action.ts`
- Endpoint: `POST /v1/<entities>`
```

**Guidelines:**
- One flow per user action or feature — not one per page
- Keep steps concise — describe what happens, not how the code works
- Always include error cases — they define UX decisions
- Technical references link to the actual files — helps Claude navigate the codebase
- Group flows by domain — matches the project structure
- Mark MVP flows explicitly if the project has an MVP defined in `product.md` — add `[MVP]` after the flow name: `### Create Order [MVP]`

---

## How Claude uses this document

- **During implementation:** Read the relevant flow before writing code. The flow defines the expected behavior — do not deviate without asking.
- **During planning:** Reference flows in the plan to show which steps are being implemented.
- **When flows are missing:** If implementing a feature that has no documented flow, ask the developer to describe the expected behavior before coding. Add the flow to this document after alignment.
- **After implementation:** Update this document in Phase 4 if the flow changed during development.

---

<!-- Add flows below, grouped by domain. Remove this comment when the first flow is added. -->

<!-- Example:

## Orders

### Create order

**Trigger:** User clicks "New order" button on the orders list page
**Actor:** Authenticated user
**Domain:** Orders

**Happy path:**
1. User clicks "New order" → sheet opens from the right
2. User fills: customer (select), items, notes → clicks "Create order"
3. `toast.loading("Creating order...")` → calls `createOrderAction`
4. Action validates with Zod → calls `POST /v1/orders` → `revalidateTag('orders')`
5. `toast.success("Order created successfully")` → sheet closes → list refreshes

**Error cases:**
- Customer not selected → inline validation: "Customer is required"
- No items added → inline validation: "At least one item is required"
- API returns 409 → toast: "Duplicate order reference"
- API returns 500 → toast: DEFAULT_ERROR_MESSAGE

**Technical references:**
- Page: `src/app/(private)/dashboard/orders/page.tsx`
- Form: `src/domains/orders/components/forms/create-order-form.tsx`
- Action: `src/domains/orders/actions/create-order-action.ts`
- Endpoint: `POST /v1/orders`

---

### Cancel order

**Trigger:** User clicks "Cancel" in the actions popover on the orders list
**Actor:** Authenticated user (owner or admin)
**Domain:** Orders

**Happy path:**
1. User clicks actions (⋯) on order row → popover opens
2. User clicks "Cancel" → confirmation dialog opens
3. User clicks "Confirm" in dialog → `toast.loading("Cancelling order...")`
4. Calls `cancelOrderAction` → `PATCH /v1/orders/:id/cancel`
5. `toast.success("Order cancelled successfully")` → dialog closes → list refreshes

**Error cases:**
- API returns 404 → toast: "Order not found"
- API returns 422 → toast: "Order cannot be cancelled in current status"
- API returns 500 → toast: DEFAULT_ERROR_MESSAGE

**Technical references:**
- Dialog: `src/domains/orders/components/dialogs/cancel-order-dialog.tsx`
- Form: `src/domains/orders/components/forms/cancel-order-form.tsx`
- Action: `src/domains/orders/actions/cancel-order-action.ts`
- Endpoint: `PATCH /v1/orders/:id/cancel`

-->
