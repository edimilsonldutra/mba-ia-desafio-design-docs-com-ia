# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

O OMS já tem padrões consolidados ([09:27] Bruno):

- **Módulos por domínio** em `src/modules/<dominio>/`, cada um com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` (ex.: `src/modules/orders/`, `src/modules/customers/`).
- **Erros de domínio**: classe `AppError` (`src/shared/errors/app-error.ts`) com `statusCode`, `errorCode` e `details`, e subclasses em `src/shared/errors/http-errors.ts` como `InsufficientStockError` (`INSUFFICIENT_STOCK`) e `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) ([09:28] Bruno).
- **Middleware de erro central** (`src/middlewares/error.middleware.ts`), que já trata `AppError`, `ZodError` e erros conhecidos do Prisma ([09:29] Bruno).
- **Logger Pino** (`src/shared/logger/index.ts`) usado no projeto inteiro ([09:29] Bruno).
- **Validação com Zod** via `validate()` (`src/middlewares/validate.middleware.ts`).
- **Autorização** com `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts:27, 49`) ([09:36] Larissa).
- **Composição** em `buildControllers` (`src/app.ts:26`) e roteamento em `buildApiRouter` (`src/routes/index.ts:21`), sob o prefixo `/api/v1` (`src/app.ts:67`).

## Decisão

"Reuso máximo do que já existe" ([09:30] Larissa):

1. Novo módulo **`src/modules/webhooks/`** com a mesma estrutura dos demais. O processamento fica em `webhook.processor.ts` e o entry point do worker em `src/worker.ts` ([09:27]–[09:28] Bruno).
2. **Erros do módulo estendem `AppError`** com códigos de prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno; [09:29] Larissa). O middleware de erro os trata sem alteração ([09:29] Bruno).
3. **Pino** para todos os logs, sem biblioteca nova ([09:29] Bruno).
4. **Schemas Zod** para entrada, incluindo a exigência de `https` ([09:23] Sofia).
5. **`requireRole('ADMIN')`** no endpoint de replay da DLQ. Os demais endpoints usam só `authenticate` ([09:36] Larissa; [09:37] Sofia).
6. A integração com pedidos é feita por uma **função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o client da transação corrente. Assim não é preciso injetar um repository inteiro no `OrderService` ([09:41] Bruno, Diego).

## Alternativas consideradas

1. **Criar infraestrutura própria para o módulo** (outro logger, outro formato de erro, outro handler). Descartada: duplicaria padrões e fragmentaria a observabilidade ([09:29] Bruno: "Não vamos botar nada novo").
2. **Injetar `WebhookRepository` no construtor de `OrderService`.** Preterida em favor da função que recebe `tx` ([09:41] Diego: "Não precisa injetar repository inteiro"). Com ela a inserção participa da transação existente sem mudar a assinatura do construtor usado em `src/app.ts:43`.

## Consequências

**Positivas**
- Curva de aprendizado mínima para o time. Revisão e testes seguem as convenções atuais (`tests/orders.test.ts` como referência).
- Respostas de erro no mesmo formato `{ error: { code, message, details } }` da API inteira.
- A transação de `changeStatus` continua sendo a única fonte de atomicidade.

**Negativas / trade-offs**
- O módulo herda as limitações atuais: só logs, sem métricas nem tracing distribuído nativos (ver Observabilidade no FDD).
- `OrderService` passa a depender de uma função do módulo de webhooks, um acoplamento explícito entre os domínios de pedidos e notificações.
