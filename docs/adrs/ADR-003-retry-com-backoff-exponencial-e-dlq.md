# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:14–09:19 e 09:35–09:36 |
| **Decisores** | Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança) |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md), [RFC](../RFC.md) |

## Contexto

Os endpoints dos clientes podem ficar fora do ar. A empresa já teve cliente com **duas horas de indisponibilidade** em manutenção planejada ([09:16] Diego). Precisamos insistir o bastante para cobrir esse tipo de janela, sem deixar eventos pendurados para sempre quando o cliente some. Também precisamos de um lugar para investigar e reprocessar o que falhou de vez.

## Decisão

1. **Backoff exponencial com teto de 5 retentativas.** Intervalos após cada falha: **1 min, 5 min, 30 min, 2 h, 12 h**. São cerca de 14h36min (a reunião arredondou para "quase 15 horas") entre a primeira falha e a última tentativa.
   - *Interpretação registrada:* como a soma dos 5 intervalos é o que dá "quase 15 horas", o modelo é **1 tentativa inicial + 5 retentativas**, com no máximo 6 chamadas HTTP por evento.
2. **Falha** é qualquer resposta que não seja 2xx, erro de rede ou timeout de 10 s (ver [FDD](../FDD.md#9-estratégias-de-resiliência)).
3. **DLQ em tabela separada `webhook_dead_letter`.** Esgotadas as retentativas, o evento vai para a DLQ com payload, motivo da falha e timestamp. A linha da outbox é marcada como `FAILED`.
4. **Replay manual por endpoint admin:** `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como `PENDING`.
   - Exige **role `ADMIN`**, reaproveitando `requireRole` de [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts).
   - O replay **registra em log quem executou**, para auditoria ([09:36] Sofia).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Retry indefinido com backoff** | Evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Três retentativas cobririam só uns 30 minutos. Uma manutenção de 2 h do cliente já mataria o evento ([09:16] Bruno, [09:16] Diego). |
| **DLQ como status `failed` na própria outbox** | Mistura eventos vivos com mortos na tabela que o worker lê a cada 2 s. A tabela separada deixa a leitura da outbox mais limpa e guarda evidência para debug e reprocessamento ([09:17] Larissa, [09:18] Diego). |
| **Reprocessamento automático da DLQ** | Não foi proposto. O replay é deliberadamente manual e restrito a `ADMIN`, porque mexer na fila de entrega "não é coisa de operador" ([09:36] Sofia). |

## Consequências

**Positivas**

- Cobre indisponibilidades de até cerca de 15 h, incluindo manutenções planejadas de algumas horas.
- Nenhum evento fica pendente para sempre: todo evento termina em `DELIVERED` ou na DLQ.
- A DLQ serve de evidência de falha (payload + motivo) e permite reprocessar de forma controlada e auditada.

**Negativas**

- Um cliente que volta depois de cerca de 15 h perde eventos até alguém fazer replay manual. Não haverá aviso automático por e-mail nesta fase (adiado, [09:37] Larissa).
- O backoff atrasa eventos do mesmo pedido que vêm depois. Isso reforça a limitação de ordem do [ADR-002](ADR-002-worker-separado-em-polling.md).
- Mais uma tabela e mais um endpoint para manter.

**Trade-off explícito:** aceitamos uma janela finita (cerca de 15 h) e intervenção humana para o que passa dela, em troca de uma outbox que sempre drena e de um processo de falha auditável.
