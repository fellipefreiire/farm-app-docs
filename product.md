# Product

High-level description of the product. This is the first document Claude reads — it shapes every technical decision, prioritization, and conversation.

**Why this matters:** Without product context, Claude makes generic decisions. With it, Claude understands what matters, what can wait, and why things are built a certain way.

---

## How to fill this document

Answer each section below. Be concise — 2-3 sentences per topic is enough. Claude will ask follow-up questions if more detail is needed.

If using `/project-setup`, Claude will guide you through these questions interactively.

---

## How Claude uses this document

- **During Phase 0 (Refinement):** Read before every feature to understand scope, priorities, and what matters to the product.
- **During planning:** Reference MVP features and constraints to size implementation correctly.
- **During code review:** Verify that implementation aligns with product goals and user roles.

---

## Product overview

_What is this product? What problem does it solve? Who is it for?_

<!-- Example: A farm management platform that helps rural producers track livestock, manage pastures, and monitor financial performance. Built for small to mid-size farms in Brazil. -->

---

## Target users

_Who are the main users? What are their roles and goals?_

<!-- Example:
- **Farm owner** — monitors overall performance, makes financial decisions
- **Farm manager** — handles day-to-day operations, registers animals and movements
- **Veterinarian** — records health events, manages vaccination schedules
-->

---

## Core domains

_What are the main business domains? What does each one do?_

<!-- Example:
- **Livestock** — animal registry, breed management, weight tracking
- **Pasture** — paddock management, rotation schedules, grazing capacity
- **Financial** — revenue, expenses, cost per animal, profitability reports
-->

---

## MVP

_Does this project have an MVP (Minimum Viable Product)?_

- **Yes / No**

_If yes, list the features that are part of the MVP. Features outside the MVP are not priorities and should only be considered after the MVP is complete._

<!-- Example:
**MVP features:**
- User authentication (sign-in, sign-up, password recovery)
- Animal registry (create, edit, list, archive)
- Pasture management (create paddocks, assign animals)
- Basic financial dashboard (revenue vs expenses)

**Post-MVP (not priority):**
- Vaccination schedule automation
- Multi-farm support
- Export reports to PDF
- Mobile app
-->

_If no MVP is defined, the entire project scope is treated as the deliverable — all features have equal priority._

---

## Key features

_What are the main features of the product? (full scope, not just MVP)_

<!-- Example:
- Authentication and authorization (role-based access)
- Animal lifecycle management (birth to sale)
- Pasture rotation with capacity alerts
- Financial tracking with cost-per-head metrics
- Audit log for all operations
- Dashboard with KPIs
-->

---

## Tech decisions

_Any product-level tech decisions that affect implementation?_

<!-- Example:
- Multi-tenant: each farm is a tenant with isolated data
- Offline-first is NOT required — always connected
- Portuguese UI, English codebase
- Currency: BRL only (no multi-currency)
-->

---

## Constraints

_Technical or business constraints that limit what can be built or how._

<!-- Example:
- Must run on shared hosting (no Docker in production)
- Max 2 developers — keep architecture simple
- Budget: no paid external services in MVP
- Must support IE11 (or: modern browsers only)
-->

---

## Out of scope

_What this project explicitly does NOT include. Helps prevent scope creep and keeps Claude focused._

<!-- Example:
- Mobile app — web only
- Multi-language support — Portuguese only in v1
- Real-time collaboration — single user per session
- Public API for third parties
-->

---

## Success metrics

_How do you measure if the product is working? Helps Claude understand what matters most._

<!-- Example:
- User can register an animal in under 2 minutes
- Dashboard loads in under 3 seconds
- Zero data loss on animal transfer operations
- 95% uptime
-->
