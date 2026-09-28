# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Henrique Castro |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião, em detalhamento |
| **Data** | 2026-09-28 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS vai **notificar clientes B2B automaticamente, via webhook HTTP, sempre que o status de um pedido deles mudar**. O cliente cadastra uma URL HTTPS, escolhe quais status quer receber e passa a receber uma chamada assinada a cada mudança relevante. Isso substitui o polling que os clientes fazem hoje.

A demanda veio de um pedido formal de três clientes B2B: **Atlas Comercial, MaxDistribuição e Nova Cargo**. O OMS tem uma máquina de estados de pedido (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED`), mas **nenhum mecanismo de notificação externa**.

## 2. Problema e motivação

- **Problema:** para saber se um pedido mudou, o cliente consulta `GET /orders` periodicamente. A integração fica **lenta** (a mudança só é percebida no próximo ciclo de polling) e **cara** para o cliente ([09:00] Marcos).
- **Motivação de negócio:** a Atlas sinalizou que pode **migrar para um concorrente** se a funcionalidade não for entregue até o fim do trimestre. O prazo alinhado depois foi o **fim de novembro** ([09:00], [09:45] Marcos).
- **Expectativa do cliente:** "tempo real" significa **menos de 10 segundos**, sem precisar atualizar manualmente ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Quem é | O que precisa |
| --- | --- | --- |
| **Integrador do cliente B2B** | Time técnico da Atlas, MaxDistribuição ou Nova Cargo | Receber eventos confiáveis e verificáveis, escolher quais status recebe e depurar entregas |
| **Usuário da plataforma que representa o cliente** | Usuário do OMS autenticado por JWT, que opera a API em nome do cliente ([09:32] Marcos) | Cadastrar, editar, listar e remover endpoints e rotacionar a secret |
| **Administrador da plataforma** | Usuário com role `ADMIN` | Reprocessar eventos que falharam de vez, com trilha de auditoria |

**Cenários**

1. **Assinatura seletiva.** A Atlas cadastra `https://…/oms-events` para ouvir só `SHIPPED` e `DELIVERED` ([09:33] Marcos). Quando um pedido dela é despachado, recebe a notificação em segundos. Mudanças para `PAID` não geram chamada.
2. **Manutenção planejada.** O endpoint da MaxDistribuição fica fora do ar por 2 horas. Os eventos do período são reentregues automaticamente quando ele volta ([09:16] Diego).
3. **Secret vazada.** A Nova Cargo descobre que a secret apareceu num log dela, pede uma nova pela API e tem 24 h para atualizar os sistemas sem perder entregas ([09:21] Sofia, [09:22] Diego).
4. **Depuração.** O integrador consulta as últimas 100 entregas do endpoint, com status, resposta e tempo, para entender por que um evento não foi processado ([09:34] Marcos).
5. **Indisponibilidade longa.** Um cliente fica fora por mais de cerca de 15 h. Os eventos vão para a fila de falhas definitivas, e um administrador os reprocessa manualmente quando o cliente volta ([09:18] Diego).
6. **Evento duplicado.** O cliente recebe o mesmo evento duas vezes e o descarta pelo identificador único do evento ([09:25] Diego).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| OBJ-1 | Notificação em "tempo real" | Latência entre a mudança de status e a entrega bem-sucedida (p95) | **< 10 s** (piso esperado de cerca de 2 s) | [09:02] Marcos, [09:10] Larissa |
| OBJ-2 | Reter os clientes solicitantes | Clientes solicitantes com webhook em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) **até o fim de novembro** | [09:00], [09:45] Marcos |
| OBJ-3 | Nenhuma notificação perdida por inconsistência | Mudanças de status com webhook interessado e sem evento registrado | **0** | [09:40] Bruno |
| OBJ-4 | Toda falha tem desfecho | Eventos sem estado final (entregue ou DLQ) | **0** após a janela de retry (~15 h) | [09:17] Diego |
| OBJ-5 | Entrega no prazo planejado | Sprints até a produção | **3 sprints**, incluindo a revisão de segurança | [09:46]–[09:47] Larissa |

> A redução do polling em `GET /orders` é o benefício esperado para o cliente. Não há baseline medido na reunião, então fica como indicador a acompanhar, sem meta numérica.

## 5. Escopo

### 5.1 Incluso

- Webhooks **outbound** (plataforma → cliente) para mudanças de status de pedido.
- CRUD de configuração de endpoints por cliente, com filtro de status.
- Secret por endpoint, gerada pela plataforma, com rotação.
- Retry automático, fila de falhas definitivas (DLQ) e reprocessamento manual por `ADMIN`.
- Histórico de entregas por endpoint.

### 5.2 Fora de escopo

| Item | Situação | Origem |
| --- | --- | --- |
| **Aviso por e-mail** ao cliente quando o webhook falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Larissa |
| **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta fase: é projeto separado do time de frontend | [09:40] Larissa |
| **Rate limiting** de envio para o cliente | **Adiado**: "observar e decidir depois" | [09:39] Larissa |
| **Webhooks inbound** (cliente enviando para nós) | **Descartado**: o cliente só quer receber | [09:02] Marcos |
| **Garantia de ordem global** e múltiplos workers | **Adiado**: ordem garantida só por pedido | [09:13] Larissa, [09:14] Marcos |
| **Arquivamento** de eventos entregues (~30 dias) | **Fora** desta feature | [09:08] Diego |
| Permissões mais restritas no CRUD de webhook | **Adiado**: "mais pra frente a gente pode endurecer" | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Prioridade | Origem |
| --- | --- | --- | --- |
| PRD-FR-01 | O usuário autenticado **cadastra um endpoint de webhook** informando `url`, lista de status desejados e o cliente (`customer_id` no corpo ou no path, **não** extraído do JWT) | Must | [09:31] Marcos, [09:32] Larissa |
| PRD-FR-02 | A plataforma **gera a secret** do endpoint e a devolve na criação | Must | [09:31] Marcos |
| PRD-FR-03 | O usuário **edita** um endpoint (URL, status de interesse, ativo/inativo) | Must | [09:33] Bruno |
| PRD-FR-04 | O usuário **remove** um endpoint | Must | [09:33] Bruno |
| PRD-FR-05 | O usuário **lista** os endpoints de um cliente | Must | [09:33] Bruno |
| PRD-FR-06 | Cada endpoint recebe **só os status que escolheu**. Status fora da lista não geram notificação | Must | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-07 | A cada **mudança de status** de pedido, cada endpoint interessado do cliente dono do pedido recebe uma notificação HTTP | Must | [09:00] Marcos, [09:40] Bruno |
| PRD-FR-08 | A notificação contém id e tipo do evento, data/hora, id e número do pedido, status anterior e novo, cliente e valor total. **Não contém os itens** | Must | [09:43] Diego |
| PRD-FR-09 | Toda notificação é **assinada**, para o cliente validar a origem e a integridade, e identifica o evento e o endpoint | Must | [09:20] Sofia, [09:44] Diego, [09:44] Sofia |
| PRD-FR-10 | O usuário **rotaciona a secret** pela API. A secret anterior continua válida por **24 h** | Must | [09:21] Sofia |
| PRD-FR-11 | Se a entrega falha, a plataforma **tenta de novo automaticamente** até 5 vezes, com intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h) | Must | [09:17] Larissa |
| PRD-FR-12 | Esgotadas as tentativas, o evento vai para uma **fila de falhas definitivas (DLQ)** com o payload e o motivo | Must | [09:18] Diego |
| PRD-FR-13 | Um **administrador** (role `ADMIN`) **reprocessa** manualmente um evento da DLQ | Must | [09:18] Diego, [09:36] Sofia |
| PRD-FR-14 | O reprocessamento **registra quem o executou**, para auditoria | Must | [09:36] Sofia |
| PRD-FR-15 | O usuário consulta o **histórico das últimas 100 entregas** de um endpoint: sucesso/falha, payload, resposta e tempo de resposta | Must | [09:34] Marcos |
| PRD-FR-16 | O cadastro **recusa URL sem HTTPS** com erro de validação | Must | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-NFR-01 | **Latência** < 10 s entre a mudança de status e a entrega, com o worker verificando novos eventos a cada 2 s | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | **Atomicidade**: se o status mudou, o evento está registrado. Se a mudança falhou, o evento não existe | [09:06] Diego, [09:40] Bruno |
| PRD-NFR-03 | **Isolamento**: lentidão ou indisponibilidade do cliente não afeta a mudança de status de nenhum pedido | [09:04] Bruno |
| PRD-NFR-04 | **Entrega at-least-once**, com identificador único do evento para deduplicação pelo cliente | [09:24]–[09:26] Diego, Larissa |
| PRD-NFR-05 | **Timeout** de 10 s por chamada ao cliente | [09:42] Diego |
| PRD-NFR-06 | **Tamanho máximo do payload** de 64 KB. Acima disso é erro, e o evento não é truncado | [09:24] Larissa |
| PRD-NFR-07 | **Segurança**: assinatura HMAC-SHA256, secret única por endpoint e TLS obrigatório | [09:22] Sofia, [09:23] Sofia |
| PRD-NFR-08 | **Ordem** garantida só por pedido e enquanto houver um único processo de entrega. Não há ordem global | [09:13] Larissa |
| PRD-NFR-09 | **Disponibilidade independente**: o processo de entrega roda separado da API e sobrevive a reinícios dela | [09:11] Diego |
| PRD-NFR-10 | **Sem infraestrutura nova**: mesmo MySQL e mesma stack | [09:07] Diego |
| PRD-NFR-11 | **Manutenibilidade**: segue os padrões do projeto (módulos, erros, logger, middlewares) | [09:30] Larissa |

## 8. Decisões e trade-offs principais

Resumo para produto. A justificativa técnica completa está em cada ADR.

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Evento registrado na mesma transação da mudança de status (outbox no MySQL) | Latência de segundos em vez de envio imediato, em troca de nunca perder evento e não criar infraestrutura | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Processo de entrega separado, verificando a cada 2 s | Piso de ~2 s de latência e ordem garantida só por pedido | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15 h e depois DLQ com replay manual | Clientes fora por mais de 15 h dependem de ação de um admin | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Assinatura HMAC com secret por endpoint e rotação de 24 h | Mais complexidade de gestão de secrets, em troca de conter vazamentos | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | O cliente precisa deduplicar | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Sem ferramentas especializadas (métricas e tracing) na fase 1 | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload enxuto, congelado no momento da mudança | O cliente consulta `GET /orders/:id` se quiser os itens | [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

| Dependência | Responsável | Por quê | Origem |
| --- | --- | --- | --- |
| **Revisão de segurança** (HMAC e geração de secret), no mínimo 2 dias úteis antes do deploy | Sofia | Condição para ir a produção | [09:46] Sofia |
| **Documentação no portal do desenvolvedor**: integração via API e deduplicação por `event_id` em destaque | Marcos | O cliente precisa saber verificar a assinatura e deduplicar | [09:26], [09:40] Marcos |
| **Comunicação de prazo** aos clientes | Marcos | Alinhar expectativa da Atlas (fim de novembro) | [09:47], [09:49] Marcos |
| **Sessão de revisão do design** com Bruno e Diego antes de codar | Larissa | Validar o desenho antes da implementação | [09:50] Larissa |
| **Endpoint HTTPS do cliente** capaz de verificar HMAC e deduplicar | Clientes B2B | Sem isso a integração não funciona | [09:20] Sofia, [09:25] Diego |
| MySQL e stack atuais (Prisma, Node) | Plataforma | Base da solução, sem infraestrutura nova | [09:07] Diego, [09:11] Diego |

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação | Origem |
| --- | --- | --- | --- | --- |
| **Atlas migrar para o concorrente** por atraso | Média | Alto | Escopo fechado. Estimativa de 3 sprints com revisão incluída. Marcos confirma o prazo com o cliente | [09:00], [09:45]–[09:47] |
| **Cliente não deduplica** e processa eventos repetidos | Média | Médio | Identificador único por evento. Destaque no portal do desenvolvedor | [09:25] Sofia, [09:26] Marcos |
| **Vazamento da secret** pelo cliente | Média | Alto | Secret por endpoint (vazamento isolado). Rotação self-service com 24 h de convivência | [09:21]–[09:22] Sofia, Diego |
| **Cliente fora por mais de ~15 h perde eventos** sem ser avisado | Baixa | Médio | DLQ com replay manual por admin. Aviso por e-mail planejado para a próxima fase | [09:17] Marcos, [09:37] Larissa |
| **Rajada de eventos sobrecarrega o cliente** (ex.: 50 mudanças em 1 min) | Baixa | Médio | Monitorar. Rate limiting fica como decisão futura | [09:38]–[09:39] Diego, Larissa |
| **Eventos do mesmo pedido fora de ordem** durante retries | Baixa | Baixo | Limitação documentada. O payload traz status anterior e novo, e os clientes não pediram ordem global | [09:13] Larissa, [09:14] Marcos |

## 11. Critérios de aceitação

- [ ] Um usuário autenticado cadastra, lista, edita e remove endpoints de um cliente, e recebe a secret na criação (FR-01 a FR-05).
- [ ] Um endpoint configurado para `SHIPPED` e `DELIVERED` recebe notificação só nessas mudanças (FR-06, FR-07).
- [ ] A notificação chega em **menos de 10 s** (p95) com os campos acordados, sem itens, assinada e com identificadores de evento e de endpoint (FR-08, FR-09, NFR-01).
- [ ] Um cadastro com URL `http://` é recusado (FR-16).
- [ ] Depois de rotacionar a secret, entregas assinadas com a secret anterior continuam verificáveis por 24 h (FR-10).
- [ ] Com o endpoint do cliente fora do ar, o evento é reentregue quando ele volta, dentro da janela de ~15 h. Passada a janela, o evento aparece na DLQ (FR-11, FR-12).
- [ ] Só um `ADMIN` reprocessa eventos da DLQ, e o registro mostra quem fez (FR-13, FR-14).
- [ ] O histórico mostra as últimas 100 entregas com resultado, payload, resposta e tempo (FR-15).
- [ ] Se o registro do evento falha, o status do pedido não muda (NFR-02).
- [ ] A revisão de segurança da Sofia foi concluída antes do deploy.

Os critérios técnicos detalhados estão no [FDD §14](FDD.md#14-critérios-de-aceite-técnicos).

## 12. Estratégia de testes e validação

| Camada | O que valida | Responsável |
| --- | --- | --- |
| **Unitários** | Cálculo do backoff, assinatura HMAC (inclusive durante a rotação), montagem do payload e limite de 64 KB | Engenharia |
| **Integração** (Vitest + MySQL, padrão atual do projeto) | Atomicidade com a mudança de status, filtro de eventos, CRUD, deliveries, permissões do replay | Engenharia |
| **Ponta a ponta** | Integração no serviço de pedidos até o recebimento num endpoint HTTPS de teste, que prevê "testes ponta a ponta" no plano de sprints | Engenharia ([09:46] Larissa) |
| **Revisão de segurança** | Geração de secret e HMAC, no mínimo 2 dias úteis | Sofia ([09:46]) |
| **Validação pós-lançamento** | Acompanhar OBJ-1 (p95 < 10 s), OBJ-3 e OBJ-4 pelos logs e consultas descritos no [FDD §10](FDD.md#10-observabilidade) | Engenharia + Produto |

**Plano de entrega** ([09:46] Larissa):

- Sprint 1: modelagem de outbox e DLQ.
- Sprint 2: worker e retry.
- Sprint 3: CRUD e deliveries (meio sprint) + integração com pedidos e testes ponta a ponta (meio sprint) + HMAC, schemas e validações + revisão da Sofia no fim.
