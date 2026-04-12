# 2026-03-10 — Subdomain folders for multi-entity domains

**Context:** The Crop domain has 3 entities (CropType, Variety, Harvest) with 200+ files across frontend and backend. Maintaining all files in a single flat folder became unwieldy.
**Decision:** Multi-entity domains use subdomain folders within the domain folder. Each subdomain mirrors the standard domain structure (enterprise/application for backend, actions/api/components/schemas/store for frontend). Infrastructure layers (controllers, mappers, repositories, events modules) stay flat.
**Options considered:** (A) Subdomain folders within the domain (chosen) / (B) Separate top-level domains per entity
**Rationale:** Option A preserves the bounded context — CropType, Variety, and Harvest share business rules and reference each other. Option B would lose this semantic grouping and make cross-entity relationships implicit. Infrastructure stays flat because NestJS modules are the unit of composition and splitting them per entity adds unnecessary fragmentation.
**Consequences:** Import paths change from `@/domain/crop/enterprise/entities/crop-type` to `@/domain/crop/crop-types/enterprise/entities/crop-type`. All coding pattern docs updated with subdomain notes. See `coding-patterns/frontend/domain-organization.md` and `coding-patterns/backend/domain-organization.md`.
