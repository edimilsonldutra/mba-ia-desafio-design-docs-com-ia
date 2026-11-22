# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) — redigido a partir da reunião técnica |
| **Status** | Em revisão |
| **Data** | Reunião técnica de quinta-feira, 09:00 (`TRANSCRICAO.md`) |
| **Revisores** | Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo (TL;DR)

Vamos notificar clientes B2B por **webhooks de saída** sempre que o status de um pedido mudar. O evento é gravado numa **outbox no MySQL**, na mesma transação da mudança de status. Um **worker separado**, em **polling de 2 s**, entrega cada evento via HTTPS, **assinado com HMAC-SHA256** usando uma secret por endpoint. Falhas são retentadas com **backoff exponencial (5 retentativas, até ~15 h)** e depois vão para uma **DLQ** com replay manual. A entrega é **at-least-once**, com deduplicação pelo cliente via `X-Event-Id`. Tudo segue os padrões atuais do projeto, sem infraestrutura nova. Estimativa: **3 sprints**, já com a revisão de segurança.

## 2. Contexto e problema

- Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser notificadas em tempo real sobre mudanças de status (RFC-CTX-01, [09:00] Marcos).
- Hoje fazem polling em `GET /orders`, o que deixa a integração lenta e cara (RFC-CTX-02, [09:00] Marcos).
- A Atlas sinalizou que pode migrar para um concorrente se a feature não sair no trimestre (RFC-CTX-03, [09:00] Marcos).
- Para os clientes, "tempo real" é qualquer coisa abaixo de 10 s (RFC-CTX-04, [09:02] Marcos).
- O OMS não tem hoje nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status acontece numa transação Prisma em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que já atualiza `orders`, `order_status_history` e o estoque (RFC-CTX-05, [09:04] Bruno).

## 3. Proposta técnica

### 3.1 Visão geral

```
PATCH /api/v1/orders/:id/status
        │
        ▼
OrderService.changeStatus ── $transaction ───────────────────────────┐
  • valida transição / estoque                                        │ atômico
  • update orders + insert order_status_history                       │
  • publishWebhookEvent(tx, …) → INSERT webhook_outbox (filtrado)     │
        └─────────────────────────────────────────────────────────────┘
                                   │ (commit)
npm run worker → src/worker.ts ── a cada 2 s ──► lê PENDING mais antigos
        │
        ├─ POST https://cliente/… (HMAC, X-Event-Id, timeout 10 s)
        ├─ sucesso → DELIVERED + registro em webhook_deliveries
        ├─ falha   → reagenda (1m, 5m, 30m, 2h, 12h)
        └─ esgotou → webhook_dead_letter ──► replay manual (ADMIN)
```

### 3.2 Elementos da proposta

| ID | Elemento | Decisão | ADR |
|---|---|---|---|
| RFC-PROP-01 | Captura do evento | Outbox no MySQL existente, gravada na mesma transação da mudança de status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| RFC-PROP-02 | Despacho | Worker em processo próprio (`src/worker.ts`, `npm run worker`), polling de 2 s, single-worker | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| RFC-PROP-03 | Falhas | Backoff 1m/5m/30m/2h/12h, depois DLQ em tabela separada com replay manual por ADMIN | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| RFC-PROP-04 | Segurança | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com 24 h de convivência, HTTPS obrigatório | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| RFC-PROP-05 | Garantia | At-least-once, deduplicação pelo cliente via `X-Event-Id` | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| RFC-PROP-06 | Código | Módulo `src/modules/webhooks` nos padrões atuais (AppError, Pino, Zod, `requireRole`) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| RFC-PROP-07 | Conteúdo | Filtro por status na inserção e payload como snapshot enxuto, sem itens | [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md) |

### 3.3 Superfície de API (resumo)

A configuração dos webhooks fica sob `/api/v1/webhooks`: criar, listar por customer, editar, remover, rotacionar a secret e consultar o histórico das últimas 100 entregas (RFC-API-01, [09:31]–[09:34] Marcos e Bruno; [09:21] Sofia). Esses endpoints exigem apenas autenticação (RFC-API-02, [09:37] Sofia). O replay da DLQ fica em `/api/v1/admin/webhooks/dead-letter/:id/replay` e exige role ADMIN (RFC-API-03, [09:36] Sofia). O `customer_id` vem no body ou no path, **não** do JWT, porque o token atual identifica o usuário operador (RFC-API-04, [09:32] Bruno, Larissa). Contratos completos no [FDD](FDD.md#5-contratos-públicos).

## 4. Alternativas consideradas

| ID | Alternativa | Por que foi descartada (trade-off) |
|---|---|---|
| RFC-ALT-01 | **Disparo síncrono** dentro de `changeStatus` | Uma chamada HTTP dentro de uma transação já pesada faria um cliente lento travar mudanças de status de outros pedidos, e não há como "dar rollback" num status por causa de um cliente fora do ar ([09:04] Bruno; [09:06] Diego) |
| RFC-ALT-02 | **Redis Streams / fila dedicada** | Exige infraestrutura nova; para um time pequeno, "overengineering". A outbox no MySQL resolve com o que já existe ([09:07] Larissa, Diego) |
| RFC-ALT-03 | **Trigger no banco para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`; o trigger não notifica processo externo e exigiria improviso. O polling de 2 s já atende os 10 s ([09:09] Diego) |
| RFC-ALT-04 | **Retry indefinido** ou **só 3 tentativas** | Indefinido deixa eventos pendurados para sempre; 3 tentativas cobrem ~30 min e perderiam clientes em manutenção de algumas horas ([09:15]–[09:16] Diego, Bruno) |
| RFC-ALT-05 | **Exactly-once** | Exige coordenação dos dois lados e é muito mais complexo; at-least-once com `event_id` resolve "99% dos casos" (padrão Stripe/GitHub) ([09:25] Diego) |
| RFC-ALT-06 | **Secret global da plataforma** | "Se vaza uma, vaza tudo" ([09:21] Sofia) |

## 5. Questões em aberto

| ID | Questão | Situação |
|---|---|---|
| RFC-OPEN-01 | **Rate limiting de saída**: se 50 pedidos de um cliente mudam de status em um minuto, ele recebe 50 chamadas | Fora do escopo; "observar e decidir depois" ([09:38] Diego; [09:39] Larissa) |
| RFC-OPEN-02 | **Escalar para múltiplos workers** sem perder a ordem por pedido (particionar por `order_id` ou lock pessimista) | "Problema do futuro"; registrado como limitação conhecida ([09:13] Diego, Larissa) |
| RFC-OPEN-03 | **Endurecer as permissões** do CRUD de webhooks (hoje qualquer role autenticada) | "Mais pra frente a gente pode endurecer" ([09:37] Sofia) |
| RFC-OPEN-04 | **Avisar o cliente por email** quando o webhook dele falha repetidamente | Adiado para a próxima fase, depois de medir o impacto ([09:37] Larissa) |
| RFC-OPEN-05 | **Arquivamento** de linhas entregues da outbox (~30 dias) | Fora do escopo desta feature ([09:08] Diego) |

## 6. Impacto e riscos

**Impacto no sistema existente.** É uma única alteração crítica em código existente: `changeStatus` passa a chamar `publishWebhookEvent` dentro da transação; se a inserção falhar, a mudança de status dá rollback (RFC-IMP-01, [09:40] Bruno). O restante é aditivo: um módulo novo, tabelas novas via migration Prisma, o entry point `src/worker.ts` e o script `npm run worker` (RFC-IMP-02, [09:11] Larissa).

**Riscos principais**

| ID | Risco | Mitigação |
|---|---|---|
| RFC-RISK-01 | Evento perdido ou emitido para mudança que deu rollback | Inserção na mesma transação ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) |
| RFC-RISK-02 | Cliente indisponível por horas | Backoff de até ~15 h, DLQ e replay ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)) |
| RFC-RISK-03 | Vazamento de secret do cliente | Secret por endpoint e rotação com grace de 24 h ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md); [09:22] Diego) |
| RFC-RISK-04 | Entrega duplicada | At-least-once explícito com `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) |
| RFC-RISK-05 | Ordem perdida ao escalar workers | Single-worker agora; ver RFC-OPEN-02 |

**Prazo.** Três sprints, incluindo pelo menos 2 dias úteis de revisão de segurança (HMAC e geração de secret) antes do deploy (RFC-IMP-03, [09:46] Larissa, Sofia). A Atlas quer a feature até o fim de novembro ([09:45] Marcos).

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling de 2 s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Snapshot e filtro na inserção](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)
