# ADR-006 — Reuso máximo dos padrões existentes: módulo `src/modules/webhooks`, `AppError`, Pino, error middleware e `requireRole`

| Campo | Valor |
| --- | --- |
| **Status** | Aceito |
| **Data da decisão** | Reunião técnica de quinta-feira, 09:27–09:30, 09:36 e 09:40–09:41 |
| **Decisores** | Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma) |
| **Relacionados** | [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [FDD — Integração com o sistema existente](../FDD.md#13-integração-com-o-sistema-existente) |

## Contexto

O código base já tem convenções claras e consistentes:

| Padrão | Onde está no código |
| --- | --- |
| Um módulo por domínio com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts` | `src/modules/orders/`, `src/modules/customers/` etc. |
| Classe base de erro `AppError(message, statusCode, errorCode, details)` | [src/shared/errors/app-error.ts](../../src/shared/errors/app-error.ts) |
| Subclasses HTTP e de domínio com códigos em `UPPER_SNAKE_CASE` (`INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`) | [src/shared/errors/http-errors.ts](../../src/shared/errors/http-errors.ts) |
| Error middleware central que trata `AppError`, `ZodError` e erros Prisma (`P2002`, `P2025`) | [src/middlewares/error.middleware.ts](../../src/middlewares/error.middleware.ts) |
| Validação por schemas Zod via `validate({ body, query, params })` | [src/middlewares/validate.middleware.ts](../../src/middlewares/validate.middleware.ts) |
| Autenticação JWT (`authenticate`) e autorização por role (`requireRole('ADMIN')`) | [src/middlewares/auth.middleware.ts](../../src/middlewares/auth.middleware.ts), usado em [src/modules/users/user.routes.ts](../../src/modules/users/user.routes.ts) |
| Logger Pino com `redact` de campos sensíveis | [src/shared/logger/index.ts](../../src/shared/logger/index.ts) |
| Composição de dependências manual em `buildControllers` e montagem de rotas em `buildApiRouter` | [src/app.ts](../../src/app.ts), [src/routes/index.ts](../../src/routes/index.ts) |
| Operações transacionais recebendo `Prisma.TransactionClient` (`TxClient`) | `debitStock(tx, ...)` em [src/modules/orders/order.service.ts](../../src/modules/orders/order.service.ts) |
| IDs UUID (`@db.Char(36)`) em todas as tabelas | [prisma/schema.prisma](../../prisma/schema.prisma) |

O time é pequeno ([09:07] Diego). Introduzir frameworks, loggers ou convenções novas só para esta feature aumentaria o custo de manutenção.

## Decisão

O módulo de webhooks é **um módulo igual aos outros** ([09:30] Larissa):

1. **Estrutura:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais `webhook.processor.ts` (lógica do worker) e `webhook.publisher.ts` (a função `publishWebhookEvent`).
2. **Erros:** novas classes estendem as existentes (`NotFoundError`, `ValidationError`, `UnprocessableEntityError`, `ConflictError` e, portanto, `AppError`). Todos os códigos do módulo usam o prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
3. **Error middleware sem alteração:** como todos os erros novos são `AppError`, o middleware central já os serializa no formato `{ error: { code, message, details } }`.
4. **Logging:** Pino, a mesma instância `logger`. **Nenhum logger novo.**
5. **Autorização:** `authenticate` em todas as rotas do módulo. `requireRole('ADMIN')` só no replay da DLQ.
6. **Integração com pedidos:** uma **função pura** `publishWebhookEvent(tx, order, fromStatus, toStatus)`, chamada de dentro de `changeStatus` com o `tx` atual. O `OrderService` **não** recebe um repository de webhooks injetado ([09:41] Bruno, [09:41] Diego).
7. **Worker:** reutiliza a fábrica `createPrismaClient()` de [src/config/database.ts](../../src/config/database.ts) para abrir **uma instância própria** no processo do worker ([09:30] Bruno).
8. **IDs:** UUID em todas as novas tabelas ([09:51] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Injetar um `WebhookRepository` inteiro no `OrderService`** | Acopla o serviço de pedidos ao módulo de webhooks e muda o construtor usado em `buildControllers`. Uma função pura que recebe o `tx` basta ([09:41] Bruno, [09:41] Diego). |
| **Compartilhar o `PrismaClient` entre API e worker** | Impossível: são processos Node diferentes, e o client é por processo ([09:29] Diego, [09:30] Bruno). |
| **Estrutura própria para o módulo (ex.: pasta `workers/` na raiz, novo logger)** | Contraria a convenção do projeto e a decisão de "reuso máximo" ([09:29] Bruno, [09:30] Larissa). *(Plausível, não defendida por ninguém.)* |

## Consequências

**Positivas**

- Curva de aprendizado quase zero para quem já trabalha no projeto.
- O error middleware, o logging e a autenticação funcionam sem nenhuma mudança nesses arquivos.
- As respostas de erro seguem o mesmo contrato do resto da API.

**Negativas**

- O módulo herda as limitações atuais: não há métricas nem tracing distribuído no projeto ([package.json](../../package.json)), então a observabilidade inicial se apoia em logs estruturados (ver FDD).
- `changeStatus` passa a depender (por import) de uma função do módulo de webhooks. O acoplamento é mínimo, mas existe.

**Trade-off explícito:** priorizamos consistência e velocidade de entrega (3 sprints) em vez de adotar ferramentas mais especializadas, como filas dedicadas ou OpenTelemetry, que poderiam ser melhores isoladamente, mas custam mais para um time pequeno.
