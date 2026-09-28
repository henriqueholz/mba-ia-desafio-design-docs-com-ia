# ADR-005 — Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:24–09:26 |
| **Decisores** | Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM) |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md), [RFC](../RFC.md) |

## Contexto

A combinação outbox + worker + retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) pode, por natureza, entregar o mesmo evento mais de uma vez. Exemplos:

- o cliente processa a requisição, mas a resposta chega depois do timeout de 10 s;
- o worker cai entre o envio HTTP e a marcação `DELIVERED`;
- um admin faz replay de um evento da DLQ que o cliente já tinha recebido parcialmente.

Precisamos definir qual garantia a plataforma oferece e como o cliente lida com duplicatas.

## Decisão

1. A plataforma garante **at-least-once**: todo evento registrado na outbox é entregue uma ou mais vezes, até ser entregue ou esgotar os retries e ir para a DLQ.
2. Cada evento recebe um **UUID (`event_id`) gerado no momento em que entra na outbox**. Ele é enviado no header `X-Event-Id` e repetido no corpo (`event_id`). O valor é **estável entre retries e replays**.
3. **A deduplicação é responsabilidade do cliente**, pelo `event_id`.
4. O produto (Marcos) documenta essa responsabilidade **em destaque no portal do desenvolvedor**.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Exactly-once** | Exigiria coordenação dos dois lados (confirmações, estado compartilhado). É muito mais complexo, e at-least-once com `event_id` resolve cerca de 99% dos casos ([09:25] Diego). |
| **At-most-once (envia uma vez, sem retry)** | Contradiz a política de retry decidida no [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) e perderia eventos a cada indisponibilidade do cliente. *(Alternativa plausível, não debatida.)* |

## Consequências

**Positivas**

- Modelo simples, igual ao de provedores de mercado (Stripe e GitHub fazem assim, [09:25] Diego).
- O worker pode ser conservador: na dúvida, reenvia. Isso simplifica a recuperação de falhas, como resetar eventos presos em `PROCESSING` após um crash.

**Negativas**

- **Transfere responsabilidade para o cliente** ([09:25] Sofia). Um cliente que não deduplica pode processar o mesmo evento duas vezes.
- Exige comunicação clara no portal do desenvolvedor, que é uma dependência de produto.

**Trade-off explícito:** trocamos a garantia mais forte (exactly-once) por simplicidade de implementação, e em troca o cliente precisa implementar deduplicação pelo `X-Event-Id`.
