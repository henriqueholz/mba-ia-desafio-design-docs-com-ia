# ADR-001 — Padrão Outbox no MySQL, gravado na mesma transação do `changeStatus`

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:03–09:08 |
| **Decisores** | Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos) |
| **Relacionados** | [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md), [RFC](../RFC.md) |

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) querem ser notificados quando o status dos seus pedidos muda. Hoje eles fazem polling em `GET /orders`.

A mudança de status acontece em `OrderService.changeStatus` ([src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts)), dentro de um `prisma.$transaction` que já faz três coisas:

1. valida a transição com `canTransition` ([src/modules/orders/order.status.ts](../../src/modules/orders/order.status.ts));
2. debita ou repõe estoque (`debitStock` / `replenishStock`);
3. atualiza `orders.status` e insere em `order_status_history`.

Precisamos de um mecanismo que garanta: **se o status mudou, o evento de notificação existe; se a transação falhou, o evento não existe**. A aplicação não tem hoje nenhuma fila, broker ou mecanismo de eventos. A única infraestrutura persistente é o MySQL 8 ([docker-compose.yml](../../docker-compose.yml), [prisma/schema.prisma](../../prisma/schema.prisma)).

## Decisão

Adotar o **padrão Transactional Outbox no MySQL existente**:

- Criar a tabela `webhook_outbox`. Cada linha representa uma entrega pendente de um evento para um endpoint de webhook.
- A inserção na outbox acontece **dentro da mesma transação** de `changeStatus`, via uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `Prisma.TransactionClient` corrente (ver [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)).
- Se a inserção na outbox falhar, a transação inteira faz rollback: o status do pedido **não muda**.
- A tabela tem status de processamento (`PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`) e índices em `status` e `created_at`, para que o worker leia só os pendentes mais antigos em lotes pequenos.
- IDs em UUID (`CHAR(36)`), seguindo o padrão de todas as tabelas do `schema.prisma`.
- O arquivamento de linhas entregues (a ideia citada foi algo como 30 dias) **fica fora do escopo** desta feature.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo HTTP síncrono dentro do `changeStatus`** | A transação já é pesada (orders + history + estoque). Um cliente lento travaria a mudança de status de outros pedidos, e não há resposta sensata para um cliente fora do ar: não dá para fazer rollback da mudança de status por causa disso ([09:04] Bruno, [09:06] Diego). |
| **Redis Streams (ou broker equivalente)** | Exige subir e operar infraestrutura nova. Para um time pequeno, um Redis Cluster só para isso é overengineering ([09:07] Larissa, [09:07] Diego). Também não resolve a atomicidade sozinho: publicar no Redis depois do commit ainda pode perder o evento. |
| **Inserir na outbox fora da transação (após o commit)** | Perde a garantia central: o status poderia mudar sem o evento sair ([09:40] Bruno, [09:41] Diego). |

## Consequências

**Positivas**

- Atomicidade real entre mudança de status e registro do evento, sem coordenação distribuída.
- Zero infraestrutura nova: mesmo MySQL, mesmo Prisma, mesma stack.
- A outbox também funciona como registro auditável do que foi (ou não) notificado.

**Negativas**

- A transação de `changeStatus` fica um pouco mais longa (inserts adicionais, um por webhook interessado).
- A tabela cresce continuamente. Sem o arquivamento (fora do escopo), o volume precisa ser monitorado.
- Latência de entrega depende do polling (ver [ADR-002](ADR-002-worker-separado-em-polling.md)), em vez de ser push imediato.

**Trade-off explícito:** aceitamos acoplar a escrita do evento ao banco transacional, e com isso ter latência de segundos e uma tabela crescendo, em troca de consistência garantida e nenhuma infraestrutura nova.
