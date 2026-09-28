# ADR-007 — Payload enxuto renderizado na inserção (snapshot) e filtro de eventos aplicado antes de gravar na outbox

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:33–09:34, 09:43–09:44 e 09:51–09:52 |
| **Decisores** | Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM) |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [FDD](../FDD.md) |

## Contexto

Três decisões secundárias afetam o formato e o volume de dados na outbox:

1. **Quando renderizar o payload:** na inserção (dentro de `changeStatus`) ou na hora do envio (no worker)?
2. **O que vai no payload:** o pedido completo com itens, ou só os campos básicos?
3. **Onde aplicar o filtro de eventos por webhook:** cada endpoint escolhe quais status quer ouvir ("só `SHIPPED` e `DELIVERED`").

## Decisão

1. **Snapshot na inserção.** O JSON é renderizado e gravado na outbox dentro da transação de `changeStatus`. Assim, reflete o estado do pedido **no momento da mudança**, mesmo que o pedido mude depois ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
2. **Payload enxuto:** `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Não inclui `items`.** Se o cliente quiser detalhes, consulta `GET /orders/:id` ([09:43] Diego).
3. **Filtro aplicado na inserção.** `publishWebhookEvent` só grava linhas para webhooks **ativos** do `customer_id` do pedido cuja lista de eventos contém o `to_status`. Se nenhum webhook se interessa, **nada é inserido** ([09:34] Bruno, [09:34] Diego).
4. **Limite de 64 KB** no payload. Se passar disso, é erro, e o evento não é truncado ([09:24] Larissa). Com o payload enxuto, na prática o limite não deve ser atingido.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Guardar só `order_id` e renderizar no envio** | Com retries de até cerca de 15 h, o pedido pode ter mudado de novo. O evento de "virou `SHIPPED`" chegaria com dados de um estado posterior ("caso esquisito", [09:52] Larissa). |
| **Incluir os itens do pedido no payload** | Infla o payload sem necessidade. O cliente pode buscar os detalhes sob demanda ([09:43] Diego, [09:44] Bruno). |
| **Filtrar na hora de enviar (no worker)** | Grava linhas que nunca serão enviadas e aumenta a tabela e o trabalho do worker ([09:34] Bruno). |
| **Truncar payloads acima do limite** | Um evento truncado é um evento corrompido. Se chegou a esse tamanho, algo está errado ([09:23] Sofia). |

## Consequências

**Positivas**

- Os eventos são imutáveis e historicamente corretos. Retry e replay enviam exatamente o mesmo conteúdo, o que combina com o `event_id` estável do [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).
- A outbox só contém trabalho real, e o worker não precisa consultar `orders`.
- Payloads pequenos, bem abaixo do limite de 64 KB.

**Negativas**

- A renderização acontece dentro da transação de `changeStatus`, com um pouco mais de trabalho no caminho crítico. Inclui a leitura dos webhooks do cliente.
- Uma mudança nas preferências de eventos do webhook **não afeta eventos já enfileirados**.
- O cliente que precisa de itens faz uma segunda chamada (`GET /orders/:id`).

**Trade-off explícito:** pagamos um pouco mais de trabalho dentro da transação de pedidos em troca de eventos fiéis ao momento da mudança e de uma outbox sem linhas inúteis.
