# Roadmap MVP

## Alpha — Ciclo Operacional Completo

> O gerente consegue planejar, imprimir, executar e finalizar uma boleta do inicio ao fim.

### Feito

- 10 dominios implementados (User, Field, Crop, Schedule, FieldTicket, Inventory, Supplier, Fleet, Employee, Audit)
- 109 use cases no backend, 50+ rotas no frontend
- Painel de execucao com fluxo Rascunho -> Revisada -> Concluida
- Criacao, edicao, revisao e cancelamento de boletas
- CRUD completo de todos os modulos de suporte

### Pendente

| Item | Tipo | Escopo |
|------|------|--------|
| UI de impressao da boleta | feature | Layout 6 por A4 landscape, botao no painel, status REVIEWED->PRINTED |
| UI de finalizacao da boleta | feature | Sheet com dados de execucao (horarios, horimetro, dosagem executada), status PRINTED->COMPLETED |
| Ajustes visuais | polish | Refinamentos de UI identificados durante uso |

---

## Beta — Enriquecimento dos Modulos

> Todos os modulos tem os campos necessarios para uso real na fazenda.

Os modulos foram construidos com o minimo para o fluxo da boleta funcionar. No Beta, cada modulo e revisado para adicionar campos, validacoes e funcionalidades que faltam para uso real.

| Modulo | Campos/Features pendentes |
|--------|--------------------------|
| **Field** | Localizacao (coordenadas/mapa), tipo de solo, cultura atual |
| **Crop/Harvest** | Produtividade esperada, custo estimado, observacoes |
| **Fleet/Vehicle** | Mais tipos (MOTORCYCLE, CAR, TRUCK), combustivel, consumo, horimetro atual |
| **Fleet/Implement** | Compatibilidade veiculo-implemento |
| **Inventory/Input** | Fabricante, codigo comercial, concentracao |
| **Supplier** | CNPJ, endereco, contato adicional |
| **Employee** | Migracao operatorName -> employeeId no FieldTicket |
| **Schedule** | Status automatico (PLANNED->ACTIVE->UNDER_REVIEW->COMPLETED) — ADR ja registrado |
| **FieldTicket** | Reavaliacao (UI para editar COMPLETED com motivo) |
| **Execution Panel** | Secao "Outros Dias" (compact rows com resumo da semana) |
| **Dashboard** | KPIs basicos (operacoes do dia, estoque baixo, cronogramas ativos) |
| **Geral** | Filtros avancados por status em todas as listagens, busca global |

---

## MVP — Entrega Final Validada

> Multi-tenant, painel admin, tudo testado e pronto para producao.

| Item | Tipo |
|------|------|
| **Multi-tenant** — tenantId em todas as entidades, isolamento de dados por fazenda, criacao de tenant | arquitetura |
| **Admin panel** — painel separado para monitorar fazendas, usuarios, metricas de uso | feature |
| Revisao completa de todos os fluxos (flows.md vs implementacao) | validacao |
| Playwright E2E cobrindo todos os fluxos documentados | testes |
| Mutation testing em todos os dominios (score >= 90%) | qualidade |
| Health-check completo (deps, seguranca, performance) | qualidade |
| Documentacao atualizada (api-reference, architecture, glossary) | docs |
| Seed data representativo para demo | dados |
| Validacao visual completa (responsividade, dark mode, acessibilidade) | UX |
| Bug fixes encontrados durante validacao | correcao |
