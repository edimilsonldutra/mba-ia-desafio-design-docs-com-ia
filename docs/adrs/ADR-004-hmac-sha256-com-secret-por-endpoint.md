# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação

- **Status:** Aceito
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma)
- **Relacionados:** [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

Os webhooks levam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado no caminho ([09:19] Sofia). A comunicação é só de saída (*outbound*) ([09:02]–[09:03] Marcos, Sofia). Um cliente já vazou uma secret em log da própria aplicação ([09:22] Diego).

## Decisão

1. **Assinar o corpo da requisição com HMAC-SHA256** e enviar a assinatura no header `X-Signature`. O cliente recalcula e compara do lado dele ([09:20] Sofia; [09:22] Sofia: "HMAC-SHA256 sobre o corpo do request").
2. **Uma secret única por endpoint de webhook**, gerada pela plataforma e devolvida na criação ([09:21] Sofia; [09:31] Marcos). Ela fica na configuração do webhook junto com `url`, `customer_id` e o estado ativo ([09:21] Bruno).
3. **Rotação pela API**, com **grace period de 24 h**: a secret antiga continua válida por 24 h em paralelo e depois deixa de valer ([09:21] Sofia).
4. **TLS obrigatório:** a URL precisa ser `https`. Uma URL `http` é recusada com erro de validação no schema Zod ([09:23] Sofia).

## Alternativas consideradas

1. **Secret global da plataforma.** Descartada: "se vaza uma, vaza tudo" ([09:21] Sofia).
2. **Rotação sem período de convivência (troca imediata).** Descartada implicitamente: o cliente precisa de tempo para migrar os sistemas dele ([09:21] Sofia).
3. **Outros algoritmos de assinatura.** Não adotados: HMAC-SHA256 é o padrão de mercado e "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).

## Consequências

**Positivas**
- Autenticidade e integridade verificáveis pelo cliente com bibliotecas padrão.
- O vazamento de uma secret afeta um único endpoint, e o cliente consegue rotacioná-la sem interromper o recebimento.

**Negativas / trade-offs**
- As secrets precisam ser recuperáveis para assinar, então não podem ser armazenadas como hash. Isso exige cuidado no armazenamento e nos logs (o logger em `src/shared/logger/index.ts` precisa redigir o campo `secret`).
- Durante o grace period o worker assina com as duas secrets válidas, o que aumenta a complexidade do header (detalhado no FDD).
- A geração da secret e o HMAC passam por **revisão de segurança dedicada** antes do deploy ([09:46] Sofia), com impacto no cronograma.
