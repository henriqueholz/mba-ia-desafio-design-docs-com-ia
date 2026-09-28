# Tracker de Rastreabilidade

Este tracker liga cada item do pacote de design docs à sua origem: uma fala da reunião ([TRANSCRICAO.md](../TRANSCRICAO.md)) ou um arquivo do código base.

- **Fonte `TRANSCRICAO`**: a localização é o timestamp e o falante, no formato `[hh:mm] Nome`. Quando uma decisão foi construída em várias falas, a linha aponta a fala **decisiva** e as demais aparecem no próprio documento.
- **Fonte `CODIGO`**: a localização é o caminho real do arquivo no repositório.
- Itens marcados como **(def. FDD)** nos documentos são escolhas de implementação feitas para tornar a especificação acionável. Aqui eles apontam para a fala ou o arquivo que motivou a escolha.

**Resumo de cobertura:** 258 itens rastreados · 211 com fonte `TRANSCRICAO` (81%) · 47 com fonte `CODIGO`. Todos os caminhos `CODIGO` foram verificados no repositório.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox: evento gravado na mesma transação SQL da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-DEC | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Decisão formal: "outbox em MySQL" | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Disparo HTTP síncrono no service de orders: trava a transação e não tem resposta para cliente fora do ar | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Redis Streams exigiria subir infraestrutura nova | TRANSCRICAO | [09:07] Larissa |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa descartada | Inserir na outbox fora da transação perde a garantia | TRANSCRICAO | [09:41] Diego |
| ADR-001-TO | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis Cluster é overengineering para time pequeno. A outbox no MySQL existente basta | TRANSCRICAO | [09:07] Diego |
| ADR-001-R-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status (pendente/processando/falhou/entregue) e created_at. Leitura em lote pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-R-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Fora de escopo | Arquivamento de linhas entregues (~30 dias) fora desta feature | TRANSCRICAO | [09:08] Diego |
| ADR-001-CTX-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | A transação de changeStatus atualiza orders, grava history e mexe no estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-CTX-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | A validação de transição usa canTransition | CODIGO | src/modules/orders/order.status.ts |
| ADR-001-CTX-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | A única infraestrutura persistente é o MySQL 8 | CODIGO | docker-compose.yml |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker lê a outbox por polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| ADR-002-DEC | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Registro formal: polling de 2 s, latência mínima de 2 s aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-D-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D-04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Mesmo banco, mesma stack, processo diferente | TRANSCRICAO | [09:11] Diego |
| ADR-002-D-05 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient separado, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D-06 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Single-worker, com ordem por created_at e, portanto, por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Trigger no MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa descartada | Múltiplos workers (particionar por order_id ou lock): problema do futuro | TRANSCRICAO | [09:13] Diego |
| ADR-002-TO | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Limitação conhecida: sem ordem global, só por order_id e com single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-002-TO-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Os clientes nunca pediram ordem global | TRANSCRICAO | [09:14] Marcos |
| ADR-002-CTX-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Entry-point atual único (bootstrap + shutdown) | CODIGO | src/server.ts |
| ADR-002-CTX-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Fábrica createPrismaClient e singleton prisma | CODIGO | src/config/database.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff exponencial e teto de tentativas, depois DLQ | TRANSCRICAO | [09:15] Diego |
| ADR-003-DEC | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Progressão 1m/5m/30m/2h/12h, total de quase 15 h | TRANSCRICAO | [09:17] Diego |
| ADR-003-D-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter separada (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-D-04 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay, que recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-003-D-05 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Restrição | O replay exige role ADMIN e registra quem executou | TRANSCRICAO | [09:36] Sofia |
| ADR-003-D-06 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Reaproveitar o requireRole existente | TRANSCRICAO | [09:36] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | Retry indefinido deixa evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | 3 tentativas (proposta mais agressiva) | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-02b | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | 3 tentativas cobrem só ~30 min, e já houve cliente com 2 h de manutenção | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa descartada | DLQ como status "failed" na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-TO | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Cliente fora por ~15 h: aceitável do ponto de vista de produto | TRANSCRICAO | [09:17] Marcos |
| ADR-003-C-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Integração | requireRole('ADMIN') para proteger o replay | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura HMAC com secret compartilhada no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 (padrão de mercado) | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint, não global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D-04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Configuração guarda url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D-05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação via API, com a secret antiga válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-DEC | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Consolidado: HMAC-SHA256 sobre o corpo, secret por endpoint, grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-CTX-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Webhook só outbound | TRANSCRICAO | [09:02] Marcos |
| ADR-004-CTX-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto | Já houve cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-004-R-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | URL precisa ser https, validação Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-R-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | Revisão de segurança de no mínimo 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa descartada | Secret global: se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once, e o cliente precisa estar preparado para duplicatas | TRANSCRICAO | [09:24] Diego |
| ADR-005-D-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | X-Event-Id com UUID gerado na entrada da outbox, para deduplicação | TRANSCRICAO | [09:25] Diego |
| ADR-005-DEC | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Registro formal: at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa descartada | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Joga responsabilidade para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-D-03 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Dependência | Documentar a deduplicação em destaque no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Lógica do worker em webhook.processor.ts dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Classes de erro baseadas em AppError, com códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL e WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo WEBHOOK_ para todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D-05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Pino e error middleware sem nada novo | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D-06 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa descartada | Compartilhar PrismaClient entre API e worker | TRANSCRICAO | [09:29] Diego |
| ADR-006-D-07 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | IDs UUID nas novas tabelas | TRANSCRICAO | [09:51] Larissa |
| ADR-006-C-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Classe base AppError(message, statusCode, errorCode, details) | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-C-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | InvalidStatusTransitionError, InsufficientStockError e códigos UPPER_SNAKE | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Error middleware trata AppError, ZodError, P2002 e P2025 | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-C-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Logger Pino com redact | CODIGO | src/shared/logger/index.ts |
| ADR-006-C-05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | validate({ body, query, params }) com Zod | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-C-06 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | requireRole('ADMIN') já em uso | CODIGO | src/modules/users/user.routes.ts |
| ADR-006-C-07 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Composição manual em buildControllers | CODIGO | src/app.ts |
| ADR-006-C-08 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Montagem de rotas em buildApiRouter | CODIGO | src/routes/index.ts |
| ADR-006-C-09 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | IDs UUID Char(36) em todos os models | CODIGO | prisma/schema.prisma |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Payload renderizado na inserção (snapshot) | TRANSCRICAO | [09:52] Larissa |
| ADR-007-DEC | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | "Snapshot. Decidido." | TRANSCRICAO | [09:52] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-D-02 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Campos do payload. Sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D-03 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Trade-off | Payload enxuto | TRANSCRICAO | [09:44] Bruno |
| ADR-007-D-04 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Filtrar na hora de enviar | TRANSCRICAO | [09:34] Diego |
| ADR-007-D-05 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Requisito Não Funcional | Limite de 64 KB, com erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| ADR-007-ALT-03 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Alternativa descartada | Truncar o payload grande | TRANSCRICAO | [09:23] Sofia |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores são os participantes da reunião, e Larissa conduziu | TRANSCRICAO | [09:00] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes B2B pedem notificação. Hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Requisito Não Funcional | "Tempo real" significa menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Prazo pedido pela Atlas: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-CTX-04 | docs/RFC.md | Contexto | changeStatus concentra a mudança de status em uma transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | customer_id no corpo ou no path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| RFC-PROP-02 | docs/RFC.md | Estimativa | 3 sprints incluindo a revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / broker dedicado | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do MySQL para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Worker dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | E-mail ao cliente após falhas: próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Escala horizontal do worker (partição por order_id ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Arquivamento da outbox (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Endurecer a autorização do CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05b | docs/RFC.md | Risco | O model User não tem vínculo com Customer, então não há checagem de posse | CODIGO | prisma/schema.prisma |
| RFC-Q-06 | docs/RFC.md | Questão em aberto | Proteção da secret em repouso e assinatura no grace period (revisão Sofia) | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | A criação do pedido (null → PENDING) não passa por changeStatus e não gera evento | CODIGO | src/modules/orders/order.service.ts |
| RFC-RISK-01 | docs/RFC.md | Risco | Cliente não deduplica. Mitigação: portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| RFC-NEXT-01 | docs/RFC.md | Próximo passo | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Status nunca muda sem evento, e não há evento de rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Latência p95 < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Todo evento termina entregue ou na DLQ em ~15 h | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Zero infraestrutura e dependências novas | TRANSCRICAO | [09:07] Diego |
| FDD-ESC-01 | docs/FDD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| FDD-ESC-02 | docs/FDD.md | Fora de escopo | Aviso por e-mail | TRANSCRICAO | [09:37] Larissa |
| FDD-ESC-03 | docs/FDD.md | Fora de escopo | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| FDD-ESC-04 | docs/FDD.md | Fora de escopo | Dashboard visual | TRANSCRICAO | [09:40] Larissa |
| FDD-ESC-05 | docs/FDD.md | Fora de escopo | Evento na criação do pedido, que grava o history direto em create() | CODIGO | src/modules/orders/order.service.ts |
| FDD-ESC-06 | docs/FDD.md | Fora de escopo | Listagem da DLQ: só o replay foi pedido | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | webhook_endpoints: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | webhook_outbox com status PENDING/PROCESSING/DELIVERED/FAILED e índices | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Uma linha por (evento, endpoint), identificada por X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | payload serializado (snapshot) em MediumText | TRANSCRICAO | [09:52] Larissa |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | webhook_deliveries: sucesso/falha, resposta, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados | IDs UUID Char(36) | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-08 | docs/FDD.md | Integração | Novos models e back-relation em Customer | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | publishWebhookEvent chamado dentro da transação de changeStatus, com rollback conjunto | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro por events contendo to_status. Nenhum interessado = nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | event_id UUID gerado na inserção | TRANSCRICAO | [09:25] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Payload acima de 64 KB lança erro, sem truncar | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Worker: claim dos PENDING mais antigos em lote, a cada 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Envio sequencial em ordem de created_at | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Worker com PrismaClient próprio via createPrismaClient() | TRANSCRICAO | [09:30] Bruno |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Bootstrap e shutdown do worker no padrão existente | CODIGO | src/server.ts |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Retry: BACKOFF_SCHEDULE 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo | Movimentação para a DLQ ao esgotar as tentativas | TRANSCRICAO | [09:15] Diego |
| FDD-FLUXO-11 | docs/FDD.md | Fluxo | Replay recoloca a linha da outbox como PENDING | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo | Log de auditoria do replay com replayedBy | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-13 | docs/FDD.md | Fluxo | Rotação: secret anterior válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-14 | docs/FDD.md | Fluxo | Recuperação PROCESSING → PENDING no startup (at-least-once) (def. FDD) | TRANSCRICAO | [09:24] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | Corpo JSON com event_type "order.status_changed" e campos do pedido | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | Formato order_number ORD-000000 | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks (url, events, customerId). Secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | customerId no body, não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | GET /webhooks para listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Paginação no formato paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-14 | docs/FDD.md | Restrição | CRUD liberado a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTRATO-15 | docs/FDD.md | Restrição | Valores válidos de events são os destinos de transição (PENDING excluído) | CODIGO | src/modules/orders/order.status.ts |
| FDD-CONTRATO-16 | docs/FDD.md | Contrato | Base /api/v1 e formato de erro { error: { code, message, details } } | CODIGO | src/app.ts |
| FDD-CONTRATO-17 | docs/FDD.md | Contrato | Timeout de 10 s. Não-2xx é falha | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (http recusado) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED: nunca envia sem assinatura | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | Prefixo WEBHOOK_ em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (422) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-07 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED leva à DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-08 | docs/FDD.md | Integração | NotFoundError e ValidationError fixam o código. Os erros com código próprio estendem AppError ou BadRequest/Conflict/UnprocessableEntity | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Nenhuma chamada HTTP dentro da transação de pedido | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-02 | docs/FDD.md | Resiliência | Fallback: DLQ + replay manual. Sem e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Worker fora do ar: eventos acumulam na outbox sem perda | TRANSCRICAO | [09:06] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Lote pequeno por ciclo | TRANSCRICAO | [09:08] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs via Pino, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Adicionar *.secret e *.previousSecret ao redact | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Motivação do redact: secret vazada em log | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Métrica de latência p95 < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Sem lib de métricas/tracing no projeto: métricas derivadas de logs e SQL | CODIGO | package.json |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Log de auditoria webhook_dlq_replayed | TRANSCRICAO | [09:36] Sofia |
| FDD-SEG-01 | docs/FDD.md | Segurança | Revisão da Sofia (geração de secret e HMAC) | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >= 20 (fetch e AbortSignal.timeout nativos) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | uuid já usado para request id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEP-03 | docs/FDD.md | Dependência | MySQL 8 existente | CODIGO | docker-compose.yml |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent após orderStatusHistory.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Destinos do mapa transitions definem os events válidos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Base AppError para os erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Error middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate converte ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildControllers ganha os controllers de webhook | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Montagem de /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | createPrismaClient() no worker | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Novas variáveis WEBHOOK_* no envSchema | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Scripts worker / worker:dev espelhando start/dev | CODIGO | package.json |
| FDD-INT-12 | docs/FDD.md | Integração | tsconfig de build já inclui src/**/*.ts (compila src/worker.ts) | CODIGO | tsconfig.build.json |
| FDD-INT-13 | docs/FDD.md | Integração | Limpeza das novas tabelas antes de customer.deleteMany() | CODIGO | tests/setup.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Padrão de router com router.use(authenticate) | CODIGO | src/modules/orders/order.routes.ts |
| FDD-CA-02 | docs/FDD.md | Critério de aceite | Falha na outbox não altera status, history nem estoque | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-07 | docs/FDD.md | Critério de aceite | Replay: OPERATOR → 403, ADMIN → 202 com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-15 | docs/FDD.md | Critério de aceite | Worker sobrevive ao restart da API | TRANSCRICAO | [09:11] Diego |
| FDD-RISCO-01 | docs/FDD.md | Risco | Ordem quebrada durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISCO-02 | docs/FDD.md | Risco | Usuário gerencia webhook de customer alheio (aceito nesta fase) | TRANSCRICAO | [09:37] Sofia |
| FDD-RISCO-03 | docs/FDD.md | Risco | Estouro do prazo | TRANSCRICAO | [09:46] Larissa |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Máquina de estados PENDING→PAID→PROCESSING→SHIPPED→DELIVERED / CANCELLED | CODIGO | src/modules/orders/order.status.ts |
| PRD-PROB-01 | docs/PRD.md | Problema | Polling em GET /orders é lento e caro | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Motivação | Risco de a Atlas migrar para o concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-PERS-01 | docs/PRD.md | Público-alvo | Usuários do sistema que representam o cliente, autenticados por JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-CEN-01 | docs/PRD.md | Cenário | Assinatura só de SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Manutenção planejada de 2 h | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário | Secret vazada e rotação | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-1 | docs/PRD.md | Métrica | p95 < 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-2 | docs/PRD.md | Métrica | 3 de 3 clientes até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-3 | docs/PRD.md | Métrica | 0 mudanças sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-4 | docs/PRD.md | Métrica | 0 eventos sem estado final após ~15 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-5 | docs/PRD.md | Métrica | 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de aviso: adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual: projeto do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting: observar | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Ordem global / múltiplos workers | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint (url, status, customer) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Requisito Funcional | customer_id no corpo ou no path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar endpoints de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de status por endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificar a cada mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Conteúdo da notificação (sem itens) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Notificação assinada e identificada | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação da secret com 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry automático em 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | DLQ com payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay manual por ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Recusar URL sem HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s, polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade status/evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Isolamento da disponibilidade do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com id único | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB, com erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256, secret por endpoint, TLS | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordem por pedido com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Processo de entrega separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Seguir os padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de ≥ 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal do desenvolvedor (integração via API) | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Comunicação do prazo aos clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Sessão de revisão do design | TRANSCRICAO | [09:50] Larissa |
| PRD-RISCO-01 | docs/PRD.md | Risco | Churn da Atlas por atraso | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Vazamento de secret | TRANSCRICAO | [09:21] Sofia |
| PRD-RISCO-04 | docs/PRD.md | Risco | Cliente fora > 15 h sem aviso | TRANSCRICAO | [09:17] Marcos |
| PRD-RISCO-05 | docs/PRD.md | Risco | Rajada de 50 eventos por minuto | TRANSCRICAO | [09:38] Diego |
| PRD-RISCO-06 | docs/PRD.md | Risco | Eventos fora de ordem em retries | TRANSCRICAO | [09:14] Marcos |
| PRD-TESTE-01 | docs/PRD.md | Estratégia de teste | Testes ponta a ponta na integração com order.service | TRANSCRICAO | [09:46] Larissa |
| PRD-TESTE-02 | docs/PRD.md | Estratégia de teste | Testes de integração com Vitest + MySQL (padrão atual) | CODIGO | tests/setup.ts |
| PRD-PLANO-01 | docs/PRD.md | Plano | Sprint 1 outbox/DLQ, sprint 2 worker/retry, sprint 3 CRUD/integração/HMAC + revisão | TRANSCRICAO | [09:46] Larissa |
