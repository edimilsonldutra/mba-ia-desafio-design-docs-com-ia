# ADR-005 — Garantia de entrega at-least-once com X-Event-Id

- **Status:** Aceito
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Contexto

Com outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e retentativas ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), um mesmo evento pode ser enviado mais de uma vez. Isso acontece, por exemplo, quando o cliente processa a requisição mas a resposta não chega antes do timeout, ou quando o worker cai entre o envio e a marcação como entregue. Precisamos definir que garantia oferecemos e como o cliente lida com duplicatas ([09:24]–[09:25] Diego, Bruno).

## Decisão

1. **Garantia at-least-once:** o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado ([09:24] Diego).
2. **Cada evento recebe um UUID (`event_id`)** gerado quando entra na outbox. Ele é único por evento e é enviado no header **`X-Event-Id`** e no campo `event_id` do payload. O cliente deduplica por esse valor ([09:25] Diego).
3. Reenvios (retry automático ou replay da DLQ) **preservam o mesmo `event_id`**, para que a deduplicação funcione.
4. O comportamento é **documentado em destaque no portal de desenvolvedor** ([09:26] Marcos).

## Alternativas consideradas

1. **Exactly-once.** Descartada: exigiria coordenação dos dois lados e "fica muito mais complexo". At-least-once com `event_id` "resolve 99% dos casos" e é o que Stripe e GitHub fazem ([09:25] Diego).
2. **At-most-once (sem retentativa).** Incompatível com a decisão de retentar por até ~15 h ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) e com o objetivo de não perder notificações.

## Consequências

**Positivas**
- Nenhum evento commitado se perde silenciosamente.
- Modelo simples do nosso lado e familiar para os clientes, por ser padrão de mercado.

**Negativas / trade-offs**
- Transfere para o cliente a responsabilidade de deduplicar ([09:25] Sofia: "Isso joga responsabilidade pro cliente").
- Exige comunicação clara no portal; clientes que ignorarem o `X-Event-Id` podem processar eventos em dobro.
