# ADR-002 — Worker em processo separado, lendo a outbox por polling a cada 2 segundos

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:08–09:14 e 09:29–09:30 |
| **Decisores** | Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos) |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [RFC](../RFC.md) |

## Contexto

Com a outbox definida no [ADR-001](ADR-001-outbox-no-mysql.md), alguém precisa ler os eventos pendentes e fazer as chamadas HTTP. Os clientes consideram "tempo real" qualquer coisa **abaixo de 10 segundos** ([09:02] Marcos).

Hoje o projeto tem um único entry-point de processo, [src/server.ts](../../src/server.ts). Ele cria o app Express e usa o `PrismaClient` singleton de [src/config/database.ts](../../src/config/database.ts). O MySQL não tem um mecanismo nativo de notificação para processos externos como o `LISTEN/NOTIFY` do Postgres.

## Decisão

1. **Polling em loop a cada 2 segundos.** A cada ciclo, o worker busca os eventos `PENDING` mais antigos (por `created_at`) em lote pequeno, processa e marca o resultado. Pior caso de latência: cerca de 2 s mais o tempo de envio, dentro da meta de 10 s.
2. **Processo separado da API.** Novo entry-point `src/worker.ts`, iniciado por um script `npm run worker`. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts` (a reunião citou `webhook.worker.ts` ou `webhook.processor.ts`).
3. **Mesmo banco e mesma stack, com `PrismaClient` próprio.** O worker usa a mesma `DATABASE_URL` validada por [src/config/env.ts](../../src/config/env.ts), mas instancia seu próprio client, porque `PrismaClient` é por processo.
4. **Single-worker.** Uma única instância do worker. A ordem de entrega segue `created_at` e, por consequência, é preservada **por `order_id`**. Não há garantia de ordem global.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Trigger no banco para acordar o worker** | Trigger no MySQL só executa SQL e não notifica processos externos. Avisar o worker exigiria gambiarras, como escrever em arquivo ou chamar um endpoint ([09:09] Bruno, [09:09] Diego). |
| **Worker dentro do processo da API** | Se a API reinicia, o worker morre junto. As duas cargas também passam a disputar os mesmos recursos ([09:11] Diego). |
| **Múltiplos workers em paralelo** | Perde-se a ordem por pedido. Particionar por `order_id` ou usar lock pessimista resolveria, mas é "problema do futuro" ([09:12]–[09:13] Diego). |

## Consequências

**Positivas**

- Implementação simples, sem dependências novas, testável isoladamente.
- A API e o worker reiniciam e fazem deploy de forma independente.
- A latência de 2 s atende com folga o requisito de negócio de menos de 10 s.

**Negativas**

- Consultas constantes ao banco a cada 2 s, mesmo sem eventos pendentes. O custo é baixo por causa do índice em `status`/`created_at`.
- **Limitação conhecida:** a ordem vale só por `order_id` e só enquanto houver um único worker. Um evento em backoff de retry (ver [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) pode ser ultrapassado por um evento posterior do mesmo pedido. Isso é aceito, porque os clientes nunca pediram ordem global ([09:14] Marcos).
- Escalar horizontalmente exigirá uma nova decisão (particionamento ou lock).
- O deploy passa a ter um segundo processo para operar e monitorar.

**Trade-off explícito:** trocamos reatividade imediata e escala horizontal por simplicidade operacional e ordem por pedido, com um piso de cerca de 2 s de latência.
