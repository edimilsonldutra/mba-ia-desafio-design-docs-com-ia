# ADR-001 — Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-e-filtro-na-insercao.md)

## Contexto

Três clientes B2B querem ser notificados quando o status dos seus pedidos muda ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`). Esse método já roda uma transação Prisma (`this.prisma.$transaction`, linha 131) que valida a transição, debita ou repõe estoque, atualiza `orders` e insere em `order_status_history` ([09:04] Bruno).

Precisamos de um mecanismo que:
- não acrescente uma chamada HTTP a um sistema externo dentro dessa transação, porque um cliente lento travaria as mudanças de status de outros pedidos ([09:04] Bruno);
- não perca eventos nem os emita para mudanças que deram rollback ([09:06] Diego);
- não exija infraestrutura nova, já que o time é pequeno ([09:07] Diego).

## Decisão

Adotar o **padrão Transactional Outbox no MySQL já usado pela aplicação** ([09:08] Larissa).

- Dentro da mesma transação de `changeStatus`, inserir o evento numa tabela `webhook_outbox` ([09:06] Diego). Se a inserção falhar, a transação inteira dá rollback ([09:40] Bruno).
- Um worker separado lê a outbox e faz as chamadas HTTP (detalhado no [ADR-002](ADR-002-worker-separado-em-polling.md)).
- A outbox tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at`. O worker lê só os pendentes, em lotes pequenos ([09:08] Diego).
- Identificadores em UUID, seguindo o padrão do projeto ([09:51] Larissa; `prisma/schema.prisma` usa `@id @default(uuid()) @db.Char(36)` em todos os modelos).

## Alternativas consideradas

1. **Disparar o webhook de forma síncrona dentro de `changeStatus`.** Descartada: a transação já é pesada e uma chamada HTTP no meio travaria mudanças de status de outros pedidos. Se o cliente estivesse fora do ar, não faria sentido dar rollback na mudança de status ([09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão").
2. **Redis Streams (ou fila equivalente).** Descartada: exigiria subir infraestrutura nova; "subir Redis Cluster pra isso é overengineering" para um time pequeno ([09:07] Larissa, Diego). Ainda manteria o problema de *dual write* entre banco e fila.

## Consequências

**Positivas**
- Atomicidade: se a transação commitou, o evento existe; se deu rollback, o evento some junto ([09:06] Diego).
- Nenhuma peça nova de infraestrutura: mesmo MySQL e mesmo Prisma.
- A mudança de status deixa de depender da disponibilidade dos clientes.

**Negativas / trade-offs**
- A entrega deixa de ser imediata e passa a depender do intervalo de polling do worker. A latência mínima fica em ~2 s ([09:10] Larissa).
- A tabela cresce continuamente. O arquivamento de linhas entregues (~30 dias) fica **fora do escopo** desta feature ([09:08] Diego) e precisa de uma ação futura.
- O banco transacional ganha uma escrita a mais por mudança de status (mitigada pelo filtro na inserção, [ADR-007](ADR-007-snapshot-e-filtro-na-insercao.md)).
