# 2026-03-09 — Single-tenant MVP with multi-tenant awareness

**Context:** The app will be a SaaS, but MVP targets a single farm for testing.
**Decision:** Build single-tenant for MVP, but avoid architecture that blocks multi-tenant migration.
**Options considered:** Single-tenant aware (chosen) / Full multi-tenant from day one / Single-tenant with no future consideration
**Rationale:** Reduces MVP complexity while keeping the door open. Avoiding things like global singletons, hardcoded tenant assumptions, or shared mutable state.
**Consequences:** Entity design should not assume a single tenant. When multi-tenant is added, a `tenantId` can be introduced without major refactoring.
