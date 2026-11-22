# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou devolvendo erro ([09:14] Larissa). Já houve cliente com duas horas de indisponibilidade por manutenção planejada ([09:16] Diego). Precisamos insistir o bastante para cobrir indisponibilidades reais, sem deixar eventos pendurados para sempre ([09:15] Diego).

## Decisão

1. **Backoff exponencial com teto de 5 tentativas de reenvio**, com intervalos de **1 min, 5 min, 30 min, 2 h e 12 h** ([09:17] Diego; [09:17] Larissa: "Decidido").
   - Interpretação adotada: o **envio inicial** é seguido de até **5 retentativas**, cada uma agendada pelo intervalo correspondente após a falha anterior. Isso dá 14 h 36 min entre a primeira falha e a última tentativa, o que bate com o "total de quase 15 horas" dito na reunião ([09:17] Diego).
   - Conta como falha: timeout de 10 s ([09:42] Diego), erro de rede ou resposta não-2xx.
2. **Esgotadas as tentativas, o evento vai para a DLQ**, numa **tabela separada `webhook_dead_letter`** com o payload, o motivo da falha e o timestamp ([09:18] Diego).
3. **Reprocessamento manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role **ADMIN** e registra em log quem fez o replay ([09:36] Sofia).

## Alternativas consideradas

1. **Retry indefinido com backoff.** Descartada: o evento pode ficar pendurado para sempre se o cliente sumiu ([09:15] Diego).
2. **3 tentativas (mais agressivo).** Descartada: cobriria só ~30 minutos e mataria eventos de clientes com manutenção de algumas horas ([09:16] Bruno propôs; Diego contra-argumentou).
3. **DLQ como status `failed` na própria outbox.** Descartada: a tabela separada deixa a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento ([09:17] Larissa perguntou; [09:18] Diego).

## Consequências

**Positivas**
- Cobre indisponibilidades de até ~15 h, aceitas pelo produto ([09:17] Marcos: "acho aceitável").
- Falhas permanentes ficam isoladas, auditáveis e reprocessáveis.
- A outbox não acumula eventos mortos.

**Negativas / trade-offs**
- Um evento pode chegar até ~15 h atrasado antes de ir para a DLQ.
- O reprocessamento é manual e depende de um administrador. A notificação automática do cliente sobre falhas foi **adiada** (email fora de escopo, [09:37] Larissa).
- Retentativas aumentam a chance de entrega duplicada, o que é aceito pela garantia at-least-once ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
