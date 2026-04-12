# Glossary

Project-specific terms and their definitions. This file ensures consistent language between the development team and Claude across all conversations.

**Why this matters:** When the team uses domain-specific terms (e.g. "batch", "tenant", "workflow"), Claude needs to understand exactly what they mean in the context of this project. Without a glossary, Claude may assume wrong meanings or ask repeatedly about the same terms.

---

## How to use

### For the developer

Add terms as the project evolves. Each term should have:
- **Term** — the word or phrase as used in conversations and code
- **Definition** — what it means in this project (1-2 sentences)
- **Context** *(optional)* — where it appears (code, UI, business rule) or an example

```markdown
### Batch
A group of products created together in a single production run. Each batch has a unique code and tracks quality metrics.
- **Context:** `Batch` entity in the `production` domain. UI shows batch list with status filters.
```

### For Claude

- **During setup:** After `product.md` and `architecture.md` are ready, prompt the developer to fill this glossary with key terms from those documents.
- **During development:** When you encounter an unfamiliar project-specific term in conversation, code, or docs — do not assume its meaning. Ask the developer for context, then add it here.
- **During code review (Phase 5):** Verify that variable names, entity names, and comments use terms consistent with this glossary. Flag any new domain term that appears in code or conversation but is not listed here.
- **When adding terms:** Use the format above. Keep definitions concise and project-specific.

---

## Terms

### Domain
A bounded context with its own entities, use cases, and repository. The primary organizational unit of the codebase (e.g., `orders`, `billing`, `users`).
- **Context:** Maps to a folder under `src/domain/<domain>/` in backend and `src/domains/<domain>/` in frontend.

### Module (NestJS)
A NestJS `@Module({})` class that wires controllers, providers, and imports. One domain typically has one or more NestJS modules.
- **Context:** Files like `<domain>-controllers.module.ts`, `<domain>-database.module.ts`, `<domain>-events.module.ts`.

### Module (architecture)
Synonym for "domain" in architectural discussions. Prefer "domain" for clarity.
- **Context:** Used in CLAUDE.md rules (e.g., "module boundaries", "cross-module communication"). Refers to domains, not NestJS modules.

### Talhao (Talhão)
A delimited area of land within a farm, used for planting a specific crop. Code: `Field`
- **Context:** `Field` entity in the `field` domain. UI label: "Talhão".

### Safra
A crop cycle — the period from planting to harvest of a specific crop/variety in a field. Code: `Crop`
- **Context:** `Crop` entity in the `crop` domain. UI label: "Safra".

### Cronograma
A per-field plan that defines which operations will be performed on which days, and which inputs each operation will use. Code: `Schedule`
- **Context:** `Schedule` entity in the `schedule` domain. UI label: "Cronograma".

### Boleta
A printed work order given to the tractor operator. Contains which products to apply in which field on a given day. Generated from the schedule. Code: `FieldTicket`
- **Context:** `FieldTicket` entity in the `field-ticket` domain. UI label: "Boleta".

### Pulverização
The operation of spraying agricultural inputs (pesticides, herbicides, fungicides) on a field. Code: `Spraying`
- **Context:** One of the operation types in a schedule. UI label: "Pulverização".

### Fertirrigação
The operation of applying fertilizer through irrigation systems. Code: `Fertigation`
- **Context:** One of the operation types in a schedule. UI label: "Fertirrigação".

### Insumo
Any agricultural input used in field operations — pesticides, fertilizers, seeds, adjuvants, etc. Code: `Input`
- **Context:** `InventoryItem` entity in the `inventory` domain. UI label: "Insumo".

### Estoque
The stock of inputs, seeds, and harvested products available on the farm. Code: `Inventory`
- **Context:** `Inventory` domain. UI label: "Estoque".

### Fornecedor
A supplier from whom the farm purchases inputs. Code: `Supplier`
- **Context:** `Supplier` entity in the `supplier` domain. UI label: "Fornecedor".

### Frota
The collection of vehicles and implements owned by the farm. Code: `Fleet`
- **Context:** `Fleet` domain. UI label: "Frota".

### Veículo
A motorized vehicle used in farm operations (initially tractors only). Code: `Vehicle`
- **Context:** `Vehicle` entity in the `fleet` domain (vehicles subdomain). UI label: "Veículo".

### Implemento
An agricultural attachment or tool used with vehicles (sprayers, spreaders, planters, etc.). Code: `Implement`
- **Context:** `Implement` entity in the `fleet` domain (implements subdomain). UI label: "Implemento".

### Funcionário
An employee registered in the farm, linked to a position (cargo). Not necessarily a system user. Code: `Employee`
- **Context:** `Employee` entity in the `employee` domain. UI label: "Funcionário".

### Cargo
An organizational position within the farm (e.g., Owner, Gerente, Operador, Tratorista). Code: `Position`
- **Context:** `Position` entity in the `employee` domain. UI label: "Cargo".

### Auditoria
A record of every user mutation in the system — who did what, when, and what changed. Code: `AuditLog`
- **Context:** `AuditLog` entity in the `audit-log` domain. Cross-cutting: subscribes to domain events from all domains. UI label: "Histórico".
