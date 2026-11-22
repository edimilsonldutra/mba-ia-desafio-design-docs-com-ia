# PRD — Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) — API |
| **PM** | Marcos |
| **Tech Lead** | Larissa |
| **Envolvidos** | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Status** | Aprovado para desenvolvimento |
| **Origem** | Reunião técnica de quinta-feira, 09:00 (`TRANSCRICAO.md`) |
| **Relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS vai notificar clientes B2B, em tempo quase real, toda vez que o status de um pedido deles mudar. A notificação usa **webhooks de saída**: o cliente cadastra uma URL HTTPS, escolhe os status que quer receber e passa a receber chamadas HTTP assinadas, sem precisar consultar a API repetidamente (PRD-CTX-01). Hoje o OMS não tem nenhum mecanismo de notificação externa (PRD-CTX-04).

## 2. Problema e motivação

- **Pedido formal de clientes:** Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real de mudanças de status (PRD-CTX-01).
- **Polling caro:** sem notificação, os clientes consultam `GET /orders` periodicamente, o que deixa a integração lenta e cara para eles (PRD-CTX-02).
- **Risco comercial:** a Atlas indicou que pode migrar para um concorrente se não houver entrega até o fim do trimestre (PRD-CTX-03).

## 3. Público-alvo e cenários de uso

| ID | Persona | Cenário |
|---|---|---|
| PRD-PERS-01 | **Integrador do cliente B2B** (ex.: Atlas) | Cadastra um webhook com os status de interesse (ex.: `SHIPPED`, `DELIVERED`) e atualiza o próprio sistema logístico assim que recebe a notificação, sem polling |
| PRD-PERS-02 | **Integrador do cliente B2B** | Valida a assinatura de cada chamada e descarta duplicatas pelo `X-Event-Id` |
| PRD-PERS-03 | **Integrador do cliente B2B** | Consulta o histórico das últimas entregas para investigar uma falha do lado dele |
| PRD-PERS-04 | **Administrador do OMS** (role ADMIN) | Reprocessa manualmente eventos que esgotaram as tentativas depois que o cliente corrige o endpoint |
| PRD-PERS-05 | **Operador/usuário do OMS que representa o cliente** | Cadastra webhooks pela API com o JWT do OMS, informando o customer |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
|---|---|---|---|
| PRD-OBJ-01 | Notificar em "tempo real" | Tempo entre a mudança de status e a primeira tentativa de entrega | **< 10 s em 95% dos eventos** (definição de "tempo real" dos clientes; o polling de 2 s dá folga) |
| PRD-OBJ-02 | Atender os clientes que pediram a feature | Clientes B2B consumindo webhooks em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o **fim de novembro** |
| PRD-OBJ-03 | Não perder notificações | Mudanças de status com webhook assinante e sem evento gerado | **0** (garantia transacional) |
| PRD-OBJ-04 | Entregar no prazo | Duração do desenvolvimento, incluindo a revisão de segurança | **3 sprints** |

## 5. Escopo

### 5.1 Incluso
- Cadastro, listagem, edição e remoção de webhooks por customer.
- Seleção dos status de interesse por webhook.
- Notificação a cada mudança de status de pedido, com assinatura HMAC.
- Retentativas automáticas e fila de eventos com falha (DLQ), com reprocessamento manual por ADMIN.
- Rotação de secret pelo cliente.
- Histórico das últimas entregas por webhook.

### 5.2 Fora de escopo

| ID | Item | Situação |
|---|---|---|
| PRD-OUT-01 | **Email ao cliente** quando o webhook falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta fase; é projeto do time de frontend |
| PRD-OUT-03 | **Rate limiting de saída** por cliente | **Adiado**: "observar e decidir depois" |
| PRD-OUT-04 | **Webhooks de entrada** (cliente enviando para nós) | **Descartado**: só saída |
| PRD-OUT-05 | **Arquivamento** de eventos entregues (~30 dias) | Fora do escopo desta feature |
| PRD-OUT-06 | **Garantia de ordem global** e múltiplos workers | Limitação conhecida; ordem garantida só por pedido |

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| PRD-FR-01 | Cadastrar um webhook informando URL e lista de status de interesse. A plataforma gera a secret e a devolve na criação |
| PRD-FR-02 | Informar o customer dono do webhook no body ou no path da requisição. Ele **não** é derivado do JWT, que é do usuário operador |
| PRD-FR-03 | Editar um webhook existente (PATCH) |
| PRD-FR-04 | Remover um webhook (DELETE) |
| PRD-FR-05 | Listar os webhooks de um customer (GET) |
| PRD-FR-06 | Filtrar eventos por status: o webhook só recebe as mudanças para os status que escolheu |
| PRD-FR-07 | Enviar uma notificação a cada mudança de status de pedido do customer, sem esquecer nenhuma mudança efetivada |
| PRD-FR-08 | Consultar o histórico das últimas 100 entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta) |
| PRD-FR-09 | Rotacionar a secret pela API, com a anterior válida por mais 24 h |
| PRD-FR-10 | Retentar automaticamente entregas com falha e, esgotadas as tentativas, guardá-las numa DLQ |
| PRD-FR-11 | Um ADMIN reprocessa manualmente um evento da DLQ, com registro de quem fez |
| PRD-FR-12 | Cada notificação é assinada (HMAC-SHA256) para que o cliente valide origem e integridade |
| PRD-FR-13 | Cada notificação traz um identificador único de evento (`X-Event-Id`) para deduplicação |
| PRD-FR-14 | O payload traz identificação do evento, tipo, timestamp, pedido (id e número), status anterior e novo, customer e total. Os itens ficam de fora |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| PRD-NFR-01 | Latência: primeira tentativa em < 10 s (polling de 2 s) |
| PRD-NFR-02 | Segurança: a URL precisa ser HTTPS; `http` é recusado com erro de validação |
| PRD-NFR-03 | Segurança: secret única por endpoint |
| PRD-NFR-04 | Limite de payload de 64 KB; acima disso, erro (sem truncar) |
| PRD-NFR-05 | Timeout de 10 s por chamada ao cliente |
| PRD-NFR-06 | Entrega **at-least-once**: o cliente pode receber duplicatas |
| PRD-NFR-07 | Ordem de entrega garantida por pedido enquanto houver um único worker |
| PRD-NFR-08 | Atomicidade: o evento é registrado na mesma transação da mudança de status |
| PRD-NFR-09 | Disponibilidade: o processamento roda em processo separado da API |
| PRD-NFR-10 | Auditoria: o replay da DLQ registra o usuário que o executou |
| PRD-NFR-11 | Retentativas cobrem indisponibilidades do cliente de até ~15 h |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | Detalhe |
|---|---|---|
| Outbox no MySQL existente em vez de fila dedicada | Latência mínima de ~2 s; tabela cresce | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2 s | Mais um processo para operar | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas (até ~15 h) e DLQ | Evento pode chegar até ~15 h atrasado; replay é manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC por endpoint com rotação | Secret precisa ser armazenada de forma recuperável | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once em vez de exactly-once | O cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Só logs; sem métricas nativas | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Snapshot e filtro na inserção | Sem eventos retroativos para webhooks novos | [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

| ID | Dependência |
|---|---|
| PRD-DEP-01 | Revisão de segurança da Sofia (≥ 2 dias úteis) antes do deploy, focada em HMAC e geração de secret |
| PRD-DEP-02 | Documentação para integradores no portal de desenvolvedor, com destaque para at-least-once e deduplicação |
| PRD-DEP-03 | Banco MySQL e Prisma existentes (`prisma/schema.prisma`); nenhuma infraestrutura nova |
| PRD-DEP-04 | Usuários do OMS que representam o cliente, autenticados com o JWT atual (`src/middlewares/auth.middleware.ts`) |
| PRD-DEP-05 | Comunicação de prazo com os clientes pelo PM |

## 10. Riscos e mitigação

Riscos levantados na reunião. A probabilidade e o impacto (Alta/Média/Baixa) são uma avaliação deste PRD, a confirmar com o time.

| ID | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| PRD-RISK-01 | Não entregar até o fim de novembro e perder a Atlas | Média | Alto | Escopo enxuto (sem email, dashboard e rate limit); estimativa de 3 sprints com a revisão de segurança incluída |
| PRD-RISK-02 | Endpoint do cliente fora do ar por horas causa perda de notificações | Alta | Médio | Retry por até ~15 h, DLQ e replay manual |
| PRD-RISK-03 | Vazamento da secret de um cliente (já aconteceu antes) | Média | Alto | Secret por endpoint e rotação com convivência de 24 h |
| PRD-RISK-04 | Cliente processa um evento duplicado | Média | Médio | `X-Event-Id` e documentação em destaque no portal |
| PRD-RISK-05 | Volume alto de mudanças "bombardeia" o cliente | Baixa | Médio | Monitorar após o lançamento e decidir sobre rate limiting (PRD-OUT-03) |
| PRD-RISK-06 | Crescimento futuro exige múltiplos workers e quebra a ordem | Baixa | Médio | Limitação documentada; particionar por pedido quando necessário |

## 11. Critérios de aceitação

| ID | Critério |
|---|---|
| PRD-AC-01 | Dado um webhook cadastrado para `SHIPPED`, quando um pedido do customer muda para `SHIPPED`, o cliente recebe a notificação em menos de 10 s |
| PRD-AC-02 | Uma mudança para um status que o webhook não assina não gera chamada |
| PRD-AC-03 | Cadastrar um webhook com URL `http://` retorna erro de validação |
| PRD-AC-04 | A resposta de criação traz a secret; listagens e edições não a expõem |
| PRD-AC-05 | Com o endpoint do cliente fora do ar, a plataforma retenta 5 vezes (1m, 5m, 30m, 2h, 12h) e depois move o evento para a DLQ |
| PRD-AC-06 | Só um ADMIN consegue reprocessar um evento da DLQ, e o log registra quem fez |
| PRD-AC-07 | Depois da rotação, a secret anterior continua aceita por 24 h |
| PRD-AC-08 | O histórico de entregas mostra até as últimas 100 tentativas com status, resposta e tempo |
| PRD-AC-09 | Cada notificação carrega `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` |

## 12. Estratégia de testes e validação

| ID | Estratégia |
|---|---|
| PRD-TEST-01 | Testes automatizados de integração no padrão atual (`tests/orders.test.ts`, supertest) cobrindo CRUD, filtro e inserção atômica na outbox |
| PRD-TEST-02 | Testes do worker com um servidor HTTP de teste simulando sucesso, 500 e timeout, verificando o backoff, a DLQ e a assinatura |
| PRD-TEST-03 | Revisão de segurança dedicada (HMAC, geração e armazenamento de secret) antes do deploy |
| PRD-TEST-04 | Testes ponta a ponta da integração com `order.service` (mudança de status → outbox → entrega), medindo a latência do PRD-OBJ-01 |
