# Webhooks de Notificação de Pedidos — Design Docs gerados com IA

Entrega do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"** (MBA Engenharia de Software com IA — Full Cycle). O enunciado original está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, numa reunião técnica de ~55 minutos, construir um **sistema de webhooks de saída** para avisar clientes B2B quando o status dos pedidos deles muda. A única coisa registrada foi a transcrição da call (`TRANSCRICAO.md`). A tarefa foi transformar essa conversa e o código existente num pacote de design docs acionável (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade) sem alterar uma linha da aplicação.

O desafio real não foi escrever documentos, foi **filtrar**: separar o que foi decidido, o que foi descartado e o que foi adiado; colocar cada informação na altura certa (produto, arquitetura, decisão pontual, implementação); e garantir que nada fosse inventado. Por isso, cada item tem origem rastreável até uma fala com timestamp ou um arquivo real do repositório.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (modelo Claude Opus 5.5) | Agente principal: leu o código e a transcrição, extraiu decisões, redigiu ADRs, RFC, FDD e PRD, e gerou e validou o tracker |
| **Script Python de validação** (escrito pelo agente durante o processo) | Gera o `docs/TRACKER.md` a partir de um mapeamento ID → fonte e **falha** se algum `[hh:mm] Falante` não existir na transcrição, se algum caminho de código não existir ou se algum ID citado nos documentos não tiver linha no tracker |

## Workflow adotado

1. **Contextualização.** Leitura integral da `TRANSCRICAO.md` e do código que a feature toca: `order.service.ts` (`changeStatus`), `order.status.ts`, `shared/errors/*`, middlewares de auth, erro e validação, `app.ts`, `routes/index.ts`, `server.ts`, `config/*`, `logger` e `prisma/schema.prisma`.
2. **Extração dirigida.** Classificação de cada fala em: decisão fechada, requisito funcional, requisito não funcional, alternativa descartada, item adiado/fora de escopo ou ponto em aberto, sempre com o timestamp.
3. **ADRs primeiro** (7 arquivos): as 6 decisões principais da reunião mais "snapshot e filtro na inserção". Elas formam o esqueleto do resto.
4. **RFC** (conciso, nível de arquitetura): proposta, alternativas descartadas, questões em aberto e links para os ADRs.
5. **FDD** (nível de implementação): modelo de dados, 8 contratos com exemplos, fluxos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção **Integração com o sistema existente**, com 13 arquivos reais e números de linha.
6. **PRD** por último, como consolidação de produto: 14 requisitos funcionais, metas quantitativas, fora de escopo e riscos com probabilidade e impacto.
7. **Tracker gerado e validado por script**: cada ID usado nos documentos aparece no tracker com fonte verificada.
8. **Revisão final** contra a checklist de critérios de aceite, item por item.

Convenção que guiou tudo: **IDs nos próprios documentos** (`PRD-FR-03`, `RFC-ALT-02`, `FDD-CONTRATO-05`…). Assim o tracker não é um anexo solto: qualquer item de qualquer documento pode ser procurado nele.

## Prompts customizados

**1. Extração dirigida da transcrição** (a base de todos os documentos):

```text
Leia TRANSCRICAO.md inteira. Para CADA fala relevante, classifique em exatamente uma categoria:
DECISAO_FECHADA | REQUISITO_FUNCIONAL | REQUISITO_NAO_FUNCIONAL | ALTERNATIVA_DESCARTADA |
ADIADO_OU_FORA_DE_ESCOPO | QUESTAO_EM_ABERTO | DETALHE_DE_IMPLEMENTACAO.
Para cada item devolva: timestamp no formato [hh:mm] Falante, citação curta, categoria,
e quem fechou a decisão (quando houver "decidido"/"anotado"/"fica como").
Regras: não infira requisitos que ninguém falou; se algo foi descartado ou adiado, isso NÃO
é requisito; se dois falantes discordaram, registre os dois lados e qual prevaleceu.
```

**2. ADR com âncoras no código e na fala:**

```text
Escreva o ADR-NNN em formato MADR (Status, Contexto, Decisão, Alternativas Consideradas,
Consequências positivas e negativas). Cada afirmação factual deve terminar com a origem:
([hh:mm] Falante) para a transcrição ou `caminho/arquivo.ts:linha` para o código.
Alternativas só podem ser as discutidas na reunião (cite quem descartou e por quê).
Nas consequências, explicite o trade-off aceito. Não descreva implementação em detalhe:
isso vai para o FDD.
```

**3. Integração com o código existente (FDD):**

```text
Para a seção "Integração com o sistema existente" do FDD, abra cada arquivo citado e devolva:
caminho real, linha(s) exatas do ponto de extensão e a mudança proposta. Se o arquivo não
precisa mudar (ex.: error.middleware.ts já trata AppError), diga isso explicitamente e mostre
o trecho que prova. Nunca cite um arquivo que não exista no repositório.
```

**4. Revisão anti-alucinação:**

```text
Releia PRD, RFC e FDD. Liste todo valor numérico, nome de endpoint, header, tabela, código de
erro e regra que NÃO tenha fala ou arquivo de origem. Para cada um: (a) remova, ou
(b) marque explicitamente como "decisão de implementação" com justificativa e leve para
"pontos a validar". Verifique também se algo descartado na reunião aparece como requisito.
```

## Iterações e ajustes

Foram **3 rodadas principais**:
1. rascunho de cada documento na ordem ADRs → RFC → FDD → PRD;
2. revisão de consistência com o código (itens 2, 3, 4, 5 e 6 abaixo);
3. revisão anti-alucinação (item 7), seguida da geração e validação do tracker por script.

As correções concretas:

1. **"5 tentativas" × 5 intervalos de backoff.** A primeira leitura tratava "5 tentativas" como 5 envios no total, mas a reunião lista 5 intervalos (1m/5m/30m/2h/12h) e diz "quase 15 horas entre primeira falha e última tentativa". A soma (14 h 36 min) só bate se forem **envio inicial + 5 retentativas**. A interpretação ficou explícita no [ADR-003](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md) e numa tabela no FDD.
2. **Prefixo real das rotas.** A transcrição fala em `POST /admin/webhooks/dead-letter/:id/replay`, mas o código monta tudo sob `/api/v1` (`src/app.ts:67`). Os contratos foram corrigidos para `/api/v1/...`.
3. **Limite de 64 KB no lugar errado.** O primeiro desenho validava o tamanho na inserção da outbox. Como a inserção roda **dentro** da transação de `changeStatus`, um payload anômalo daria rollback na mudança de status do pedido. A checagem foi movida para o worker (o evento vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`).
4. **Retry quebrava a ordem por pedido.** Com backoff, um evento `PROCESSING` reagendado seria ultrapassado pelo `SHIPPED` seguinte, violando a garantia "ordering por order_id" ([09:12] Diego). Acrescentei no FDD a regra de elegibilidade por `(webhook, pedido)` e um critério de aceite específico (FDD-AC-12).
5. **Remoção × desativação.** O `DELETE` foi modelado como `active = false`, o que colidia com `PATCH { active: false }`. Entrou o campo `deletedAt`.
6. **`WEBHOOK_INVALID_URL` × Zod.** A reunião manda validar `https` "no schema Zod", mas o `validate()` existente sempre emite `VALIDATION_ERROR`. A matriz exige `WEBHOOK_*`. Solução: a regra fica declarada no schema e o service usa `safeParse` para emitir o código do módulo.
7. **Conteúdo sem origem removido.** O PRD trazia uma "validação piloto com a Atlas" que ninguém mencionou: foi trocada pelos testes ponta a ponta citados em [09:46]. As notas de probabilidade e impacto dos riscos foram marcadas como avaliação do documento. Detalhes necessários para tornar o FDD acionável, mas não discutidos (multi-assinatura no grace period, tamanho do lote, semântica do DELETE), ficaram rotulados como **decisão de implementação** e listados em "Pontos a validar".
8. **Estrutura de pastas.** `docs/adrs/README.md` do boilerplate foi removido para que a pasta contenha só os arquivos `ADR-NNN-*.md` exigidos. O índice dos ADRs está na seção 7 do RFC.

## Como navegar a entrega

| Ordem | Arquivo | Para quê |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Problema, público, metas, escopo e fora de escopo |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica, alternativas descartadas, questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | Uma decisão por arquivo: [001 outbox](docs/adrs/ADR-001-outbox-no-mysql.md) · [002 worker](docs/adrs/ADR-002-worker-separado-em-polling.md) · [003 retry/DLQ](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md) · [004 HMAC](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) · [005 at-least-once](docs/adrs/ADR-005-at-least-once-com-x-event-id.md) · [006 padrões](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) · [007 snapshot/filtro](docs/adrs/ADR-007-snapshot-e-filtro-na-insercao.md) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: dados, contratos, fluxos, erros, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item (transcrição ou código) |

O código da aplicação (`src/`, `prisma/`, `tests/`) e a `TRANSCRICAO.md` não foram alterados.
