# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Henrique Castro (a partir da reunião técnica conduzida por Larissa) |
| **Status** | Em revisão |
| **Data** | 2026-09-28 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos irmãos** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B sobre mudanças de status de pedido por meio de **webhooks outbound**. O evento é gravado numa **outbox no MySQL, na mesma transação do `changeStatus`**. Um **worker em processo separado** lê essa outbox **a cada 2 s** e faz a entrega.

- **Assinatura:** cada entrega é assinada com **HMAC-SHA256**, usando uma secret própria de cada endpoint e rotacionável.
- **Garantia:** **at-least-once**, com `X-Event-Id` para o cliente deduplicar.
- **Falhas:** retry com backoff de **1m/5m/30m/2h/12h**. Depois disso, o evento vai para uma **DLQ em tabela separada**, com replay manual por `ADMIN`.
- **Infraestrutura:** nada novo. O módulo segue os padrões atuais do projeto.

Estimativa: **3 sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente para ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders`, o que é lento e caro para eles. A Atlas sinalizou que pode migrar para um concorrente se não houver entrega até o fim do trimestre. Pelo que foi alinhado depois, o prazo é o fim de novembro.

Para eles, "tempo real" significa **menos de 10 segundos**.

No lado técnico, o OMS não tem **nenhum** mecanismo de eventos, filas ou notificação externa. A mudança de status está concentrada em `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../src/modules/orders/order.service.ts)). Ela roda numa transação que valida a máquina de estados ([order.status.ts](../src/modules/orders/order.status.ts)), movimenta estoque e grava `order_status_history`. Qualquer solução precisa se encaixar nessa transação **sem torná-la dependente da disponibilidade de terceiros**.

## 3. Objetivos e não-objetivos

**Objetivos**

- Notificar o cliente em menos de 10 s após a mudança de status, no caso normal.
- Nunca haver status alterado sem o evento correspondente registrado, nem evento de uma mudança que sofreu rollback.
- Permitir ao cliente verificar a origem e a integridade de cada chamada.
- Ser operável por um time pequeno, sem infraestrutura nova.

**Não-objetivos (desta fase)**

- Webhooks inbound (o cliente enviar para nós).
- Ordem global de eventos.
- Aviso por e-mail quando o webhook falha.
- Dashboard visual para o cliente.
- Rate limiting de saída.
- Arquivamento da outbox.

## 4. Proposta técnica

```
 API (processo 1)                                     Worker (processo 2)
 ┌────────────────────────────────────┐              ┌──────────────────────────────┐
 │ PATCH /orders/:id/status           │              │ loop a cada 2s               │
 │  └─ OrderService.changeStatus      │              │  ├─ lê PENDING mais antigos  │
 │      BEGIN                         │              │  ├─ assina (HMAC-SHA256)     │
 │       ├─ update orders             │   MySQL      │  ├─ POST https://cliente     │
 │       ├─ insert status_history     │ ┌──────────┐ │  │   (timeout 10s)           │
 │       ├─ estoque (debit/replenish) │ │ webhook_ │ │  ├─ 2xx → DELIVERED          │
 │       └─ publishWebhookEvent(tx) ──┼─► outbox   ◄─┤  ├─ falha → agenda retry     │
 │      COMMIT                        │ └──────────┘ │  └─ 5 retries → DLQ          │
 │ CRUD /webhooks, deliveries, replay │ webhook_dead_│                              │
 └────────────────────────────────────┘ letter, ...  └──────────────────────────────┘
```

**Componentes propostos**

1. **Configuração de webhooks (API).** CRUD autenticado de endpoints por cliente. Cada endpoint tem `url` HTTPS, uma lista de status de interesse e uma secret gerada pela plataforma. O `customer_id` vai no corpo ou no path, **não vem do JWT**, porque o JWT é do usuário operador. A API também oferece endpoints para rotacionar a secret e consultar o histórico de entregas.
2. **Publicação transacional.** `publishWebhookEvent(tx, …)` é chamada dentro de `changeStatus`. Ela aplica o filtro de eventos, renderiza um snapshot enxuto do pedido e grava uma linha na outbox para cada endpoint interessado. Se falhar, a mudança de status sofre rollback ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)).
3. **Worker de entrega.** Novo entry-point `src/worker.ts` (`npm run worker`), com `PrismaClient` próprio e polling de 2 s. Roda uma única instância ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).
4. **Resiliência.** Timeout de 10 s. Retry com backoff 1m/5m/30m/2h/12h. DLQ separada, com replay manual por `ADMIN` e registro de auditoria ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).
5. **Segurança.** HMAC-SHA256 do corpo, secret por endpoint, rotação com 24 h de convivência, HTTPS obrigatório e payload de no máximo 64 KB ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
6. **Semântica de entrega.** At-least-once, com `X-Event-Id` estável entre retries ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
7. **Padrões do projeto.** Módulo `src/modules/webhooks`, erros `AppError` com prefixo `WEBHOOK_`, Pino, error middleware central e `requireRole` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

Contratos, schema das tabelas, algoritmo do worker e matriz de erros estão no [FDD](FDD.md).

## 5. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Origem |
| --- | --- | --- | --- |
| ALT-1 | **Disparo HTTP síncrono no `changeStatus`** | Acopla a disponibilidade do cliente à nossa transação. Um cliente lento trava a mudança de status de outros pedidos, e cliente fora do ar não tem saída: não dá para fazer rollback da mudança de status. | [09:04] Bruno, [09:06] Diego |
| ALT-2 | **Redis Streams / broker dedicado** | Resolveria o transporte, mas exige subir e operar infraestrutura nova, o que é overengineering para um time pequeno. A outbox no MySQL existente entrega a atomicidade que o broker sozinho não dá. | [09:07] Larissa, [09:07] Diego |
| ALT-3 | **Trigger do MySQL para acordar o worker** | O MySQL não tem `LISTEN/NOTIFY`, e uma trigger só executa SQL. Notificar o worker exigiria gambiarras. O polling de 2 s já atende a meta de menos de 10 s. | [09:09] Bruno, [09:09] Diego |
| ALT-4 | **Worker dentro do processo da API** | Um restart da API derruba o worker junto. | [09:11] Diego |
| ALT-5 | **Retry indefinido ou só 3 tentativas** | Indefinido deixa eventos pendurados para sempre. Com 3 tentativas, a janela fica em uns 30 min e não cobre manutenções de 2 h que clientes já tiveram. | [09:15]–[09:16] Diego, Bruno |
| ALT-6 | **Secret global da plataforma** | Um vazamento compromete todos os clientes. | [09:21] Sofia |
| ALT-7 | **Exactly-once** | Exige coordenação dos dois lados. É muito mais complexo, para um ganho marginal sobre at-least-once com `event_id`. | [09:25] Diego |

## 6. Questões em aberto

| # | Questão | Situação na reunião | Dono sugerido |
| --- | --- | --- | --- |
| Q1 | **Rate limiting de saída.** Um cliente com 50 pedidos mudando em um minuto recebe 50 chamadas. | Fora do escopo por enquanto: "observar e decidir depois". | Diego ([09:38]–[09:39]) |
| Q2 | **Aviso ao cliente quando o webhook falha repetidamente** (e-mail após N falhas). | Adiado para a próxima fase, depois de medir o impacto. | Marcos / Larissa ([09:37]) |
| Q3 | **Escala horizontal do worker.** Particionar por `order_id` ou usar lock pessimista. | Limitação conhecida, "problema do futuro". | Diego ([09:13]) |
| Q4 | **Arquivamento de linhas entregues da outbox** (algo como 30 dias). | Citado e explicitamente fora do escopo desta feature. | Diego ([09:08]) |
| Q5 | **Endurecer a autorização do CRUD de webhooks.** Hoje vale qualquer role autenticada. O modelo `User` não tem vínculo com `Customer` ([prisma/schema.prisma](../prisma/schema.prisma)), então um usuário pode gerenciar webhooks de qualquer cliente. | "Mais pra frente a gente pode endurecer." | Sofia ([09:37]) |
| Q6 | **Proteção da secret em repouso e formato da assinatura durante o grace period de 24 h.** O HMAC exige a secret recuperável, e duas secrets ficam válidas ao mesmo tempo. | Não discutido. Entra na revisão de segurança de 2 dias. | Sofia ([09:46]) |
| Q7 | **Evento na criação do pedido.** `OrderService.create` grava o histórico `null → PENDING` sem passar por `changeStatus`, então essa transição **não** gera webhook nesta proposta. | Não discutido. A reunião tratou só de "quando o status muda". | Marcos / Bruno |

## 7. Impacto e riscos

**Impacto no sistema existente**

- `changeStatus` ganha uma escrita a mais, na mesma transação. Um erro de outbox passa a impedir a mudança de status, **de propósito**.
- Um segundo processo (`npm run worker`) entra no deploy.
- Três ou quatro tabelas novas no MySQL. A outbox cresce sem arquivamento (Q4).
- Nenhuma mudança em contratos existentes da API, no error middleware ou na autenticação.

**Principais riscos** (detalhes e mitigações no [FDD](FDD.md#15-riscos-e-mitigação))

| Risco | Mitigação resumida |
| --- | --- |
| Cliente não deduplica e processa o mesmo evento duas vezes | `X-Event-Id` estável e documentação em destaque no portal ([09:26] Marcos) |
| Ordem quebrada entre eventos do mesmo pedido durante um retry | Limitação documentada. O cliente tem `from_status`/`to_status` e pode consultar `GET /orders/:id` |
| Vazamento de secret | Secret por endpoint, rotação com 24 h de convivência, `redact` no logger, revisão de segurança |
| Crescimento da outbox degradando o polling | Índices em `status`/`created_at`, leitura só de pendentes em lote pequeno. Arquivamento como follow-up |
| Prazo (fim de novembro) | Estimativa de 3 sprints, com a revisão da Sofia (mínimo 2 dias úteis) planejada no fim |

## 8. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL, na transação do `changeStatus` |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry 1m/5m/30m/2h/12h + DLQ separada + replay `ADMIN` |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com 24 h |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) | Payload snapshot enxuto e filtro na inserção |

## 9. Próximos passos

1. Sessão de revisão deste RFC e do FDD com Bruno e Diego antes de começar a codar ([09:50] Larissa).
2. Fechar Q6 e Q7 antes da sprint 1. As demais questões ficam como follow-up.
3. Agendar a revisão de segurança da Sofia no fim da sprint 3.
