# ADR-007 — Payload como snapshot e filtro de eventos na inserção da outbox

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

Cada webhook escolhe quais status quer receber, por exemplo "só SHIPPED e DELIVERED" ([09:33] Marcos). Também é preciso decidir **quando** montar o payload: no momento da mudança de status ou no momento do envio ([09:51] Bruno). O payload é enxuto e não leva os itens do pedido ([09:43] Diego).

## Decisão

1. **Filtrar na inserção:** ao mudar o status, só é gravado evento na outbox para os webhooks ativos do customer que assinam o `to_status`. Se nenhum assina, **nada é inserido** ([09:34] Bruno, Diego: "Concordo").
2. **Payload renderizado na inserção (snapshot):** a linha da outbox guarda o JSON final. Se o pedido mudar depois, o evento continua refletindo o estado do momento da mudança ([09:52] Larissa, Diego, Bruno: "snapshot. Decidido").
3. O payload contém `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, **sem itens**. O cliente consulta `GET /orders/:id` se precisar de detalhes ([09:43] Diego).

## Alternativas consideradas

1. **Filtrar no momento do envio.** Descartada: grava linhas desnecessárias na outbox ([09:34] Bruno: "Economiza linha na tabela").
2. **Guardar só `order_id` e renderizar no envio.** Descartada: o evento poderia refletir um estado posterior do pedido, o "caso esquisito" apontado em [09:52] Larissa.
3. **Incluir os itens do pedido no payload.** Descartada para não inflar o payload ([09:43] Diego).

## Consequências

**Positivas**
- A outbox só contém trabalho real.
- O evento é fiel ao momento da transição, inclusive em retentativas horas depois ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)).
- O payload pequeno fica longe do limite de 64 KB ([09:24] Diego).

**Negativas / trade-offs**
- Um webhook criado ou editado **depois** da mudança de status não recebe eventos retroativos.
- Mudar o filtro de um webhook não afeta eventos já enfileirados.
- A inserção em `changeStatus` passa a fazer uma leitura extra (webhooks ativos do customer) dentro da transação.
