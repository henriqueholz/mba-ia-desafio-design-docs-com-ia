# Da Reunião ao Documento — Design Docs do Sistema de Webhooks de Pedidos

> Entrega do desafio do MBA. O enunciado original está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, numa reunião de cerca de 55 minutos, construir um **sistema de webhooks para notificar clientes B2B quando o status de um pedido muda**. O único registro da reunião é a transcrição literal ([TRANSCRICAO.md](TRANSCRICAO.md)). O desafio é transformar essa conversa, junto com o código existente, num pacote de design docs acionável: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade.

A parte difícil não é escrever. É **filtrar**. A reunião mistura decisões fechadas, requisitos, ideias descartadas (e-mail, dashboard, Redis), pontos adiados (rate limiting, escala do worker) e até correções no meio da conversa, como o `customer_id` que "vem do JWT" e, dois minutos depois, não vem mais. Cada documento opera numa altura diferente, e toda afirmação precisa apontar para uma fala `[hh:mm] Nome` ou para um arquivo real do repositório.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (extensão do VS Code, modelo Claude Opus 5.5) | Ferramenta principal. Leu o repositório inteiro (schema Prisma, `order.service.ts`, middlewares, erros, logger), cruzou com a transcrição, redigiu os documentos, fez a auditoria de rastreabilidade e os commits item a item |
| **Leitura direta do código pela IA** (ferramentas de leitura de arquivo do Claude Code) | Garantir que cada caminho citado existe e que o comportamento descrito bate com o código. Exemplos: `NotFoundError` não aceita código customizado, e `validate.middleware.ts` converte todo `ZodError` em `VALIDATION_ERROR` |

## Workflow adotado

Segui a ordem sugerida no enunciado, com um commit por entregável:

1. **Contextualização.** Leitura completa da transcrição e mapeamento do código: módulos, `changeStatus` e sua transação, máquina de estados em `order.status.ts`, hierarquia de erros, `requireRole`, error middleware, logger Pino, `createPrismaClient` e `server.ts`.
2. **Filtragem dirigida da transcrição.** Classifiquei cada fala em uma de seis categorias: decisão fechada, requisito funcional, restrição/RNF, descartado, adiado/em aberto, detalhe técnico secundário. Essa tabela virou o esqueleto de todos os documentos.
3. **ADRs primeiro** (7 no total). As 6 decisões principais, mais 1 ADR de decisões secundárias (snapshot do payload + filtro na inserção), porque elas mudam o modelo de dados.
4. **RFC.** Proposta em nível de arquitetura, com 7 alternativas descartadas e 7 questões em aberto, sem descer ao nível do FDD.
5. **FDD.** Fluxos, contratos, matriz de erros e a seção "Integração com o sistema existente", escrita **com o código aberto ao lado**.
6. **PRD.** Consolidação para produto: problema, personas, métricas, escopo, 16 RFs e 11 RNFs.
7. **Tracker.** Montado varrendo os documentos prontos. Cada linha foi conferida contra o timestamp e o falante na transcrição.
8. **README** e revisão final contra a checklist de critérios de aceite.

Regra que mantive durante todo o processo: **quando o FDD precisou de um detalhe que a reunião não definiu** (tamanho do lote, formato da assinatura durante a rotação, tipo de coluna), ele foi marcado como **(def. FDD)**. Se era algo que um revisor deveria decidir, foi para as **Questões em aberto** do RFC.

## Prompts customizados

**1. Filtragem da transcrição**, o prompt mais importante do processo:

```text
Você é um tech lead revisando a transcrição TRANSCRICAO.md de uma reunião sobre webhooks.
Para CADA fala relevante, classifique em exatamente uma categoria:
  DECISAO_FECHADA | REQUISITO_FUNCIONAL | RESTRICAO_RNF | DESCARTADO | ADIADO_OU_ABERTO | DETALHE_SECUNDARIO
Regras:
- Cite sempre "[hh:mm] Nome" exatamente como na transcrição.
- Se uma fala é corrigida depois (ex.: alguém propõe X e outro participante muda para Y), registre só a versão final e anote a correção.
- NÃO promova a requisito nada que foi descartado ou adiado ("fora de escopo", "próxima fase", "observar", "agora não").
- Liste separadamente os pontos que a reunião NÃO decidiu mas que a implementação vai precisar decidir.
Saída: tabela markdown com colunas Categoria | Resumo | Localização.
```

**2. Integração com o código no FDD:**

```text
Com base nos ADRs em docs/adrs/ e no código real do repositório, escreva a seção
"Integração com o sistema existente" do FDD.
Para cada arquivo citado:
- confirme que o caminho existe lendo o arquivo (não suponha);
- diga se é alteração, reuso ou modelo, e o que exatamente muda (função, tipo, linha lógica);
- aponte restrições do código que afetam o design (ex.: classes de erro que fixam o errorCode,
  middlewares que reescrevem erros, fábricas vs. singletons).
Inclua o trecho de changeStatus mostrando onde publishWebhookEvent(tx, ...) entra.
Não invente arquivos. Se algo não existe (ex.: lib de métricas), diga que não existe.
```

**3. Auditoria anti-alucinação:**

```text
Percorra docs/PRD.md, docs/RFC.md, docs/FDD.md e docs/adrs/*.md.
Para cada requisito, decisão, restrição, risco, alternativa e questão em aberto:
1. Encontre a fala de origem em TRANSCRICAO.md ([hh:mm] Nome) ou o arquivo de origem no código.
2. Se não encontrar origem, marque como SEM_ORIGEM e sugira: remover, reescrever ou mover para "Questões em aberto".
3. Verifique se algum item DESCARTADO/ADIADO aparece como requisito.
4. Verifique se o timestamp citado realmente corresponde ao falante citado.
Gere a tabela no formato: ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização.
```

## Iterações e ajustes

Foram **4 ciclos principais** de geração, revisão crítica e correção:

1. **Filtragem da transcrição (ciclo 1).**
   - *`customer_id` do JWT:* a primeira extração registrava o `customer_id` como "implícito do JWT", que é o que o Marcos diz em [09:31]. Em [09:32], Bruno aponta que o JWT é do operador, e Larissa corrige: o `customer_id` vai no body ou no path. Reforcei no prompt a regra "registre só a versão final de falas corrigidas".
   - *Retentativas:* a IA também tratou "5 tentativas" e a lista de 5 intervalos como a mesma coisa. Fazendo a conta (1m+5m+30m+2h+12h ≈ 14h36, que é o "quase 15 horas" da reunião), o modelo coerente é 1 tentativa inicial + 5 retentativas. A interpretação ficou registrada explicitamente no ADR-003.
2. **ADRs e RFC (ciclo 2).**
   - *Garantia de ordem:* o rascunho do ADR-002 repetia a fala da reunião e afirmava que a ordem por pedido era garantida com single-worker. Não é, quando há retry: um evento em backoff é ultrapassado pelo próximo evento do mesmo pedido. Ajustei o ADR-002 para registrar isso como limitação conhecida e levei o ponto para os riscos do RFC e do FDD.
   - *Ideias descartadas:* conferi que e-mail, dashboard e rate limiting aparecem **só** como fora de escopo ou questões em aberto, nunca como requisito.
3. **FDD contra o código (ciclo 3).** Foi o ciclo com mais correções, todas vindas de ler o código em vez de supor:
   - O rascunho da matriz de erros fazia `WebhookNotFoundError extends NotFoundError`. Mas `NotFoundError` fixa o código `NOT_FOUND` no construtor ([http-errors.ts](src/shared/errors/http-errors.ts)). Os 404 do módulo passaram a estender `AppError` direto.
   - `validate.middleware.ts` transforma todo `ZodError` em `VALIDATION_ERROR`. Colocar a regra `https` só no schema nunca geraria `WEBHOOK_INVALID_URL`. A regra fica no schema, mas é aplicada no service para emitir o código certo.
   - `PENDING` aparecia como evento assinável, mas nenhuma transição de `order.status.ts` tem `PENDING` como destino. Os valores válidos de `events` passaram a ser derivados do mapa `transitions`.
   - O payload estava como `TEXT`, que tem teto de 65.535 bytes, 1 byte abaixo do limite de 64 KB. Mudei para `MediumText`.
   - `OrderService.create` grava o histórico `null → PENDING` **sem** passar por `changeStatus`. Em vez de inventar um requisito para isso, virou a questão em aberto Q7 do RFC.
4. **PRD e Tracker (ciclo 4).** A auditoria do prompt 3 encontrou, por exemplo, um exemplo inventado num risco do PRD ("baixa estoque duas vezes do lado do cliente"), que foi removido. Também encontrou métricas sem baseline: a redução do polling em `GET /orders` ficou como indicador sem meta numérica, porque a reunião não deu número. Todos os timestamps do tracker foram conferidos contra o falante.

## Como navegar a entrega

Ordem sugerida de leitura:

| # | Documento | Pergunta que responde |
| --- | --- | --- |
| 1 | [docs/PRD.md](docs/PRD.md) | Por que e o quê? Problema, personas, métricas, escopo, requisitos |
| 2 | [docs/RFC.md](docs/RFC.md) | Como pretendemos resolver e o que está em aberto? |
| 3 | [docs/adrs/](docs/adrs/) | Por que decidimos exatamente assim? |
| 4 | [docs/FDD.md](docs/FDD.md) | Como construir, em detalhe? Fluxos, contratos, erros, integração com o código |
| 5 | [docs/TRACKER.md](docs/TRACKER.md) | De onde veio cada item? |

ADRs:

- [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Payload snapshot e filtro na inserção](docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)

O código da aplicação (`src/`, `prisma/`, `tests/`) **não foi alterado**. Ele serve apenas de referência.
