# FDD — Feature Design Document: Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks de saída para mudança de status de pedido |
| **Status** | Pronto para implementação (sujeito à revisão de segurança) |
| **Responsáveis** | Bruno (Pedidos), Diego (Plataforma); revisão: Sofia (Segurança), Larissa (Tech Lead) |
| **Base** | [RFC](RFC.md) · [ADRs](adrs/) · [PRD](PRD.md) · [Tracker](TRACKER.md) |

Convenção: cada item rastreável tem um ID (`FDD-…`) listado no [Tracker](TRACKER.md). Itens marcados com **(decisão de implementação)** não foram discutidos na reunião. São escolhas deste FDD para torná-lo acionável e precisam ser validadas na revisão com Bruno e Diego ([09:50] Larissa).

---

## 1. Contexto e motivação técnica

- O OMS é uma API Node.js 20 + TypeScript + Express + Prisma (MySQL 8). Não tem hoje eventos, filas nem notificações externas (`package.json`, `docker-compose.yml`).
- Clientes B2B fazem polling em `GET /api/v1/orders` para detectar mudanças de status ([09:00] Marcos).
- Toda mudança de status passa por `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`). Esse método abre uma transação (`$transaction`, linha 131), valida a transição com `canTransition` (`src/modules/orders/order.status.ts:12`), ajusta o estoque, atualiza `orders` e insere em `order_status_history`.
- Esse é o único ponto onde o evento pode nascer com garantia de consistência. A notificação **não pode** ser feita de forma síncrona ali ([09:04] Bruno), e por isso adotamos outbox + worker ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
|---|---|---|
| FDD-OBJ-01 | Toda mudança de status commitada que interessa a algum webhook gera exatamente um evento na outbox, na mesma transação | [09:40] Bruno |
| FDD-OBJ-02 | Primeira tentativa de entrega em < 10 s após o commit (polling de 2 s) | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | Requisições assinadas com HMAC-SHA256 e secret por endpoint | [09:22] Sofia |
| FDD-OBJ-04 | Falhas retentadas com backoff 1m/5m/30m/2h/12h e depois enviadas à DLQ | [09:17] Larissa |
| FDD-OBJ-05 | Nenhuma dependência de infraestrutura ou biblioteca nova | [09:07] Diego; [09:29] Bruno |

## 3. Escopo e exclusões

**Incluso:** CRUD de configuração de webhooks, rotação de secret, histórico de entregas, inserção na outbox em `changeStatus`, worker de entrega com retry e DLQ, replay manual por ADMIN.

**Excluído / adiado**

| ID | Item | Origem |
|---|---|---|
| FDD-EXC-01 | Webhooks de entrada (*inbound*) | [09:02] Marcos |
| FDD-EXC-02 | Email ao cliente quando o webhook falha | [09:37] Larissa |
| FDD-EXC-03 | Rate limiting de saída (observar e decidir depois) | [09:39] Larissa |
| FDD-EXC-04 | Dashboard visual | [09:40] Larissa |
| FDD-EXC-05 | Arquivamento de linhas entregues da outbox | [09:08] Diego |
| FDD-EXC-06 | Múltiplos workers / ordem global | [09:13] Diego |
| FDD-EXC-07 | Itens do pedido no payload | [09:43] Diego |

## 4. Modelo de dados

Migration Prisma aditiva em `prisma/schema.prisma`. Todos os IDs são UUID `@db.Char(36)`, como nos demais modelos ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  FAILED      // tentativas esgotadas: o evento foi copiado para webhook_dead_letter
  DELIVERED
}

model WebhookEndpoint {                     // FDD-DADOS-01
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[] assinados, ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  deletedAt               DateTime? // remoção lógica (DELETE)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer             @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {                       // FDD-DADOS-02
  id            String              @id @default(uuid()) @db.Char(36)   // = event_id / X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)                     // "order.status_changed"
  payload       Json                                                    // snapshot renderizado
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([webhookId, orderId, createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {                     // FDD-DADOS-03
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  webhookId      String   @db.Char(36)
  attempt        Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.VarChar(2048)  // truncado
  durationMs     Int
  errorCode      String?  @db.VarChar(64)    // WEBHOOK_DELIVERY_TIMEOUT etc.
  createdAt      DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {                   // FDD-DADOS-04
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @unique @db.Char(36)
  webhookId     String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)

  @@index([webhookId])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:
- **Uma linha de outbox por (evento × webhook assinante)** (decisão de implementação). Cada endpoint tem o seu próprio ciclo de retry e DLQ e o seu próprio `X-Event-Id`, coerente com `X-Webhook-Id` ([09:44] Sofia) e com o filtro por webhook ([09:33] Marcos).
- `status` com índice e `created_at` com índice, conforme [09:08] Diego. O índice composto `status, nextAttemptAt` atende a consulta do worker.
- `secret` em texto recuperável, porque é necessário para assinar. Ele **nunca** é devolvido fora da criação e da rotação e é redigido nos logs ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- `Customer` ganha a relação `webhooks WebhookEndpoint[]`.

## 5. Contratos públicos

Prefixo `/api/v1` (`src/app.ts:67`). Autenticação via `Authorization: Bearer <JWT>` (`authenticate`, `src/middlewares/auth.middleware.ts:27`). Erros no formato padrão do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`.

### FDD-CONTRATO-01 — Criar webhook · `POST /api/v1/webhooks`
Origem: [09:31] Marcos (url, secret gerada e devolvida, lista de status); [09:32] Larissa (`customerId` no body). Qualquer role autenticada ([09:37] Sofia).

```http
POST /api/v1/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "customerId": "6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```
```http
201 Created
{
  "id": "0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f",
  "customerId": "6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "q3Zr0bH2m8YwK1d9xV7pN4sT6uE5cA0fJ2gL8iO1hRk",
  "createdAt": "2026-11-05T12:00:00.000Z",
  "updatedAt": "2026-11-05T12:00:00.000Z"
}
```
Status: `201`; `400 VALIDATION_ERROR` (schema); `400 WEBHOOK_INVALID_URL`; `400 WEBHOOK_INVALID_EVENTS`; `401 UNAUTHORIZED`; `404 WEBHOOK_CUSTOMER_NOT_FOUND`.
O `secret` só aparece nesta resposta e na rotação.

### FDD-CONTRATO-02 — Listar webhooks de um customer · `GET /api/v1/webhooks?customerId=…`
Origem: [09:33] Bruno. Paginação no padrão `paginated()` (`src/shared/http/response.ts:22`).

```http
GET /api/v1/webhooks?customerId=6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c&page=1&pageSize=20
Authorization: Bearer <jwt>
```
```http
200 OK
{
  "data": [
    {
      "id": "0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f",
      "customerId": "6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-05T12:00:00.000Z",
      "updatedAt": "2026-11-05T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Status: `200`; `400 VALIDATION_ERROR` (sem `customerId` ou UUID inválido); `401`.

### FDD-CONTRATO-03 — Editar webhook · `PATCH /api/v1/webhooks/:id`
Origem: [09:33] Bruno.

```http
PATCH /api/v1/webhooks/0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f
Authorization: Bearer <jwt>
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
```http
200 OK
{
  "id": "0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f",
  "customerId": "6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-11-05T12:00:00.000Z",
  "updatedAt": "2026-11-06T09:30:00.000Z"
}
```
Status: `200`; `400 VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` / `WEBHOOK_INVALID_EVENTS`; `401`; `404 WEBHOOK_NOT_FOUND`.
Campos aceitos: `url`, `events`, `active`. A edição não afeta eventos já enfileirados ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md)).

### FDD-CONTRATO-04 — Remover webhook · `DELETE /api/v1/webhooks/:id`
Origem: [09:33] Bruno.

```http
DELETE /api/v1/webhooks/0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f
Authorization: Bearer <jwt>
```
```http
204 No Content
```
Status: `204`; `401`; `404 WEBHOOK_NOT_FOUND`.
Semântica **(decisão de implementação)**: a remoção é lógica (`deletedAt = now`, `active = false`; diferente de só desativar via PATCH), e o webhook removido deixa de aparecer na listagem e de ser aceito nos demais endpoints (`404 WEBHOOK_NOT_FOUND`). Eventos `PENDING` desse webhook passam para `FAILED` com o motivo `webhook removed`, sem ir para a DLQ. O histórico de entregas é preservado para auditoria.

### FDD-CONTRATO-05 — Rotacionar secret · `POST /api/v1/webhooks/:id/rotate-secret`
Origem: [09:21] Sofia (rotação pela API, grace period de 24 h).

```http
POST /api/v1/webhooks/0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f/rotate-secret
Authorization: Bearer <jwt>
```
```http
200 OK
{
  "id": "0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f",
  "secret": "Xk8vP1nQ5rT3wY7zA2cE6gI0kM4oS8uW1yB5dF9hJ3l",
  "previousSecretExpiresAt": "2026-11-07T09:30:00.000Z"
}
```
Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`.
A secret anterior vai para `previousSecret`, com expiração em `now + 24h`. Uma nova rotação dentro do grace period descarta a secret mais antiga.

### FDD-CONTRATO-06 — Histórico de entregas · `GET /api/v1/webhooks/:id/deliveries`
Origem: [09:34] Marcos (últimas 100, sucesso/falha, payload, resposta, tempo de resposta).

```http
GET /api/v1/webhooks/0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f/deliveries
Authorization: Bearer <jwt>
```
```http
200 OK
{
  "data": [
    {
      "id": "5c9e1a2b-7d3f-4b6a-9e8c-1f2a3b4c5d6e",
      "eventId": "a3f5c1d2-8b7e-4f6a-9c0d-2e1f3a4b5c6d",
      "attempt": 1,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": { "event_id": "a3f5c1d2-8b7e-4f6a-9c0d-2e1f3a4b5c6d", "event_type": "order.status_changed", "…": "…" },
      "createdAt": "2026-11-06T10:15:02.410Z"
    }
  ]
}
```
Status: `200` (no máximo 100 itens, do mais recente para o mais antigo); `401`; `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-07 — Replay da DLQ · `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
Origem: [09:18] Diego (recoloca na outbox como pendente); [09:36] Sofia (role ADMIN e log de auditoria).

```http
POST /api/v1/admin/webhooks/dead-letter/7e2d4c6b-1a3f-4e5d-8b9c-0a1b2c3d4e5f/replay
Authorization: Bearer <jwt de usuário ADMIN>
```
```http
202 Accepted
{
  "deadLetterId": "7e2d4c6b-1a3f-4e5d-8b9c-0a1b2c3d4e5f",
  "eventId": "a3f5c1d2-8b7e-4f6a-9c0d-2e1f3a4b5c6d",
  "status": "PENDING",
  "replayedAt": "2026-11-07T08:00:00.000Z",
  "replayedById": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f"
}
```
Status: `202`; `401`; `403 FORBIDDEN` (`requireRole('ADMIN')`); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`; `409 WEBHOOK_INACTIVE`.
O replay mantém o **mesmo `event_id`** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)), zera `attempts`, define `nextAttemptAt = now` e status `PENDING`.

### FDD-CONTRATO-08 — Requisição de saída (o que o cliente recebe)
Origem: [09:43] Diego (payload); [09:44] Diego e Sofia (headers); [09:42] Diego (timeout).

```http
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: a3f5c1d2-8b7e-4f6a-9c0d-2e1f3a4b5c6d
X-Webhook-Id: 0b7d2c1e-9f4a-4e3b-8c2d-5a6b7c8d9e0f
X-Timestamp: 2026-11-06T10:15:02.226Z
X-Signature: sha256=5d41402abc4b2a76b9719d911017c592ae2b5f4f2c8a1e0f3d9b7c6a5e4f3a2b

{
  "event_id": "a3f5c1d2-8b7e-4f6a-9c0d-2e1f3a4b5c6d",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-06T10:15:00.512Z",
  "order_id": "9d8c7b6a-5f4e-4d3c-2b1a-0f9e8d7c6b5a",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a7e-3b0d-4c8e-9a51-1d2e3f4a5b6c",
  "total_cents": 459900
}
```
- `timestamp` (payload) = momento da mudança de status (snapshot). `X-Timestamp` = momento do envio ([09:44] Diego).
- `X-Signature` = `sha256=` + HMAC-SHA256 em hex dos **bytes exatos do corpo** com a secret do endpoint ([09:22] Sofia). Durante o grace period de rotação, o header leva as duas assinaturas separadas por vírgula, a nova primeiro: `sha256=<nova>,sha256=<anterior>` **(decisão de implementação** para viabilizar o grace de 24 h com webhooks de saída).
- Sucesso = resposta `2xx` em até 10 s. Qualquer outra resposta, timeout ou erro de rede conta como falha.

## 6. Fluxos detalhados

### FDD-FLUXO-01 — Criação do evento na outbox (API, síncrono na transação)
Origem: [09:40]–[09:41] Bruno e Diego; [09:34] Bruno (filtro); [09:52] (snapshot).

1. `PATCH /api/v1/orders/:id/status` chega a `OrderService.changeStatus`.
2. Dentro de `$transaction`, após `tx.order.update` e `tx.orderStatusHistory.create` (linhas 158-167), chama-se `publishWebhookEvent(tx, order, from, to)`.
3. `publishWebhookEvent` busca, **com o mesmo `tx`**, os `WebhookEndpoint` com `customerId = order.customerId` e `active = true`, e filtra os que contêm `to` em `events`.
4. Se não houver nenhum, retorna sem inserir ([09:34] Bruno).
5. Para cada webhook, gera um `event_id` (UUID), monta o payload snapshot (FDD-CONTRATO-08) e faz `tx.webhookOutbox.create({ status: PENDING, nextAttemptAt: now })`.
6. Qualquer erro nesse passo propaga e a transação inteira dá rollback: o status não muda sem evento ([09:40] Bruno).
7. Log `webhook_event_enqueued` (`eventId`, `webhookId`, `orderId`, `toStatus`) após o commit.

### FDD-FLUXO-02 — Processamento pelo worker
Origem: [09:09] Diego (polling 2 s); [09:11] Diego e Larissa (processo separado); [09:12] Diego (ordem por `created_at`).

1. `npm run worker` inicia `src/worker.ts`, que cria o próprio `PrismaClient` ([09:30] Bruno) e o logger, e inicia o loop.
2. Na inicialização, eventos em `PROCESSING` (resto de uma queda anterior) voltam para `PENDING`. Isso pode reenviar um evento, o que é aceitável pela garantia at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
3. A cada `WEBHOOK_POLL_INTERVAL_MS` (2000), o worker seleciona até `WEBHOOK_BATCH_SIZE` eventos com `status = PENDING` e `nextAttemptAt <= now`, ordenados por `createdAt` ([09:08] Diego: "batch pequeno").
4. **Ordem por pedido** (decisão de implementação que preserva a garantia de [09:12] Diego mesmo com retentativas): um evento só é elegível se não houver evento mais antigo **não finalizado** (`PENDING`/`PROCESSING`) para o mesmo `(webhookId, orderId)`.
5. Marca os eventos selecionados como `PROCESSING` (`UPDATE … WHERE status = 'PENDING'`) e processa **sequencialmente**:
   1. carrega o webhook. Se estiver inativo, o evento vai para `FAILED` sem DLQ;
   2. serializa o payload. Se passar de **64 KB**, o evento vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem envio ([09:23] Sofia; [09:24] Diego, Larissa). A checagem é feita no worker, não na transação, para que um payload anômalo nunca bloqueie a mudança de status;
   3. assina e envia (FDD-CONTRATO-08) com `fetch` nativo e `AbortSignal.timeout(10000)`;
   4. grava `webhook_deliveries` (`attempt`, `success`, `responseStatus`, `responseBody` truncado, `durationMs`, `errorCode`);
   5. se der `2xx`, marca `DELIVERED`. Senão, segue o FDD-FLUXO-03.
6. Em SIGINT/SIGTERM, termina o evento corrente, devolve os demais `PROCESSING` do lote para `PENDING` e desconecta o Prisma, no mesmo padrão de `src/server.ts:13-21`.

### FDD-FLUXO-03 — Retry com backoff
Origem: [09:15]–[09:17] Diego e Larissa ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

| Falha nº (`attempts` após incrementar) | Próxima tentativa em |
|---|---|
| 1 (envio inicial falhou) | +1 min |
| 2 | +5 min |
| 3 | +30 min |
| 4 | +2 h |
| 5 | +12 h |
| 6 (5ª retentativa falhou) | → DLQ (FDD-FLUXO-04) |

Ao falhar: `attempts += 1`, `lastError = <código e status>`, `status = PENDING`, `nextAttemptAt = now + BACKOFF[attempts-1]`. O intervalo total entre a primeira falha e a última tentativa é de 14 h 36 min ("quase 15 horas", [09:17] Diego).

### FDD-FLUXO-04 — Dead Letter Queue e replay
Origem: [09:18] Diego; [09:36] Sofia.

1. Ao esgotar as tentativas (ou com `WEBHOOK_PAYLOAD_TOO_LARGE`), numa transação: cria `webhook_dead_letter` (`outboxId`, `webhookId`, `payload`, `failureReason`, `failedAt`) e marca a outbox como `FAILED`. Log `webhook_moved_to_dead_letter`.
2. Replay (FDD-CONTRATO-07), numa transação: valida que o DLQ existe, que não há replay pendente e que o webhook está ativo; marca a outbox como `PENDING`, com `attempts = 0` e `nextAttemptAt = now`; grava `replayedAt` e `replayedById = req.user.id`. Log de auditoria `webhook_dead_letter_replayed` com `userId`, `deadLetterId` e `eventId` ([09:36] Sofia).
3. O worker reprocessa o evento normalmente (FDD-FLUXO-02).

### FDD-FLUXO-05 — Rotação de secret
Origem: [09:21] Sofia.

1. Gera a nova secret com `crypto.randomBytes(32)` em base64url.
2. Move a secret atual para `previousSecret`, com `previousSecretExpiresAt = now + 24h`.
3. O worker assina com as duas secrets enquanto `previousSecretExpiresAt > now` (FDD-CONTRATO-08) e depois só com a nova.

## 7. Matriz de erros

Todas as classes estendem `AppError` (`src/shared/errors/app-error.ts`) e ficam em `src/modules/webhooks/webhook.errors.ts`. Prefixo `WEBHOOK_` conforme [09:28] Bruno e [09:29] Larissa.

| ID | Código | HTTP | Onde | Quando | Ação |
|---|---|---|---|---|---|
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | API | `:id` de webhook inexistente ou removido | — |
| FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | API | URL não `https` ou malformada (regra declarada no schema Zod `webhookUrlSchema`; o service usa `safeParse` para emitir este código) | — |
| FDD-ERR-03 | `WEBHOOK_INVALID_EVENTS` | 400 | API | `events` vazio ou com status fora de `OrderStatus` | — |
| FDD-ERR-04 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` inexistente na criação | — |
| FDD-ERR-05 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API (admin) | `:id` de DLQ inexistente | — |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API (admin) | Evento já recolocado e ainda pendente | — |
| FDD-ERR-07 | `WEBHOOK_INACTIVE` | 409 | API (admin) | Replay para webhook removido ou desativado | — |
| FDD-ERR-08 | `WEBHOOK_SECRET_REQUIRED` | — | Worker | Endpoint sem secret no momento de assinar (inconsistência de dados) | DLQ imediata + log `error` |
| FDD-ERR-09 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Worker | Corpo serializado > 64 KB | DLQ imediata, sem envio |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker | Sem resposta em 10 s | Retry |
| FDD-ERR-11 | `WEBHOOK_DELIVERY_FAILED` | — | Worker | Resposta não-2xx ou erro de rede/TLS | Retry |
| FDD-ERR-12 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Worker | 5ª retentativa falhou | DLQ |

Erros genéricos reaproveitados sem alteração: `VALIDATION_ERROR` (400, `validate()`), `UNAUTHORIZED` (401) e `FORBIDDEN` (403), de `src/middlewares/auth.middleware.ts`. Erros do worker não viram resposta HTTP: vão para `webhook_deliveries.errorCode`, `webhook_dead_letter.failureReason` e logs.

## 8. Estratégias de resiliência

| ID | Estratégia | Detalhe | Origem |
|---|---|---|---|
| FDD-RES-01 | Isolamento transacional | A entrega nunca acontece dentro da transação de pedido | [09:04] Bruno |
| FDD-RES-02 | Timeout | 10 s por requisição (`AbortSignal.timeout`) | [09:42] Diego |
| FDD-RES-03 | Retry com backoff | 1m/5m/30m/2h/12h, persistido em `nextAttemptAt` (sobrevive a restart do worker) | [09:17] Larissa |
| FDD-RES-04 | Fallback | DLQ + replay manual; sem fallback por email nesta fase | [09:18] Diego; [09:37] Larissa |
| FDD-RES-05 | Limite de payload | 64 KB, erro (não truncar) | [09:24] Larissa |
| FDD-RES-06 | Recuperação de queda | `PROCESSING` → `PENDING` na inicialização; idempotência via `X-Event-Id` | [09:24] Diego |
| FDD-RES-07 | Isolamento de processo | Worker separado; restart da API não interrompe entregas | [09:11] Diego |
| FDD-RES-08 | Processamento sequencial | Um cliente lento consome no máximo 10 s por evento do ciclo; outros eventos aguardam o próximo ciclo. Limitação aceita enquanto for single-worker | [09:12]–[09:13] Diego |

## 9. Observabilidade

O projeto só tem logs estruturados com Pino (`src/shared/logger/index.ts`) e correlação por `X-Request-Id` (`src/middlewares/request-logger.middleware.ts`). Não há biblioteca de métricas nem de tracing em `package.json`. Seguindo "não vamos botar nada novo" ([09:29] Bruno):

**Logs (Pino)** — FDD-OBS-01
- API: `webhook_event_enqueued`, `webhook_created`, `webhook_updated`, `webhook_deleted`, `webhook_secret_rotated` e `webhook_dead_letter_replayed` (auditoria com `userId`, [09:36] Sofia).
- Worker: `worker_started`/`worker_stopped`, `webhook_delivery_attempt` (`eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs`), `webhook_delivery_failed` (`errorCode`, `nextAttemptAt`), `webhook_moved_to_dead_letter`.
- Segurança: incluir `'*.secret'`, `'*.previousSecret'` e `'*.signature'` em `redactPaths`. Nunca logar a secret nem o corpo da resposta do cliente além do truncado em `webhook_deliveries`.

**Métricas** — FDD-OBS-02 (derivadas, sem nova dependência)
A cada 30 ciclos (~1 min) o worker calcula e emite o log `webhook_worker_stats` com:
- `outbox_pending_count`;
- `outbox_oldest_pending_age_ms`, o indicador do objetivo de < 10 s ([09:02] Marcos);
- `deliveries_success_rate_5m`;
- `deliveries_p95_duration_ms_5m`;
- `dead_letter_open_count`.

O alerta inicial pode ser feito sobre esses logs. Exportar para Prometheus ou similar fica para quando a plataforma tiver esse stack.

**Tracing** — FDD-OBS-03 (correlação)
- `event_id` é o identificador de correlação ponta a ponta: está no log de enqueue, nos logs de tentativa, em `webhook_deliveries`, na DLQ e no header `X-Event-Id` que o cliente recebe ([09:25] Diego).
- O log `webhook_event_enqueued` inclui o `requestId` da chamada `PATCH /orders/:id/status`, o que liga a requisição HTTP original às entregas. Para isso, `OrderController.changeStatus` repassa `req.id` ao service.
- Tracing distribuído (OpenTelemetry) não existe no projeto e fica fora do escopo.

## 10. Dependências e compatibilidade

| ID | Item | Detalhe |
|---|---|---|
| FDD-DEP-01 | Runtime | Node ≥ 20 (`package.json` `engines`): `fetch`, `AbortSignal.timeout` e `crypto` nativos, **sem dependências npm novas** |
| FDD-DEP-02 | Banco | MySQL 8 (`docker-compose.yml`) + Prisma 5.22.0; migration **aditiva** (4 tabelas, 1 enum, 1 relação em `Customer`) |
| FDD-DEP-03 | Compatibilidade de API | Só rotas novas. A resposta de `PATCH /orders/:id/status` não muda |
| FDD-DEP-04 | Configuração | Novas variáveis opcionais com default em `src/config/env.ts`: `WEBHOOK_POLL_INTERVAL_MS=2000`, `WEBHOOK_HTTP_TIMEOUT_MS=10000`, `WEBHOOK_MAX_PAYLOAD_BYTES=65536`, `WEBHOOK_BATCH_SIZE` (valor inicial a calibrar, ver FDD-OPEN-02) |
| FDD-DEP-05 | Operação | Novo processo `npm run worker` no deploy, ao lado da API |
| FDD-DEP-06 | Pessoas | Revisão de segurança (≥ 2 dias úteis) antes do deploy ([09:46] Sofia); documentação no portal ([09:26] Marcos) |

## 11. Integração com o sistema existente

| ID | Arquivo real | Como o módulo de webhooks se integra |
|---|---|---|
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus` (linhas 126-179), logo após `tx.orderStatusHistory.create` (linhas 159-167), chamar `await publishWebhookEvent(tx, order, from, to)`. O `order` carregado na linha 132 já tem `orderNumber`, `customerId` e `totalCents` para o snapshot. Usa o mesmo tipo `TxClient` (linha 24). Nenhuma mudança no construtor (instanciado em `src/app.ts:43`) |
| FDD-INT-02 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | `webhook.errors.ts` define `WebhookNotFoundError`, `WebhookInvalidUrlError` etc. estendendo `AppError` (status + `errorCode` `WEBHOOK_*`), no mesmo estilo de `InvalidStatusTransitionError` (`http-errors.ts:45`) e `InsufficientStockError` (`http-errors.ts:55`) |
| FDD-INT-03 | `src/middlewares/error.middleware.ts` | **Sem alteração**: o ramo `err instanceof AppError` (linhas 15-24) já serializa os erros do módulo ([09:29] Bruno) |
| FDD-INT-04 | `src/middlewares/auth.middleware.ts` | `webhook.routes.ts` usa `router.use(authenticate)` como `order.routes.ts`. A rota de replay usa `requireRole('ADMIN')` (linha 49), como em `src/modules/users/user.routes.ts:15`. `req.user.id` alimenta `replayedById` |
| FDD-INT-05 | `src/middlewares/validate.middleware.ts` | Schemas Zod em `webhook.schemas.ts` (`createWebhookSchema`, `updateWebhookSchema`, `webhookIdParamSchema`, `listWebhooksQuerySchema`) aplicados com `validate({ body, params, query })` |
| FDD-INT-06 | `src/routes/index.ts` e `src/app.ts` | Adicionar `webhooks` e `adminWebhooks` ao tipo `Controllers` e montar `router.use('/webhooks', …)` e `router.use('/admin/webhooks', …)` em `buildApiRouter`. Instanciar `WebhookRepository`, `WebhookService` e `WebhookController` em `buildControllers` |
| FDD-INT-07 | `src/server.ts` e `src/config/database.ts` | `src/worker.ts` replica o bootstrap e o shutdown de `server.ts` (linhas 6-27), mas cria sua própria instância com `createPrismaClient()` (`database.ts:4`), já que é outro processo ([09:30] Bruno) |
| FDD-INT-08 | `src/config/env.ts` | Acrescentar as variáveis de FDD-DEP-04 ao `envSchema` (linhas 3-10) com `z.coerce.number()` e defaults |
| FDD-INT-09 | `src/shared/logger/index.ts` | Incluir `'*.secret'`, `'*.previousSecret'` e `'*.signature'` em `redactPaths` (linhas 4-11) |
| FDD-INT-10 | `prisma/schema.prisma` | Novos modelos da seção 4. Relação `webhooks WebhookEndpoint[]` em `Customer` (linhas 40-54). `OrderStatus` (linhas 16-23) é o domínio válido de `events` |
| FDD-INT-11 | `package.json` | Scripts `"worker": "node --env-file=.env dist/worker.js"` e `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`, no padrão de `start`/`dev` |
| FDD-INT-12 | `src/shared/http/response.ts` | A listagem usa `paginated()` como os demais módulos |
| FDD-INT-13 | `tests/orders.test.ts`, `tests/helpers/factories.ts` | Novos testes em `tests/webhooks.test.ts` seguindo o padrão existente (supertest + factories) |

Estrutura do módulo ([09:27]–[09:28] Bruno):

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts          # CRUD, rotação, deliveries, replay
├── webhook.repository.ts
├── webhook.routes.ts           # /webhooks e /admin/webhooks
├── webhook.schemas.ts
├── webhook.errors.ts
├── webhook.publisher.ts        # publishWebhookEvent(tx, order, from, to)
├── webhook.signer.ts           # HMAC-SHA256
└── webhook.processor.ts        # loop do worker
src/worker.ts                   # entry point do worker
```

## 12. Critérios de aceite técnicos

| ID | Critério |
|---|---|
| FDD-AC-01 | Dado um webhook ativo que assina `SHIPPED`, quando um pedido do customer vai para `SHIPPED`, então existe exatamente 1 linha `PENDING` em `webhook_outbox` criada na mesma transação |
| FDD-AC-02 | Se o insert na outbox falhar (simulado), a mudança de status sofre rollback e `order_status_history` não ganha linha |
| FDD-AC-03 | Sem webhook assinando o `to_status`, nenhuma linha é inserida |
| FDD-AC-04 | Com o worker rodando e o destino respondendo 200, o evento fica `DELIVERED` em < 10 s após o commit, com o header `X-Signature` verificável com a secret do endpoint |
| FDD-AC-05 | Com o destino respondendo 500, as tentativas seguem 1m/5m/30m/2h/12h e, depois da 5ª retentativa, o evento aparece em `webhook_dead_letter` com a outbox `FAILED` |
| FDD-AC-06 | Um destino que demora > 10 s gera `WEBHOOK_DELIVERY_TIMEOUT` e reagendamento |
| FDD-AC-07 | `POST /webhooks` com `url` `http://…` retorna `400 WEBHOOK_INVALID_URL` |
| FDD-AC-08 | Depois de `rotate-secret`, por 24 h o `X-Signature` contém as assinaturas com a secret nova e a anterior; depois disso, só a nova |
| FDD-AC-09 | O replay por usuário OPERATOR retorna 403; por ADMIN retorna 202, o evento volta a `PENDING` com o mesmo `event_id` e o log registra o `userId` |
| FDD-AC-10 | Payload > 64 KB vai à DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem requisição de saída |
| FDD-AC-11 | `GET /webhooks/:id/deliveries` devolve no máximo 100 itens, do mais recente para o mais antigo |
| FDD-AC-12 | Eventos do mesmo pedido e webhook são entregues na ordem de `created_at`, mesmo quando um deles precisa de retry |

## 13. Riscos e mitigação

| ID | Risco | Mitigação |
|---|---|---|
| FDD-RISK-01 | Crescimento da `webhook_outbox` degrada o polling | Índice `status, nextAttemptAt`; filtro na inserção; arquivamento planejado (FDD-EXC-05) |
| FDD-RISK-02 | Cliente lento atrasa outros eventos (processamento sequencial) | Timeout de 10 s; acompanhar `outbox_oldest_pending_age_ms`; paralelizar por webhook é evolução futura (RFC-OPEN-02) |
| FDD-RISK-03 | Vazamento de secret em logs | `redactPaths` ampliado; secret só nas respostas de criação e rotação; revisão de segurança |
| FDD-RISK-04 | Queda do worker entre envio e marcação gera duplicata | At-least-once + `X-Event-Id` documentados no portal |
| FDD-RISK-05 | Rajadas de eventos para um mesmo cliente | Fora de escopo; monitorar por cliente e decidir depois (FDD-EXC-03) |
| FDD-RISK-06 | Leitura extra de webhooks dentro da transação de status | Índice `customerId, active`; consulta pequena por customer |

## 14. Pontos a validar na revisão técnica (implementação)

| ID | Ponto |
|---|---|
| FDD-OPEN-01 | A criação do pedido (status inicial `PENDING`, em `OrderService.create`) **não** emite evento: a reunião tratou só de mudanças de status em `changeStatus`. Confirmar com Marcos |
| FDD-OPEN-02 | Valor inicial de `WEBHOOK_BATCH_SIZE`: a reunião definiu apenas "batch pequeno" ([09:08] Diego) |
| FDD-OPEN-03 | Formato multi-assinatura do `X-Signature` durante o grace period (seção 5) |
| FDD-OPEN-04 | Como o ADMIN descobre os IDs da DLQ (não há endpoint de listagem acordado) |
