# Product

**Why this matters:** This is the foundational document that defines the product scope, target users, and core domains. All other docs (architecture, flows, rules) derive from the decisions made here. Claude uses this to understand the business context behind every implementation request.

## Product overview

Farm management platform for crop producers. Helps farmers plan field operations, track input usage, manage inventory, and maintain full auditability of all activities. Built as a SaaS — single-tenant for MVP, designed to not block future multi-tenant migration.

## Target users

- **Farm owner** — creates schedules, reviews and prints field tickets, manages fields and crops, oversees financials. May delegate access to family members (children with their own logins).
- **Farm manager** — handles day-to-day operations, registers schedule execution, finalizes field tickets, manages inventory.

> The tractor operator does NOT have system access — they receive printed field tickets only.

## Core domains

- **Auth** — authentication, authorization, role-based access, user management
- **Field (Talhao)** — field registry, area, planted crop/variety
- **Crop (Safra)** — crop cycles, planting periods, harvest tracking
- **Schedule (Cronograma)** — per-field operation planning: which operations (spraying, fertigation), which inputs, on which days
- **FieldTicket** — generated from schedule, pre-populated with inputs per field per day. Review → print → execute → return → finalize workflow
- **Inventory (Estoque)** — inputs, seeds, harvested products, stock control
- **Supplier (Fornecedor)** — supplier registry, input purchase tracking for future price comparison
- **Financial** — revenue, expenses, cost per crop/field
- **Audit** — full audit trail of every user action in the system

## MVP

**Yes — MVP is defined.**

**MVP features:**
- Authentication (sign-in, sign-up, password recovery)
- Field management (create, edit, list fields with area and crop info)
- Crop management (crop cycles, varieties, planting/harvest periods)
- Inventory management (inputs, seeds, stock in/out)
- Schedule system (core of the MVP):
  - Create per-field schedule with crop, variety, time period
  - Define operations per day (spraying, fertigation, etc.)
  - Assign inputs to each operation per day
  - Edit schedule over time (changes reflect in field tickets)
- FieldTicket workflow:
  - Auto-generate field tickets from schedule (pre-populated)
  - Review field tickets before printing (edit inputs if needed)
  - Print field tickets for tractor operators
  - Register field ticket completion (same day or next day)
  - Validate input usage before finalizing (handle last-minute changes)
  - Finalize field ticket with confirmation
  - Re-evaluate finalized field tickets if registered incorrectly
- Full audit trail on every action

**Post-MVP (not priority):**
- Dashboard with KPIs (needs full app running first to decide what to show)
- Financial module (revenue, expenses, cost per crop)
- Supplier management (registry, price comparison)
- Packaging management
- Employee management
- Multi-tenant SaaS
- Offline mode with data sync
- Mobile app

## Tech decisions

- Single-tenant for MVP — but no design decisions that block multi-tenant migration
- No offline mode in MVP — planned for future (some clients have poor connectivity)
- Portuguese UI, English codebase
- Currency: BRL only
- Tractor operators receive printed field tickets only — no system access for them
- Every user action must be audited (audit log is a first-class requirement)

## Constraints

- Low concurrent users expected (single farm in MVP)
- Infrastructure decisions deferred to post-MVP
- No paid external services in MVP unless strictly necessary
- Must support future offline-first with sync capability — avoid architecture that makes this impossible

## Out of scope

- Native mobile app
- Multi-language support
- Multi-currency
- Public API for third parties
- IoT / machinery integration
- Insumo marketplace
- Multi-tenant infrastructure (MVP is single-tenant)

## Success metrics

- Zero data loss on field ticket operations (create, finalize, re-evaluate)
- Full auditability — every user action traceable
- FieldTicket generation from schedule is fast (data is pre-populated)
- Schedule creation is naturally slower (one per field, detailed planning) — no speed target
- System remains simple enough for non-technical farm owners
