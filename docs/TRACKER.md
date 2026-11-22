# Tracker de Rastreabilidade

Liga cada item registrado nos documentos à sua origem: a transcrição da reunião (`TRANSCRICAO.md`) ou o código-fonte do repositório.

- **ID**: o identificador usado no próprio documento (`PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`…). Os ADRs usam o número do ADR; linhas complementares de um mesmo ADR têm sufixo (`ADR-001-ALT`).
- **Fonte/Localização**: `TRANSCRICAO` → `[hh:mm] Falante` da fala principal; `CODIGO` → caminho real do arquivo.
- Itens marcados no FDD como *decisão de implementação* apontam para a fala que originou o requisito que eles detalham.

**Resumo:** 214 linhas · 187 de TRANSCRICAO (87%) · 27 de CODIGO · cobertura de 203/203 itens identificáveis (100%).

Validação automática na geração: toda localização `[hh:mm] Falante` foi conferida contra uma fala real da transcrição e todo caminho de código foi conferido no repositório.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox no MySQL na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Streams descartado por exigir infra nova | TRANSCRICAO | [09:07] Diego |
| ADR-001-COD | `docs/adrs/ADR-001-outbox-no-mysql.md` | Contexto (código) | Transação de changeStatus como ponto de inserção | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker separado em polling de 2 s, single-worker | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PROC | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo próprio (src/worker.ts, npm run worker) | TRANSCRICAO | [09:11] Larissa |
| ADR-002-ALT | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Trigger/LISTEN descartado (MySQL não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | 5 retentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DLQ | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | 3 tentativas descartado (cobre só ~30 min) | TRANSCRICAO | [09:16] Diego |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa descartada | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | At-least-once com X-Event-Id para deduplicação | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Alternativa descartada | Exactly-once descartado pela complexidade | TRANSCRICAO | [09:25] Diego |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-COD | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Contexto (código) | AppError e subclasses com errorCode | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-FN | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | publishWebhookEvent(tx, …) em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-007 | `docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md` | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-FILTRO | `docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md` | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Atlas, MaxDistribuição e Nova Cargo pedem notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Contexto | Clientes fazem polling em GET /orders (lento e caro) | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | `docs/PRD.md` | Contexto | Atlas pode migrar para concorrente se não houver entrega no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | `docs/PRD.md` | Contexto (código) | OMS sem mecanismo de eventos/filas/notificação | CODIGO | `package.json` |
| PRD-PERS-01 | `docs/PRD.md` | Cenário de uso | Integrador recebe só os status de interesse | TRANSCRICAO | [09:33] Marcos |
| PRD-PERS-02 | `docs/PRD.md` | Cenário de uso | Integrador valida assinatura e deduplica por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-PERS-03 | `docs/PRD.md` | Cenário de uso | Integrador consulta histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-PERS-04 | `docs/PRD.md` | Cenário de uso | ADMIN reprocessa DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-PERS-05 | `docs/PRD.md` | Cenário de uso | Usuários do OMS que representam o cliente usam a API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-OBJ-01 | `docs/PRD.md` | Métrica | Primeira tentativa < 10 s (definição de tempo real) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Métrica | 3 clientes integrados até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Métrica | Zero mudanças de status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | `docs/PRD.md` | Métrica | Entrega em 3 sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-OUT-01 | `docs/PRD.md` | Fora de escopo | Email ao cliente em falhas (adiado) | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual (projeto do frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída (observar) | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | `docs/PRD.md` | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | `docs/PRD.md` | Fora de escopo | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | `docs/PRD.md` | Fora de escopo | Ordem global / múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook (url, status, secret gerada e devolvida) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos por status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Notificar a cada mudança de status efetivada | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Retry automático e DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Replay manual da DLQ por ADMIN, com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Notificação assinada com HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | X-Event-Id único por evento | TRANSCRICAO | [09:25] Diego |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Campos do payload, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência < 10 s com polling de 2 s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | URL HTTPS obrigatória | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Payload máximo de 64 KB, erro acima disso | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Ordem por pedido com single-worker | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Evento na mesma transação da mudança | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Retentativas cobrem ~15 h de indisponibilidade | TRANSCRICAO | [09:17] Diego |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Documentação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | `docs/PRD.md` | Dependência (código) | MySQL e Prisma existentes | CODIGO | `prisma/schema.prisma` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência (código) | JWT atual de usuário operador | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | PM confirma prazo com os clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Perder a Atlas por prazo | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Cliente fora do ar por horas | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Evento duplicado no cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-05 | `docs/PRD.md` | Risco | Volume alto bombardeia o cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | `docs/PRD.md` | Risco | Escalar workers quebra a ordem | TRANSCRICAO | [09:13] Diego |
| PRD-AC-01 | `docs/PRD.md` | Critério de Aceitação | Notificação SHIPPED em < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | `docs/PRD.md` | Critério de Aceitação | Status não assinado não gera chamada | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | `docs/PRD.md` | Critério de Aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-04 | `docs/PRD.md` | Critério de Aceitação | Secret só na resposta de criação | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-05 | `docs/PRD.md` | Critério de Aceitação | 5 retentativas e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-06 | `docs/PRD.md` | Critério de Aceitação | Replay só por ADMIN, com log | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-07 | `docs/PRD.md` | Critério de Aceitação | Secret anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-08 | `docs/PRD.md` | Critério de Aceitação | Histórico de até 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | `docs/PRD.md` | Critério de Aceitação | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de Teste (código) | Testes de integração no padrão existente | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de Teste | Testes do worker: sucesso, 500, timeout, backoff, DLQ | TRANSCRICAO | [09:42] Diego |
| PRD-TEST-03 | `docs/PRD.md` | Estratégia de Teste | Revisão de segurança de HMAC e secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-04 | `docs/PRD.md` | Estratégia de Teste | Testes ponta a ponta da integração com order.service | TRANSCRICAO | [09:46] Larissa |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Pedido formal de três clientes B2B | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | Polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Contexto | Risco de migração da Atlas | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-04 | `docs/RFC.md` | Restrição | Tempo real = abaixo de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-05 | `docs/RFC.md` | Contexto (código) | changeStatus em transação com orders, history e estoque | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox no MySQL | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker separado, polling 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | Backoff e DLQ com replay ADMIN | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | HMAC, secret por endpoint, rotação, HTTPS | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Módulo src/modules/webhooks nos padrões atuais | TRANSCRICAO | [09:27] Bruno |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | Filtro e snapshot na inserção | TRANSCRICAO | [09:52] Bruno |
| RFC-API-01 | `docs/RFC.md` | Requisito Funcional | CRUD, rotação e deliveries sob /webhooks | TRANSCRICAO | [09:33] Bruno |
| RFC-API-02 | `docs/RFC.md` | Restrição | CRUD exige só autenticação | TRANSCRICAO | [09:37] Sofia |
| RFC-API-03 | `docs/RFC.md` | Restrição | Replay exige ADMIN | TRANSCRICAO | [09:36] Larissa |
| RFC-API-04 | `docs/RFC.md` | Restrição | customer_id não vem do JWT | TRANSCRICAO | [09:32] Bruno |
| RFC-ALT-01 | `docs/RFC.md` | Trade-off | Síncrono descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Trade-off | Redis Streams descartado | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | `docs/RFC.md` | Trade-off | Trigger/LISTEN descartado | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Trade-off | Retry indefinido e 3 tentativas descartados | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Trade-off | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | `docs/RFC.md` | Trade-off | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Múltiplos workers e ordem | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Email em falhas (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Alteração crítica em changeStatus com rollback | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | `docs/RFC.md` | Impacto | Entry point src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| RFC-IMP-03 | `docs/RFC.md` | Impacto | 3 sprints com revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-RISK-01 | `docs/RFC.md` | Risco | Evento perdido/fantasma mitigado pela transação | TRANSCRICAO | [09:06] Diego |
| RFC-RISK-02 | `docs/RFC.md` | Risco | Indisponibilidade longa do cliente | TRANSCRICAO | [09:16] Diego |
| RFC-RISK-03 | `docs/RFC.md` | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-04 | `docs/RFC.md` | Risco | Entrega duplicada | TRANSCRICAO | [09:24] Diego |
| RFC-RISK-05 | `docs/RFC.md` | Risco | Perda de ordem ao escalar | TRANSCRICAO | [09:13] Larissa |
| FDD-OBJ-01 | `docs/FDD.md` | Objetivo técnico | Evento na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | `docs/FDD.md` | Objetivo técnico | Primeira tentativa < 10 s | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-03 | `docs/FDD.md` | Objetivo técnico | Assinatura HMAC por endpoint | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-04 | `docs/FDD.md` | Objetivo técnico | Backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-05 | `docs/FDD.md` | Objetivo técnico | Sem infra/biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| FDD-EXC-01 | `docs/FDD.md` | Exclusão | Inbound webhooks | TRANSCRICAO | [09:02] Marcos |
| FDD-EXC-02 | `docs/FDD.md` | Exclusão | Email em falhas | TRANSCRICAO | [09:37] Larissa |
| FDD-EXC-03 | `docs/FDD.md` | Exclusão | Rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-EXC-04 | `docs/FDD.md` | Exclusão | Dashboard | TRANSCRICAO | [09:40] Larissa |
| FDD-EXC-05 | `docs/FDD.md` | Exclusão | Arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-EXC-06 | `docs/FDD.md` | Exclusão | Múltiplos workers | TRANSCRICAO | [09:13] Diego |
| FDD-EXC-07 | `docs/FDD.md` | Exclusão | Itens no payload | TRANSCRICAO | [09:43] Diego |
| FDD-DADOS-01 | `docs/FDD.md` | Modelo de dados | webhook_endpoints: url, secret, customer, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | `docs/FDD.md` | Modelo de dados | webhook_outbox com índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | `docs/FDD.md` | Modelo de dados | webhook_deliveries para o histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | `docs/FDD.md` | Modelo de dados | webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /webhooks?customerId= | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries (100 últimas) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Requisição de saída: headers e payload | TRANSCRICAO | [09:44] Diego |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox dentro de changeStatus | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Loop do worker em ordem de created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Tabela de backoff | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | DLQ e replay com auditoria | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-01 | `docs/FDD.md` | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_URL (https) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_EVENTS | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-04 | `docs/FDD.md` | Erro | WEBHOOK_CUSTOMER_NOT_FOUND | TRANSCRICAO | [09:32] Larissa |
| FDD-ERR-05 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-06 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-07 | `docs/FDD.md` | Erro | WEBHOOK_INACTIVE | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-08 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-09 | `docs/FDD.md` | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (64 KB) | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-10 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_TIMEOUT (10 s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-11 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_FAILED | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-12 | `docs/FDD.md` | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED | TRANSCRICAO | [09:15] Diego |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Entrega fora da transação | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Backoff persistido | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-04 | `docs/FDD.md` | Resiliência | Fallback = DLQ + replay | TRANSCRICAO | [09:18] Diego |
| FDD-RES-05 | `docs/FDD.md` | Resiliência | Limite de 64 KB com erro | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-06 | `docs/FDD.md` | Resiliência | Recuperação de queda com idempotência | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | `docs/FDD.md` | Resiliência | Isolamento de processo | TRANSCRICAO | [09:11] Diego |
| FDD-RES-08 | `docs/FDD.md` | Resiliência | Processamento sequencial single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs Pino, incluindo auditoria de replay | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Métricas derivadas, idade do pendente mais antigo vs 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade (código) | Correlação por event_id e X-Request-Id | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência (código) | Node ≥ 20: fetch e crypto nativos | CODIGO | `package.json` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência (código) | MySQL 8 + Prisma, migration aditiva | CODIGO | `docker-compose.yml` |
| FDD-DEP-03 | `docs/FDD.md` | Compatibilidade (código) | Só rotas novas; PATCH /orders/:id/status inalterado | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-DEP-04 | `docs/FDD.md` | Dependência (código) | Novas variáveis em env.ts | CODIGO | `src/config/env.ts` |
| FDD-DEP-05 | `docs/FDD.md` | Dependência | Novo processo npm run worker no deploy | TRANSCRICAO | [09:11] Larissa |
| FDD-DEP-06 | `docs/FDD.md` | Dependência | Revisão de segurança e portal | TRANSCRICAO | [09:46] Sofia |
| FDD-INT-01 | `docs/FDD.md` | Integração (código) | publishWebhookEvent em changeStatus | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração (código) | Erros WEBHOOK_* estendendo AppError | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração (código) | errorMiddleware sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração (código) | authenticate e requireRole('ADMIN') | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração (código) | Schemas Zod com validate() | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração (código) | Registrar rotas em buildApiRouter e buildControllers | CODIGO | `src/routes/index.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração (código) | src/worker.ts no padrão de server.ts com createPrismaClient | CODIGO | `src/server.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração (código) | Variáveis no envSchema | CODIGO | `src/config/env.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração (código) | Redact de secret no logger | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração (código) | Novos modelos e relação em Customer | CODIGO | `prisma/schema.prisma` |
| FDD-INT-11 | `docs/FDD.md` | Integração (código) | Scripts worker e worker:dev | CODIGO | `package.json` |
| FDD-INT-12 | `docs/FDD.md` | Integração (código) | Listagem com paginated() | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-13 | `docs/FDD.md` | Integração (código) | Testes no padrão supertest + factories | CODIGO | `tests/helpers/factories.ts` |
| FDD-AC-01 | `docs/FDD.md` | Critério técnico | 1 linha PENDING por webhook assinante na transação | TRANSCRICAO | [09:41] Bruno |
| FDD-AC-02 | `docs/FDD.md` | Critério técnico | Falha na outbox causa rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-03 | `docs/FDD.md` | Critério técnico | Sem assinante, sem insert | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-04 | `docs/FDD.md` | Critério técnico | DELIVERED em < 10 s com assinatura verificável | TRANSCRICAO | [09:09] Diego |
| FDD-AC-05 | `docs/FDD.md` | Critério técnico | Backoff completo e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-AC-06 | `docs/FDD.md` | Critério técnico | Timeout > 10 s reagenda | TRANSCRICAO | [09:42] Diego |
| FDD-AC-07 | `docs/FDD.md` | Critério técnico | http retorna WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-08 | `docs/FDD.md` | Critério técnico | Assinatura dupla por 24 h após rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-09 | `docs/FDD.md` | Critério técnico | Replay 403 para OPERATOR, 202 para ADMIN, mesmo event_id | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-10 | `docs/FDD.md` | Critério técnico | > 64 KB vai à DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-11 | `docs/FDD.md` | Critério técnico | Deliveries limitado a 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-12 | `docs/FDD.md` | Critério técnico | Ordem por pedido mesmo com retry | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-01 | `docs/FDD.md` | Risco | Crescimento da outbox | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-02 | `docs/FDD.md` | Risco | Cliente lento atrasa a fila | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-03 | `docs/FDD.md` | Risco | Secret em logs | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-04 | `docs/FDD.md` | Risco | Duplicata após queda do worker | TRANSCRICAO | [09:24] Diego |
| FDD-RISK-05 | `docs/FDD.md` | Risco | Rajadas sem rate limit | TRANSCRICAO | [09:38] Diego |
| FDD-RISK-06 | `docs/FDD.md` | Risco (código) | Leitura extra na transação de status | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-OPEN-01 | `docs/FDD.md` | Ponto a validar (código) | Criação de pedido (PENDING) não emite evento | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-OPEN-02 | `docs/FDD.md` | Ponto a validar | Tamanho do lote ("batch pequeno") | TRANSCRICAO | [09:08] Diego |
| FDD-OPEN-03 | `docs/FDD.md` | Ponto a validar | Formato da multi-assinatura no grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-OPEN-04 | `docs/FDD.md` | Ponto a validar | Descoberta dos IDs da DLQ pelo ADMIN | TRANSCRICAO | [09:18] Diego |
