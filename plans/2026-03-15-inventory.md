# Plan: Inventory Domain

**Issue:** fellipefreiire/farm-app-docs#17
**Branch:** `FS-17/inventory`
**Type:** New Domain (multi-entity with subdomains, same pattern as Crop)

---

## Domain Structure

Multi-entity domain with 4 subdomains:

```
inventory/
  categories/        ← Category entity
  inputs/            ← Input entity
  purchases/         ← Purchase + PurchaseItem (aggregate)
  stock-movements/   ← StockMovement entity
```

Infrastructure modules (NestJS) stay flat at domain level.

---

## Implementation Strategy

Implemented in 4 incremental phases (A→D) within a single branch/PR. Each phase adds one subdomain, validates, then proceeds.

### Phase A — Category (no dependencies)
### Phase B — Input (depends on Category)
### Phase C — Purchase + PurchaseItem (depends on Input + Supplier)
### Phase D — StockMovement + GetStockBalance (depends on Input + Purchase)

---

## File List

### Backend

#### Phase A — Category

**Wave 1 — Domain Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/categories/enterprise/entities/category.ts` | Entity (AggregateRoot) |
| `src/domain/inventory/categories/enterprise/events/category-created-event.ts` | Domain event |
| `src/domain/inventory/categories/enterprise/events/category-updated-event.ts` | Domain event |
| `src/domain/inventory/categories/enterprise/events/category-deleted-event.ts` | Domain event |
| `src/domain/inventory/categories/enterprise/events/category-active-status-changed-event.ts` | Domain event |
| `src/domain/inventory/categories/application/use-cases/errors/category-not-found-error.ts` | Error |
| `src/domain/inventory/categories/application/use-cases/errors/category-already-exists-error.ts` | Error |

**Wave 2 — Application Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/categories/application/repositories/categories-repository.ts` | Repository interface |
| `src/domain/inventory/categories/application/use-cases/create-category.ts` | Use case |
| `src/domain/inventory/categories/application/use-cases/edit-category.ts` | Use case |
| `src/domain/inventory/categories/application/use-cases/delete-category.ts` | Use case |
| `src/domain/inventory/categories/application/use-cases/toggle-category-status.ts` | Use case |
| `src/domain/inventory/categories/application/use-cases/list-categories.ts` | Use case |
| `src/domain/inventory/categories/application/use-cases/list-category-audit-logs.ts` | Use case |

**Wave 3 — Tests**
| File | Description |
|------|-------------|
| `test/factories/make-category.ts` | Factory |
| `test/repositories/inventory/categories/in-memory-categories-repository.ts` | InMemory repo |
| `src/domain/inventory/categories/application/use-cases/__tests__/create-category.spec.ts` | Unit test |
| `src/domain/inventory/categories/application/use-cases/__tests__/edit-category.spec.ts` | Unit test |
| `src/domain/inventory/categories/application/use-cases/__tests__/delete-category.spec.ts` | Unit test |
| `src/domain/inventory/categories/application/use-cases/__tests__/toggle-category-status.spec.ts` | Unit test |
| `src/domain/inventory/categories/application/use-cases/__tests__/list-categories.spec.ts` | Unit test |

**Wave 4 — Database**
| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Add Category model |
| `src/infra/database/prisma/mappers/inventory/prisma-category.mapper.ts` | Mapper |
| `src/infra/database/prisma/repositories/inventory/prisma-categories.repository.ts` | Prisma repo |
| `src/infra/database/prisma/repositories/inventory/inventory-database.module.ts` | Database module (shared across all inventory subdomains) |

**Wave 5 — HTTP**
| File | Description |
|------|-------------|
| `src/infra/http/presenters/category.presenter.ts` | Presenter |
| `src/infra/http/dtos/requests/inventory/` | Request DTOs |
| `src/infra/http/dtos/response/inventory/` | Response DTOs |
| `src/infra/http/dtos/error/inventory/` | Error DTOs |
| `src/infra/http/filters/inventory-error.filter.ts` | Error filter (shared across all inventory subdomains) |
| `src/infra/http/controllers/inventory/create-category.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/edit-category.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/delete-category.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/toggle-category-status.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/list-categories.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/list-category-audit-logs.controller.ts` | Controller |
| `src/infra/http/controllers/inventory/inventory-http.module.ts` | HTTP module (shared) |
| `src/infra/http/controllers/inventory/__tests__/create-category.controller.e2e-spec.ts` | E2E test |
| `src/infra/http/controllers/inventory/__tests__/edit-category.controller.e2e-spec.ts` | E2E test |
| `src/infra/http/controllers/inventory/__tests__/delete-category.controller.e2e-spec.ts` | E2E test |
| `src/infra/http/controllers/inventory/__tests__/toggle-category-status.controller.e2e-spec.ts` | E2E test |
| `src/infra/http/controllers/inventory/__tests__/list-categories.controller.e2e-spec.ts` | E2E test |

**Wave 6 — Registration**
| File | Description |
|------|-------------|
| `src/infra/http/controllers/controllers.module.ts` | Register InventoryHttpModule |
| `src/infra/auth/casl/models/category.ts` | CASL model |
| `src/infra/auth/casl/subjects/category.ts` | CASL subject |
| `src/infra/auth/casl/casl-ability.factory.ts` | Add categorySubject |
| `src/infra/auth/casl/permissions.ts` | Add Category permissions |

---

#### Phase B — Input

**Wave 1 — Domain Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/inputs/enterprise/entities/input.ts` | Entity |
| `src/domain/inventory/inputs/enterprise/events/input-created-event.ts` | Domain event |
| `src/domain/inventory/inputs/enterprise/events/input-updated-event.ts` | Domain event |
| `src/domain/inventory/inputs/enterprise/events/input-deleted-event.ts` | Domain event |
| `src/domain/inventory/inputs/enterprise/events/input-active-status-changed-event.ts` | Domain event |
| `src/domain/inventory/inputs/application/use-cases/errors/input-not-found-error.ts` | Error |
| `src/domain/inventory/inputs/application/use-cases/errors/input-already-exists-error.ts` | Error |

**Wave 2 — Application Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/inputs/application/repositories/inputs-repository.ts` | Repository interface |
| `src/domain/inventory/inputs/application/use-cases/create-input.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/edit-input.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/delete-input.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/toggle-input-status.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/find-input-by-id.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/list-inputs.ts` | Use case |
| `src/domain/inventory/inputs/application/use-cases/list-input-audit-logs.ts` | Use case |

**Wave 3 — Tests**
| File | Description |
|------|-------------|
| `test/factories/make-input.ts` | Factory |
| `test/repositories/inventory/inputs/in-memory-inputs-repository.ts` | InMemory repo |
| `src/domain/inventory/inputs/application/use-cases/__tests__/create-input.spec.ts` | Unit test |
| `src/domain/inventory/inputs/application/use-cases/__tests__/edit-input.spec.ts` | Unit test |
| `src/domain/inventory/inputs/application/use-cases/__tests__/delete-input.spec.ts` | Unit test |
| `src/domain/inventory/inputs/application/use-cases/__tests__/toggle-input-status.spec.ts` | Unit test |
| `src/domain/inventory/inputs/application/use-cases/__tests__/find-input-by-id.spec.ts` | Unit test |
| `src/domain/inventory/inputs/application/use-cases/__tests__/list-inputs.spec.ts` | Unit test |

**Wave 4 — Database**
| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Add Input model + UnitOfMeasure enum |
| `src/infra/database/prisma/mappers/inventory/prisma-input.mapper.ts` | Mapper |
| `src/infra/database/prisma/repositories/inventory/prisma-inputs.repository.ts` | Prisma repo |
| Update `inventory-database.module.ts` | Register InputsRepository |

**Wave 5 — HTTP**
| File | Description |
|------|-------------|
| `src/infra/http/presenters/input.presenter.ts` | Presenter |
| Controllers: create, edit, delete, toggle, find, list, audit-logs | 7 controllers |
| E2E tests for each controller | 6 E2E tests |
| Update `inventory-error.filter.ts` | Add Input errors |
| Update `inventory-http.module.ts` | Register Input controllers + use cases |

**Wave 6 — CASL**
| File | Description |
|------|-------------|
| `src/infra/auth/casl/models/input.ts` | CASL model |
| `src/infra/auth/casl/subjects/input.ts` | CASL subject |
| Update `casl-ability.factory.ts` + `permissions.ts` | Add Input permissions |

---

#### Phase C — Purchase + PurchaseItem

**Wave 1 — Domain Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/purchases/enterprise/entities/purchase.ts` | Purchase entity (aggregate root) |
| `src/domain/inventory/purchases/enterprise/entities/purchase-item.ts` | PurchaseItem entity (child of Purchase) |
| `src/domain/inventory/purchases/enterprise/events/purchase-created-event.ts` | Domain event |
| `src/domain/inventory/purchases/enterprise/events/purchase-updated-event.ts` | Domain event |
| `src/domain/inventory/purchases/application/use-cases/errors/purchase-not-found-error.ts` | Error |
| `src/domain/inventory/purchases/application/use-cases/errors/empty-purchase-items-error.ts` | Error |

**Wave 2 — Application Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/purchases/application/repositories/purchases-repository.ts` | Repository interface |
| `src/domain/inventory/purchases/application/use-cases/create-purchase.ts` | Use case (validates supplier + inputs) |
| `src/domain/inventory/purchases/application/use-cases/edit-purchase.ts` | Use case (requires editReason) |
| `src/domain/inventory/purchases/application/use-cases/find-purchase-by-id.ts` | Use case |
| `src/domain/inventory/purchases/application/use-cases/list-purchases.ts` | Use case |
| `src/domain/inventory/purchases/application/use-cases/list-purchase-audit-logs.ts` | Use case |

**Wave 3 — Tests**
| File | Description |
|------|-------------|
| `test/factories/make-purchase.ts` | Factory (includes PurchaseItems) |
| `test/repositories/inventory/purchases/in-memory-purchases-repository.ts` | InMemory repo |
| Unit tests for all 5 use cases | 5 spec files |

**Wave 4 — Database**
| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Add Purchase + PurchaseItem models |
| `src/infra/database/prisma/mappers/inventory/prisma-purchase.mapper.ts` | Mapper (with items) |
| `src/infra/database/prisma/repositories/inventory/prisma-purchases.repository.ts` | Prisma repo |
| Update `inventory-database.module.ts` | Register PurchasesRepository |

**Wave 5 — HTTP**
| File | Description |
|------|-------------|
| `src/infra/http/presenters/purchase.presenter.ts` | Presenter (with items denormalized) |
| Controllers: create, edit, find, list, audit-logs | 5 controllers |
| E2E tests | 4 E2E tests |
| Update `inventory-error.filter.ts` + `inventory-http.module.ts` | Add Purchase controllers |

**Wave 6 — CASL**
| File | Description |
|------|-------------|
| `src/infra/auth/casl/models/purchase.ts` | CASL model |
| `src/infra/auth/casl/subjects/purchase.ts` | CASL subject |
| Update `casl-ability.factory.ts` + `permissions.ts` | Add Purchase permissions |

---

#### Phase D — StockMovement + GetStockBalance

**Wave 1 — Domain Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/stock-movements/enterprise/entities/stock-movement.ts` | Entity |
| `src/domain/inventory/stock-movements/enterprise/events/stock-movement-created-event.ts` | Domain event |
| `src/domain/inventory/stock-movements/enterprise/events/stock-movement-updated-event.ts` | Domain event |
| `src/domain/inventory/stock-movements/enterprise/events/stock-movement-cancelled-event.ts` | Domain event |
| `src/domain/inventory/stock-movements/application/use-cases/errors/stock-movement-not-found-error.ts` | Error |
| `src/domain/inventory/stock-movements/application/use-cases/errors/stock-movement-already-cancelled-error.ts` | Error |

**Wave 2 — Application Layer**
| File | Description |
|------|-------------|
| `src/domain/inventory/stock-movements/application/repositories/stock-movements-repository.ts` | Repository interface |
| `src/domain/inventory/stock-movements/application/use-cases/create-stock-movement.ts` | Use case |
| `src/domain/inventory/stock-movements/application/use-cases/edit-stock-movement.ts` | Use case |
| `src/domain/inventory/stock-movements/application/use-cases/cancel-stock-movement.ts` | Use case |
| `src/domain/inventory/stock-movements/application/use-cases/list-stock-movements.ts` | Use case |
| `src/domain/inventory/stock-movements/application/use-cases/get-stock-balance.ts` | Use case (cross-repo: queries PurchaseItems + StockMovements) |

**Wave 3 — Tests**
| File | Description |
|------|-------------|
| `test/factories/make-stock-movement.ts` | Factory |
| `test/repositories/inventory/stock-movements/in-memory-stock-movements-repository.ts` | InMemory repo |
| Unit tests for all 5 use cases | 5 spec files |

**Wave 4 — Database**
| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Add StockMovement model + StockMovementType/Reason enums |
| `src/infra/database/prisma/mappers/inventory/prisma-stock-movement.mapper.ts` | Mapper |
| `src/infra/database/prisma/repositories/inventory/prisma-stock-movements.repository.ts` | Prisma repo |
| Update `inventory-database.module.ts` | Register StockMovementsRepository |

**Wave 5 — HTTP**
| File | Description |
|------|-------------|
| `src/infra/http/presenters/stock-movement.presenter.ts` | Presenter |
| Controllers: create, edit, cancel, list, get-stock-balance | 5 controllers |
| E2E tests | 4 E2E tests |
| Update `inventory-error.filter.ts` + `inventory-http.module.ts` | Add StockMovement controllers |

**Wave 6 — CASL**
| File | Description |
|------|-------------|
| `src/infra/auth/casl/models/stock-movement.ts` | CASL model |
| `src/infra/auth/casl/subjects/stock-movement.ts` | CASL subject |
| Update `casl-ability.factory.ts` + `permissions.ts` | Add StockMovement permissions |

---

### Frontend

Each subdomain has its own independent page in the menu.

#### Phase A — Category
- `src/domains/inventory/categories/` — schemas, api, actions, store, components
- `src/app/(private)/categories/` — list, detail, audit pages
- `tests/mocks/handlers/category.handler.ts` — MSW handler
- `tests/inventory/manage-categories.e2e-spec.ts` — Playwright E2E

#### Phase B — Input
- `src/domains/inventory/inputs/` — schemas, api, actions, store, components
- `src/app/(private)/inputs/` — list, detail, audit pages
- `tests/mocks/handlers/input.handler.ts` — MSW handler
- `tests/inventory/manage-inputs.e2e-spec.ts` — Playwright E2E

#### Phase C — Purchase
- `src/domains/inventory/purchases/` — schemas, api, actions, store, components
- `src/app/(private)/purchases/` — list, detail, audit pages
- Special: create/edit form with dynamic item list (add/remove PurchaseItems)
- `tests/mocks/handlers/purchase.handler.ts` — MSW handler
- `tests/inventory/manage-purchases.e2e-spec.ts` — Playwright E2E

#### Phase D — StockMovement
- `src/domains/inventory/stock-movements/` — schemas, api, actions, store, components
- `src/app/(private)/stock-movements/` — list page (no detail page — simpler entity)
- `tests/mocks/handlers/stock-movement.handler.ts` — MSW handler
- `tests/inventory/manage-stock-movements.e2e-spec.ts` — Playwright E2E

#### Navigation
- Update sidebar: add "Categorias", "Insumos", "Compras", "Saídas de Estoque" under the existing "Estoque" menu group

---

## Integration Points

- **CASL:** Add Category, Input, Purchase, StockMovement subjects
- **Audit:** Domain events follow existing pattern — Audit subscriber picks them up
- **ControllersModule:** Register InventoryHttpModule
- **Supplier domain:** Purchase references Supplier by ID — use SuppliersRepository for validation in CreatePurchase

---

## Notes

- PurchaseItem is NOT a standalone entity — it's embedded in Purchase (created/edited together)
- Stock balance is never stored — always calculated via GetStockBalance use case
- StockMovement cancel adds a `cancelledAt` field — cancelled movements excluded from balance calculation
- Purchase cannot be deleted — only edited with mandatory editReason
- Frontend: Category and Input are independent pages (NOT nested like Variety under CropType)
