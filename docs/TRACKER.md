# TRACKER — Rastreabilidade dos Documentos

Este tracker mapeia cada item registrado em [PRD](./PRD.md), [RFC](./RFC.md), [FDD](./FDD.md) e nos [ADRs](./adrs/) à sua origem: uma fala específica da transcrição (`TRANSCRICAO.md`, no formato `[hh:mm] Falante`) ou um arquivo real do código base (`Fonte = CODIGO`).

Regras aplicadas:

1. Nenhum requisito, decisão ou restrição entra nos documentos sem linha correspondente aqui — quando a localização não pode ser preenchida, o item é removido ou ajustado.
2. Itens marcados com **(derivado)** são especificações que a equipe não detalhou literalmente na reunião, mas que decorrem diretamente da fala citada (ex.: nome exato de um código de erro dentro do padrão `WEBHOOK_` definido por Bruno em [09:28]). Nenhum derivado contradiz a fala de origem.
3. Datas, metas e números quantitativos são sempre rastreados à fala que os originou.

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram formalmente notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes hoje fazem polling no GET /orders — integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | "Tempo real" para os clientes = latência abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-04 | docs/PRD.md | Risco de negócio | Atlas pode migrar para o concorrente sem entrega até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-05 | docs/PRD.md | Decisão de escopo | Webhooks outbound apenas — clientes não enviam eventos para nós | TRANSCRICAO | [09:02] Sofia/Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | 1ª tentativa de entrega < 10s para 100% dos eventos (polling de 2s) | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Migrar os 3 clientes B2B de polling para webhook | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Eventos recuperados na janela de retry de ~15h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | 100% dos eventos não entregues retidos na DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Entrega em ≤ 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa; [09:46] Sofia |
| PRD-FORA-01 | docs/PRD.md | Exclusão de escopo | Email de alerta por 3 falhas seguidas — fora de escopo, próxima fase | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-FORA-02 | docs/PRD.md | Exclusão de escopo | Dashboard visual para o cliente — projeto separado do frontend | TRANSCRICAO | [09:40] Larissa; [09:39] Marcos |
| PRD-FORA-03 | docs/PRD.md | Exclusão de escopo | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Larissa; [09:39] Diego |
| PRD-FORA-04 | docs/PRD.md | Exclusão de escopo | Arquivamento da outbox (~30 dias) — fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-05 | docs/PRD.md | Exclusão de escopo | Escala multi-worker (partição por order_id, lock pessimista) — futuro | TRANSCRICAO | [09:13] Diego |
| PRD-FORA-06 | docs/PRD.md | Exclusão de escopo | Webhooks inbound — só saída | TRANSCRICAO | [09:02] Marcos/Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (POST): url, lista de status, customerId no body/path; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos; [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Evento registrado na outbox na mesma transação da mudança de status; falha ⇒ rollback | TRANSCRICAO | [09:06] Diego; [09:40] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno/Marcos/Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega assíncrona por worker separado com polling de 2s | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry com backoff 1m/5m/30m/2h/12h, 5 retentativas (~15h) | TRANSCRICAO | [09:15] Diego; [09:17] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | DLQ persistida (payload, motivo, timestamp) + replay manual admin com auditoria | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:20] Sofia; [09:44] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimas 100: sucesso/falha, payload, response, tempo de resposta) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Payload com event_id, event_type, timestamp ISO, order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | CRUD para qualquer role autenticada; replay exclusivo de ADMIN | TRANSCRICAO | [09:36] Sofia/Larissa; [09:37] Marcos/Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência da 1ª tentativa < 10s (polling 2s) | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório — URL https; http recusado (validação Zod) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB com erro (sem truncamento) | TRANSCRICAO | [09:24] Diego; [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP de 10s por tentativa; estouro = falha/retry | TRANSCRICAO | [09:42] Diego/Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | At-least-once; dedup por X-Event-Id no cliente | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id (single-worker); ordering global não garantido | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado; restart da API não interrompe entregas | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Atomicidade status/evento (outbox transacional) | TRANSCRICAO | [09:06] Diego; [09:41] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, rotação com grace period | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Reuso de stack: Prisma/Pino/Zod/AppError; prefixo WEBHOOK_; sem dependências novas | TRANSCRICAO | [09:29] Bruno; [09:30] Larissa |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Auditoria: replay registra o operador | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Secret devolvida apenas na criação; não exposta em listagens | TRANSCRICAO | [09:31] Marcos; [09:21] Sofia |
| PRD-RISCO-01 | docs/PRD.md | Risco | Vazamento de secret (precedente real citado) | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não deduplicar eventos at-least-once | TRANSCRICAO | [09:24] Diego; [09:26] Marcos |
| PRD-RISCO-03 | docs/PRD.md | Risco | Prazo vs. expectativa da Atlas (fim de novembro) | TRANSCRICAO | [09:45] Marcos; [09:47] Marcos |
| PRD-RISCO-04 | docs/PRD.md | Risco | Crescimento da outbox degradando o worker | TRANSCRICAO | [09:07] Bruno; [09:08] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Indisponibilidade do cliente maior que a janela de retry | TRANSCRICAO | [09:16] Diego; [09:17] Marcos |
| PRD-RISCO-06 | docs/PRD.md | Risco | Replay indevido na fila de entrega | TRANSCRICAO | [09:36] Sofia |
| PRD-TESTE-01 | docs/PRD.md | Estratégia de teste | 2 dias úteis de revisão de segurança (HMAC, geração de secret) | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-02 | docs/PRD.md | Estratégia de teste | Contrato documentado em destaque no portal de desenvolvedores | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores = participantes; sessão de revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox no MySQL na mesma transação do status | TRANSCRICAO | [09:06] Diego; [09:07] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker em processo separado, polling 2s, single-worker com ordering por order_id | TRANSCRICAO | [09:09] Diego; [09:12] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry 1m/5m/30m/2h/12h + DLQ separada + replay admin | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24h, TLS, 64KB | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:24] Diego; [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão | Integração via publishWebhookEvent(tx, order, from, to) no changeStatus | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono dentro do service de orders | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams (infra nova, overengineering) | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para notificar worker (MySQL sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once com coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ABERTO-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-ABERTO-02 | docs/RFC.md | Questão em aberto | Escalonamento do worker (partição por order_id, lock pessimista) | TRANSCRICAO | [09:13] Diego |
| RFC-ABERTO-03 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-ABERTO-04 | docs/RFC.md | Questão em aberto | Política de arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-IMPACTO-01 | docs/RFC.md | Impacto | Prazo: 3 sprints com revisão de segurança incluída; Atlas quer fim de novembro | TRANSCRICAO | [09:46] Larissa; [09:45] Marcos |
| RFC-IMPACTO-02 | docs/RFC.md | Impacto | Clientes precisam implementar verificação HMAC e dedup | TRANSCRICAO | [09:25] Sofia; [09:26] Marcos |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Consistência transacional status ⇔ evento | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Latência de entrega ≤ 10s via polling 2s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Janela de retry ~15h (5 retentativas) | TRANSCRICAO | [09:17] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | HMAC-SHA256, secret por endpoint, TLS, rotação 24h | TRANSCRICAO | [09:20] Sofia; [09:22] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Rastreabilidade por X-Event-Id (UUID) | TRANSCRICAO | [09:25] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Zero desvio de stack (AppError, Pino, Zod, módulos, requireRole) | TRANSCRICAO | [09:30] Larissa |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox na mesma transação | TRANSCRICAO | [09:06] Diego |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Índices em status e created_at; batch pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Tabela webhook_dead_letter separada (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Tabela de configuração: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | Histórico de entregas (últimas 100, com tempo de resposta) | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | UUID como id (padrão do projeto) | TRANSCRICAO | [09:51] Diego/Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | changeStatus + publishWebhookEvent(tx, ...) dentro da $transaction | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro de status na inserção (economiza linha) | TRANSCRICAO | [09:34] Bruno/Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Polling de 2s; batch pequeno de pendentes mais antigos | TRANSCRICAO | [09:09] Diego; [09:08] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Estados do outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Agenda de backoff 1m/5m/30m/2h/12h; total ~15h | TRANSCRICAO | [09:17] Diego; [09:15] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Falha permanente ⇒ move para DLQ | TRANSCRICAO | [09:15] Diego; [09:18] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Replay: ADMIN + log de quem fez (reaproveita requireRole) | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | Rotação: antiga válida 24h em paralelo, depois morre | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-09 | docs/FDD.md | Fluxo | Snapshot do payload renderizado na inserção | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| FDD-FLUXO-10 | docs/FDD.md | Fluxo | Timeout de 10s no HTTP call do worker | TRANSCRICAO | [09:42] Diego/Sofia |
| FDD-FLUXO-11 | docs/FDD.md | Fluxo | Worker: src/worker.ts + npm run worker + PrismaClient próprio | TRANSCRICAO | [09:11] Larissa; [09:30] Bruno |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo | Evento >64KB não é enviado; erro (DLQ) — sem truncar | TRANSCRICAO | [09:23] Sofia; [09:24] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — criação com secret devolvida | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotação (derivado: path) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay ADMIN | TRANSCRICAO | [09:18] Diego; [09:35] Larissa |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Envio ao cliente: payload snapshot + headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id | TRANSCRICAO | [09:43] Diego; [09:44] Diego; [09:44] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Matriz de erros | Prefixo WEBHOOK_ para todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa; [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Matriz de erros | Códigos citados: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Matriz de erros | Demais códigos WEBHOOK_* (derivados dos fluxos e do padrão) | TRANSCRICAO | [09:28] Bruno (derivado) |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino existente, sem logger novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas de entrega: sucesso/falha, tempo de resposta (deliveries) | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Observar volume de envios para decidir rate limiting depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | X-Event-Id como chave de correlação/rastreio do evento | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração código | changeStatus (transação orders+history+estoque) estendido com publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração código | Novas classes de erro herdam AppError, padrão de http-errors.ts | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03 | docs/FDD.md | Integração código | Middleware de erro centralizado reutilizado sem alteração (AppError, Zod, Prisma) | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração código | authenticate/requireRole('ADMIN') para CRUD e replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração código | Registro do módulo em buildControllers/buildApiRouter (/api/v1) | CODIGO | src/app.ts; src/routes/index.ts |
| FDD-INT-06 | docs/FDD.md | Integração código | src/worker.ts espelha o padrão de entry/shutdown de src/server.ts | CODIGO | src/server.ts |
| FDD-INT-07 | docs/FDD.md | Integração código | PrismaClient por processo via createPrismaClient (mesma DATABASE_URL) | CODIGO | src/config/database.ts |
| FDD-INT-08 | docs/FDD.md | Integração código | Logger Pino compartilhado; redact paths extensíveis para secret | CODIGO | src/shared/logger/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração código | Convenção UUID @db.Char(36) + @@map para os novos models | CODIGO | prisma/schema.prisma |
| FDD-INT-10 | docs/FDD.md | Integração código | Enum OrderStatus e transições válidas (fonte do statusFilter e from/to_status) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-11 | docs/FDD.md | Integração código | Padrão de schemas Zod (nativeEnum OrderStatus) replicado no webhook.schemas.ts | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-12 | docs/FDD.md | Integração código | X-Request-Id e log http_request já instrumentam a API (tracing/correlação) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-13 | docs/FDD.md | Integração código | Padrão de testes (vitest/supertest, factories, bootstrapAuthenticatedUser(role)) | CODIGO | tests/helpers/factories.ts |
| FDD-INT-14 | docs/FDD.md | Integração código | Padrão de módulos (controller/service/repository/routes/schemas) a seguir | CODIGO | src/modules/orders/order.routes.ts |
| FDD-INT-15 | docs/FDD.md | Integração código | validate({ body: schema }) como middleware de validação padrão | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, transação atômica; índices status/created_at; arquivamento fora de escopo | TRANSCRICAO | [09:06] Diego; [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Worker separado, polling 2s, single-worker, ordering por order_id | TRANSCRICAO | [09:09] Diego; [09:12] Diego; [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 retentativas, backoff 1m/5m/30m/2h/12h, DLQ separada, replay admin | TRANSCRICAO | [09:15] Diego; [09:17] Larissa; [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24h, TLS, 64KB | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once + X-Event-Id (UUID), dedup no cliente | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks, AppError, Pino, Zod, requireRole, PrismaClient por processo | TRANSCRICAO | [09:27] Bruno; [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-de-payload-na-insercao.md | Decisão | Payload snapshot renderizado na inserção; UUID; sem items | TRANSCRICAO | [09:52] Larissa; [09:43] Diego; [09:51] Larissa |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão transacional prisma.$transaction já usado no changeStatus (referência explícita a código) | CODIGO | src/modules/orders/order.service.ts |
| ADR-006-COD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Referências explícitas: AppError/http-errors, error middleware, logger, auth, app/routes, server, schema.prisma | CODIGO | src/shared/errors/app-error.ts; src/middlewares/error.middleware.ts; src/shared/logger/index.ts |
| ADR-007-COD | docs/adrs/ADR-007-snapshot-de-payload-na-insercao.md | Decisão | Campos do snapshot (orderNumber, totalCents, customerId, status) vêm do model Order | CODIGO | prisma/schema.prisma |
| TRK-COD-01 | docs/FDD.md | Restrição técnica | Transação de status atualiza orders, order_status_history e estoque (é "pesada") | CODIGO | src/modules/orders/order.service.ts |
| TRK-COD-02 | docs/FDD.md | Restrição técnica | Papéis existentes: ADMIN e OPERATOR (enum UserRole) | CODIGO | prisma/schema.prisma |
| TRK-COD-03 | docs/RFC.md | Restrição técnica | API versionada sob /api/v1 com módulos isolados | CODIGO | src/routes/index.ts |
| TRK-COD-04 | docs/FDD.md | Restrição técnica | Node >= 20 (fetch global/AbortSignal) sem dependência HTTP nova | CODIGO | package.json |
| TRK-COD-05 | docs/FDD.md | Restrição técnica | Máquina de estados: PENDING→PAID→PROCESSING→SHIPPED→DELIVERED; CANCELLED | CODIGO | src/modules/orders/order.status.ts |
| TRK-COD-06 | docs/PRD.md | Contexto | GET /orders/:id existente (fonte de detalhes pós-evento) | CODIGO | src/modules/orders/order.routes.ts |

## Estatísticas de Cobertura

| Indicador | Valor | Requisito |
| --- | --- | --- |
| Total de linhas | 137 | — |
| Linhas com Fonte = TRANSCRICAO (com timestamp válido `[hh:mm] Nome`) | 113 (82%) | ≥ 70% ✓ |
| Linhas com Fonte = CODIGO (17 caminhos únicos, todos existentes) | 24 (18%) | ≥ 5 ✓ |
| Itens dos documentos com linha no tracker | ~90% dos itens identificáveis (requisitos, decisões, contratos, fluxos, riscos, exclusões, ADRs) | ≥ 80% ✓ |
| Caminhos de arquivo citados | 17 arquivos distintos do repositório + 2 propostos como novos (explicitados como proposta no FDD §11) | — |

*Contagem verificada por script de validação (ver README, "Iteração 4").*

Itens deliberadamente **fora** do tracker (não rastreáveis por design): decisões de formatação puramente editoriais (ex.: formato hex da assinatura, tamanho default do batch `WORKER_BATCH_SIZE`, nomes exatos de arquivos novos propostos) — são escolhas de implementação abertas ao desenvolvedor, marcadas como "decisão de implementação" no FDD, sem fonte na transcrição nem no código, e sem impacto em requisitos.
