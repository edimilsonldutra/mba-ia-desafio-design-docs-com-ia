# ADR-002 — Worker em processo separado, com polling de 2 segundos

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), algo precisa ler os eventos pendentes e entregá-los. Os clientes consideram "tempo real" qualquer entrega abaixo de 10 segundos ([09:02] Marcos). A aplicação tem hoje um único entry point HTTP, `src/server.ts`, que monta o Express e o `PrismaClient` de `src/config/database.ts`.

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos (ordem de `created_at`), processa e marca o resultado ([09:09] Diego; [09:10] Larissa: "Worker em polling, 2s").
2. **Processo separado da API**, com um entry point novo `src/worker.ts` e o script `npm run worker` ([09:11] Diego, Larissa). Ele usa o mesmo banco (`DATABASE_URL`), mas **uma instância própria de `PrismaClient`**, porque "PrismaClient é por processo" ([09:30] Bruno). A instância é criada com a função `createPrismaClient()` já existente em `src/config/database.ts`.
3. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts`. `src/worker.ts` só faz o bootstrap ([09:28] Bruno).
4. **Um único worker (single-worker).** A ordem de entrega é garantida **por `order_id` e só enquanto houver um único worker**. Não existe garantia de ordem global ([09:12] Diego; [09:13] Larissa).

## Alternativas consideradas

1. **Trigger do banco / notificação reativa (estilo `LISTEN/NOTIFY` do Postgres).** Descartada: o MySQL não tem listener nativo; um trigger "só executa SQL" e não notifica processo externo. Avisar o worker exigiria improvisos como arquivo ou endpoint ([09:09] Diego, em resposta a Bruno).
2. **Rodar o worker dentro do processo da API.** Descartada: se a API reinicia, o worker cai junto ([09:11] Diego: "só não pode ser o mesmo processo").
3. **Vários workers em paralelo desde já.** Adiada: perderia a garantia de ordem. Escalar exigirá particionar por `order_id` ou usar lock pessimista, "problema do futuro" ([09:13] Diego).

## Consequências

**Positivas**
- 2 s de polling ficam bem abaixo do teto de 10 s dos clientes ([09:09] Diego; [09:10] Marcos).
- Ciclos de vida independentes: deploy ou restart da API não interrompem entregas.
- Reaproveita stack, banco e configuração (`src/config/env.ts`) da API.

**Negativas / trade-offs**
- Latência mínima de ~2 s, aceita explicitamente ([09:10] Larissa).
- Consultas periódicas ao banco mesmo sem eventos (custo baixo, com índice em `status`/`created_at`, ver [ADR-001](ADR-001-outbox-no-mysql.md)).
- Um único worker limita a vazão e é ponto único de falha. A ordem por pedido é **limitação conhecida** documentada ([09:13] Larissa).
- Mais um processo para operar e monitorar.
