# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks de notificação de mudança de status de pedidos (outbound) |
| **Versão** | 1.0 (2026-11-24) |
| **Status** | Aguardando revisão (sessão com Bruno e Diego — [09:50] Larissa) |
| **Documentos irmãos** | [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

---

## 1. Contexto e Motivação Técnica

O OMS atual (Node.js 20 + TypeScript + Express + Prisma + MySQL) opera o ciclo de vida do pedido com máquina de estados (`src/modules/orders/order.status.ts`), controle transacional de estoque e auditoria de status (`OrderStatusHistory`). Não existe hoje nenhum mecanismo de notificação externa, eventos, filas ou webhooks — esse vácuo é o que a feature preenche.

Clientes B2B hoje detectam mudanças consultando `GET /orders` periodicamente ([09:00] Marcos). O novo fluxo inverte o modelo: o OMS empurra o evento para o endpoint do cliente, assíncronamente, com garantias transacionais.

A restrição técnica central, levantada por Bruno: a transação de `changeStatus` já grava `orders`, `order_status_history` e movimenta estoque — nenhum I/O externo pode entrar nela ([09:04] Bruno). Daí o desenho outbox + worker (ADRs [001](./adrs/ADR-001-outbox-no-mysql.md) e [002](./adrs/ADR-002-worker-separado-com-polling.md)).

## 2. Objetivos Técnicos

1. **Consistência transacional**: status mudado ⇔ evento registrado na outbox, atomicamente — "não pode ter caso de status mudar e evento não sair" ([09:40]–[09:41] Bruno/Diego).
2. **Latência de entrega ≤ 10s** (percepção de "tempo real" dos clientes — [09:02] Marcos), atingida com polling de 2s ([09:09]–[09:10] Diego/Marcos/Larissa).
3. **Resiliência a cliente offline**: 1 tentativa inicial + 5 retentativas com backoff 1m/5m/30m/2h/12h, janela de ~15h entre a primeira falha e a última tentativa ([09:15]–[09:17] Diego), depois DLQ.
4. **Integridade e autenticidade**: HMAC-SHA256 sobre o corpo, secret por endpoint, TLS obrigatório, rotação com grace period de 24h ([09:19]–[09:24] Sofia).
5. **Rastreabilidade**: cada evento identificável por UUID (`X-Event-Id`) do nascimento à entrega/DLQ, com histórico de entregas consultável ([09:25] Diego, [09:34] Marcos).
6. **Zero desvio de stack**: módulo no padrão `src/modules`, `AppError`, Pino, Zod, middleware de erro e `requireRole` reaproveitados ([09:27]–[09:30] Bruno/Diego/Larissa, [09:36] Larissa).

## 3. Escopo e Exclusões

**Incluído:**

- Módulo `src/modules/webhooks` (configuração de webhooks: CRUD + deliveries + rotação de secret).
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries` (Prisma).
- Worker em processo separado (`src/worker.ts`, script `npm run worker`) com polling, retry, assinatura HMAC e DLQ.
- Endpoint admin de replay de DLQ (role ADMIN, com auditoria).
- Integração transacional em `OrderService.changeStatus` via `publishWebhookEvent(tx, ...)`.

**Excluído (decisões da reunião):**

- **Email de alerta ao cliente por falhas recorrentes** — próxima fase, após medir impacto ([09:37]–[09:38] Marcos/Larissa).
- **Dashboard visual** para o cliente — projeto separado do time de frontend; cliente integra via API/endpoints ([09:39]–[09:40] Marcos/Larissa).
- **Rate limiting de saída por cliente** — ponto em aberto; observar e decidir depois ([09:38]–[09:39] Diego/Larissa).
- **Arquivamento/purga da outbox (~30 dias)** — fora do escopo desta feature ([09:08] Diego).
- **Escalar para múltiplos workers** (particionamento por `order_id`, lock pessimista) — problema do futuro ([09:13] Diego).
- **Webhooks inbound** — só saída; clientes não enviam eventos para nós ([09:02] Sofia/Marcos).

## 4. Modelo de Dados

Novos models no `prisma/schema.prisma`, seguindo as convenções existentes (id UUID `@db.Char(36)`, `@@map` em snake_case, índices explícitos — [09:51] Diego/Larissa):

```prisma
model WebhookEndpoint {
  id                        String   @id @default(uuid()) @db.Char(36)
  customerId                String   @db.Char(36)
  url                       String   @db.VarChar(2048) // obrigatoriamente https
  secret                    String   @db.VarChar(128)  // secret ativa (HMAC-SHA256)
  previousSecret            String?  @db.VarChar(128)  // durante grace period de 24h
  previousSecretExpiresAt   DateTime?
  statusFilter              Json     // lista de OrderStatus que este webhook escuta
  active                    Boolean  @default(true)
  createdAt                 DateTime @default(now())
  updatedAt                 DateTime @updatedAt

  customer Customer           @relation(fields: [customerId], references: [id])
  outbox    WebhookOutbox[]   @relation("OutboxEndpoint")
  deliveries WebhookDelivery[] @relation("DeliveryEndpoint")

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id             String   @id @default(uuid()) @db.Char(36) // = event_id (X-Event-Id)
  webhookId      String?  @db.Char(36) // null se endpoint removido antes da entrega
  orderId        String   @db.Char(36)
  orderNumber    String   @db.VarChar(20)
  customerId     String   @db.Char(36)
  fromStatus     String   @db.VarChar(20)
  toStatus       String   @db.VarChar(20)
  payload        Json     // snapshot renderizado na inserção (ADR-007)
  status         String   @db.VarChar(20) @default("PENDING") // PENDING | PROCESSING | DELIVERED | FAILED
  attempts       Int      @default(0)
  nextAttemptAt  DateTime?
  lastError      String?  @db.VarChar(500)
  createdAt      DateTime @default(now())
  deliveredAt    DateTime?

  webhook WebhookEndpoint? @relation("OutboxEndpoint", fields: [webhookId], references: [id])

  @@index([status, nextAttemptAt]) // leitura do worker: pendentes elegíveis
  @@index([createdAt])
  @@index([customerId])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  eventId      String   @db.Char(36) // espelha webhook_outbox.id
  webhookId    String?  @db.Char(36)
  orderId      String   @db.Char(36)
  payload      Json
  failureReason String   @db.VarChar(500) // ex.: timeout, HTTP 500, connection refused
  attempts     Int
  failedAt     DateTime @default(now()) // timestamp da falha permanente
  replayedAt   DateTime?
  replayedById String?  @db.Char(36)    // auditoria: quem executou o replay

  @@index([failedAt])
  @@index([replayedAt])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id          String   @id @default(uuid()) @db.Char(36)
  webhookId   String   @db.Char(36)
  outboxId    String   @db.Char(36)
  attempt     Int
  success     Boolean
  statusCode  Int?     // resposta HTTP do cliente (null p/ erro de rede/timeout)
  response    String?  @db.VarChar(500) // trecho da resposta
  payload     Json
  durationMs  Int      // tempo de resposta ([09:34] Marcos)
  createdAt   DateTime @default(now())

  webhook WebhookEndpoint @relation("DeliveryEndpoint", fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}
```

Notas:

- O campo `status` do outbox cobre os quatro estados citados por Diego — pendente, processando, falhou, entregue ([09:08]) — com índice em status e `created_at` ([09:08] Diego).
- `WebhookOutbox.id` **é** o `event_id` (UUID único por evento — [09:25] Diego, [09:51] Larissa), enviado como `X-Event-Id`.
- A proteção da(s) secret(s) em repouso (cifrada ou não) é ponto sensível a validar na revisão de segurança da Sofia ([09:46] — "HMAC e geração de secret eu quero olhar com calma").
- `statusFilter` como lista de `OrderStatus` reflete "lista dos status que o webhook quer ouvir" ([09:33] Marcos).

## 5. Fluxos Detalhados

### 5.1 Criação do evento na outbox (caminho da API)

**Gatilho**: `PATCH /api/v1/orders/:id/status` → `OrderService.changeStatus`.

1. A transação existente (`prisma.$transaction`) executa como hoje: valida transição (`canTransition`), debita/repostoca estoque, atualiza `orders` e insere em `order_status_history` (`src/modules/orders/order.service.ts`).
2. **Nova etapa, ainda dentro da mesma transação**: `changeStatus` chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` — função pura que recebe o `tx` client ([09:41] Bruno/Diego), sem injetar repository inteiro.
3. `publishWebhookEvent`:
   a. Consulta os `WebhookEndpoint` ativos do `order.customerId` cujo `statusFilter` inclua `toStatus` ([09:33]–[09:34] Marcos/Bruno — filtro **na inserção**: "Se nenhum webhook do customer quer aquele status, nem insere").
   b. Para cada endpoint elegível, monta o snapshot do payload (ver §5.6) com UUID recém-gerado e insere a linha `PENDING` em `webhook_outbox`.
   c. Se o snapshot montado exceder **64KB**, o evento não é enviado ([09:23]–[09:24] Sofia/Diego): registra log `error` e insere direto em `webhook_dead_letter` com motivo `WEBHOOK_PAYLOAD_TOO_LARGE` — a transação de status **não** é impactada.
4. Commit da transação: status do pedido e eventos são atomicamente consistentes; rollback: tudo some junto ([09:06] Diego).

**Efeito colateral garantido**: se a inserção na outbox falhar, a mudança de status recebe rollback — "não pode ter caso de status mudar e evento não sair" ([09:41] Bruno/Diego).

### 5.2 Processamento pelo worker

**Processo**: `src/worker.ts` (entry separada, script `npm run worker` — [09:11] Larissa), com lógica em `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno). Instancia seu **próprio PrismaClient** (`createPrismaClient()` de `src/config/database.ts`), mesmo banco e `DATABASE_URL` ([09:30] Bruno).

Loop (single-worker — [09:12] Diego):

1. A cada **2 segundos** ([09:09] Diego): seleciona um **batch pequeno** (implementação: parâmetro `WORKER_BATCH_SIZE`, default 50 — [09:08] Diego "batch pequeno") de linhas `webhook_outbox` com `status = PENDING` e (`nextAttemptAt IS NULL OR nextAttemptAt <= now`), ordenadas por `createdAt ASC` — ordering por `order_id` preservado por ser single-worker ([09:12] Diego).
2. Marca as linhas como `PROCESSING` antes de enviar (evita re-pick no próximo tick se o tick seguinte rodar antes do fim do envio).
3. Para cada linha: monta o request HTTP (payload snapshot + headers §5.6), assina com a secret **ativa** do endpoint e envia com `fetch` + `AbortSignal.timeout(10_000)` — **timeout de 10s**; cliente que não responde em 10s é tratado como falha e marcado para retry ([09:42] Sofia/Diego).
4. Registra a tentativa em `webhook_deliveries` (attempt, success, statusCode, response, `durationMs`, payload).
5. Resultado:
   - **Resposta 2xx** → `status = DELIVERED`, `deliveredAt = now`. *(Critério 2xx: decisão de implementação derivada de "sucesso/falha" [09:34] Marcos; timeout e erro de rede são falha [09:42].)*
   - **Falha** (não-2xx, timeout, erro de rede) → fluxo de retry (§5.3).
6. Repete. Shutdown gracioso em `SIGINT`/`SIGTERM` (padrão de `src/server.ts`: fechar loop, `prisma.$disconnect()`).

### 5.3 Retry com backoff

Após uma tentativa falha ([09:15]–[09:17] Diego, [09:48] Larissa):

| Evento | Agenda |
| --- | --- |
| Tentativa 1 (envio inicial) | imediata (polling) |
| Falha 1 → tentativa 2 | +**1 minuto** |
| Falha 2 → tentativa 3 | +**5 minutos** |
| Falha 3 → tentativa 4 | +**30 minutos** |
| Falha 4 → tentativa 5 | +**2 horas** |
| Falha 5 → tentativa 6 (última) | +**12 horas** |
| Falha 6 → **DLQ** | falha permanente |

- Janela total entre a primeira falha e a última tentativa: 1m + 5m + 30m + 2h + 12h ≈ **14,6h ("quase 15 horas" — [09:17] Diego)**, cobrindo indisponibilidades reais de clientes (caso citado: 2h de manutenção planejada — [09:16] Diego) e a janela "12 ou 24 horas" citada em [09:15].
- Implementação: `attempts++`, `nextAttemptAt = now + DELAYS[attempts]`, `status = PENDING` (volta ao pool elegível), `lastError` atualizado. Sem fila externa: o próprio `nextAttemptAt` + índice `(status, nextAttemptAt)` agenda as retentativas.
- As 5 **retentativas** correspondem ao "teto de tentativas" de 5 discutido ([09:15]–[09:16] Diego/Bruno/Larissa); a agenda completa acima preserva o total de ~15h declarado por Diego ([09:17]) e confirmado por Larissa ([09:48]).

### 5.4 DLQ e replay

**Esgotadas as tentativas** ([09:18] Diego, [09:48] Larissa):

1. A linha do outbox recebe `status = FAILED` (registro local).
2. Uma linha é inserida em `webhook_dead_letter` com **payload, motivo da falha e timestamp** ([09:18] Diego) — tabela separada para manter a leitura da outbox limpa e servir de evidência para debug e reprocessamento ([09:18] Diego).
3. **Replay manual**: `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego, [09:35] Larissa):
   - Exige **role ADMIN** — "mexer em fila de entrega de notificação não é coisa de operador" ([09:36] Sofia), via `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa — "a gente reaproveita o requireRole que já existe").
   - **Loga quem fez o replay** para auditoria ([09:36] Sofia) — campo `replayedById` + log Pino com `userId`.
   - Efeito: a linha original da outbox volta a `PENDING`, `attempts = 0`, `nextAttemptAt = now` ("recoloca na outbox como pendente" — [09:18] Diego). O `event_id` é preservado, de modo que um cliente que eventualmente já tenha recebido aquele evento consiga deduplicar (at-least-once — ADR-005).

### 5.5 Rotação de secret (grace period de 24h)

([09:21] Sofia, [09:22] Sofia/Diego)

1. Cliente chama `POST /api/v1/webhooks/:id/rotate-secret` (autenticado).
2. Sistema gera nova secret, move a atual para `previousSecret` com `previousSecretExpiresAt = now + 24h`.
3. **Envios seguintes assinam com a nova secret.** Durante 24h, a secret antiga permanece **válida para verificação** pelo cliente (ele valida com qualquer uma das duas), dando tempo de migrar os sistemas dele ([09:21] Sofia).
4. Após expirar, a antiga morre: somente a nova assina/valida ([09:21] Sofia).
5. Racional do incidente que motiva o desenho: cliente já vazou secret em log de aplicação ([09:22] Diego); rotação rápida limita o dano, e o grace period evita quebrar a integração ([09:21] Sofia).

### 5.6 Contrato do envio HTTP (para o endpoint do cliente)

**Payload** (snapshot renderizado na inserção — ADR-007, [09:43] Diego): JSON enxuto, sem items:

```json
{
  "event_id": "7f9c24e8-3b1a-4c8f-9d2e-5a6b7c8d9e0f",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-24T12:34:56.789Z",
  "order_id": "b1e2c3d4-...",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "a9b8c7d6-...",
  "total_cents": 12345
}
```

- `timestamp` em **ISO 8601** ([09:43] Diego).
- Sem items: "Se o cliente quiser detalhes, ele bate no GET /orders/:id depois" ([09:43] Diego) — payload enxuto endossado por Bruno ([09:44]).

**Headers** ([09:44] Diego, [09:44]–[09:45] Sofia):

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento (idempotência/dedup) | [09:25] Diego, [09:44] |
| `X-Signature` | HMAC-SHA256 do corpo do request, secret do endpoint | [09:20] Sofia |
| `X-Timestamp` | timestamp do **envio** — permite ao cliente detectar replay attack | [09:44] Diego |
| `X-Webhook-Id` | id do cadastro do webhook — cliente com vários endpoints identifica qual recebeu | [09:44] Sofia, [09:45] Diego |

Assinatura: `HMAC_SHA256(secret, corpo_serializado)` (formato de saída hex — decisão de implementação, não discutida na reunião; documentar no portal).

## 6. Contratos Públicos (API do OMS)

Autenticação: `Authorization: Bearer <JWT>` (middleware `authenticate`) em todas as rotas abaixo. CRUD aceita **qualquer role autenticada** por enquanto ([09:36]–[09:37] Sofia); replay exige ADMIN ([09:36] Sofia/Larissa). `customerId` vem do **body/query**, nunca do JWT — o JWT é do usuário operador, não do cliente ([09:32] Bruno/Larissa/Marcos).

Formato de erro segue o padrão vigente do `error.middleware`: `{ "error": { "code": "...", "message": "...", "details": ... } }`.

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

Campos: `url`, lista de status, `customerId`; secret **gerada pelo sistema e devolvida na criação** ([09:31] Marcos).

Request:

```json
{
  "customerId": "a9b8c7d6-1111-2222-3333-444455556666",
  "url": "https://api.atlascomercial.example/hooks/orders",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Response `201`:

```json
{
  "id": "c1d2e3f4-...",
  "customerId": "a9b8c7d6-1111-2222-3333-444455556666",
  "url": "https://api.atlascomercial.example/hooks/orders",
  "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whk_3f9d2a7e...",
  "createdAt": "2026-11-24T12:00:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL` (URL `http://` — [09:23] Sofia), `400 WEBHOOK_INVALID_STATUS_FILTER` (status fora do enum), `404 NOT_FOUND` (customer inexistente).

> A secret é devolvida **uma única vez**, na criação ([09:31] Marcos). Nas demais respostas ela não aparece.

### 6.2 `GET /api/v1/webhooks?customerId={id}` — listar webhooks de um customer

([09:33] Bruno — "GET pra listar os webhooks de um customer")

Response `200`:

```json
{
  "data": [
    {
      "id": "c1d2e3f4-...",
      "customerId": "a9b8c7d6-...",
      "url": "https://api.atlascomercial.example/hooks/orders",
      "statusFilter": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-24T12:00:00.000Z"
    }
  ]
}
```

### 6.3 `PATCH /api/v1/webhooks/{id}` — editar webhook

([09:33] Bruno — "PATCH pra editar")

Request (campos parciais):

```json
{
  "url": "https://api2.atlascomercial.example/hooks/orders",
  "statusFilter": ["SHIPPED", "DELIVERED"],
  "active": false
}
```

Response `200`: objeto do webhook atualizado (mesma forma do `GET`). Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`.

### 6.4 `DELETE /api/v1/webhooks/{id}` — remover webhook

([09:33] Bruno — "DELETE pra remover")

Response: `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.5 `GET /api/v1/webhooks/{id}/deliveries` — histórico de entregas

Últimas **100** entregas com sucesso/falha, payload, response e **tempo de resposta** ([09:34] Marcos).

Response `200`:

```json
{
  "data": [
    {
      "id": "d5e6f7a8-...",
      "outboxId": "7f9c24e8-3b1a-4c8f-9d2e-5a6b7c8d9e0f",
      "attempt": 2,
      "success": false,
      "statusCode": 503,
      "response": "Service Unavailable",
      "durationMs": 143,
      "createdAt": "2026-11-24T12:05:03.100Z"
    },
    {
      "id": "e6f7a8b9-...",
      "outboxId": "7f9c24e8-3b1a-4c8f-9d2e-5a6b7c8d9e0f",
      "attempt": 1,
      "success": true,
      "statusCode": 200,
      "response": "{\"received\":true}",
      "durationMs": 87,
      "createdAt": "2026-11-24T12:04:01.050Z"
    }
  ]
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `404 WEBHOOK_DELIVERY_NOT_FOUND` (id inexistente).

### 6.6 `POST /api/v1/webhooks/{id}/rotate-secret` — rotacionar secret

"Endpoint pro cliente conseguir pedir nova secret pela API" ([09:21] Sofia); antiga válida por 24h em paralelo.

Request: corpo vazio.

Response `200`:

```json
{
  "id": "c1d2e3f4-...",
  "secret": "whk_9b8a7c6d...",
  "previousSecretValidUntil": "2026-11-25T12:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`, `409 WEBHOOK_INACTIVE` (webhook inativo).

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/{id}/replay` — reprocessar DLQ

([09:18] Diego, [09:35] Larissa, [09:36] Sofia/Larissa). **Role ADMIN obrigatória** (`requireRole('ADMIN')`); execução registrada para auditoria ([09:36] Sofia).

Request: corpo vazio. Response `200`:

```json
{
  "id": "f0e1d2c3-...",
  "eventId": "7f9c24e8-3b1a-4c8f-9d2e-5a6b7c8d9e0f",
  "status": "PENDING",
  "attempts": 0
}
```

Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN` (role não-ADMIN — comportamento do `requireRole` existente).

## 7. Matriz de Erros Previstos

Todos os códigos do módulo usam o prefixo **`WEBHOOK_`** ([09:28]–[09:29] Bruno/Larissa), herdam de `AppError` e fluem pelo middleware de erro centralizado sem alteração ([09:29] Bruno).

| Código | Superfície | HTTP | Cenário | Ação do sistema |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | API | 404 | Operação sobre webhook inexistente (`PATCH`/`DELETE`/`deliveries`/`rotate-secret`) | Rejeitar; citado explicitamente em [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | API | 400 | URL cadastrada sem `https://` | Rejeitar na validação Zod ([09:23] Sofia, [09:28] Bruno) |
| `WEBHOOK_SECRET_REQUIRED` | API | 400 | Operação que exige secret ativa sem secret disponível (ex.: rotação/entrega em endpoint sem secret válida) | Rejeitar; citado explicitamente em [09:28] Bruno |
| `WEBHOOK_INVALID_STATUS_FILTER` | API | 400 | `statusFilter` com valor fora do enum `OrderStatus` | Rejeitar na validação Zod (derivado de [09:33] Marcos + padrão `nativeEnum` de `order.schemas.ts`) |
| `WEBHOOK_INACTIVE` | API | 409 | Operação sobre webhook desativado (ex.: rotação de secret) | Rejeitar (derivado do "estado ativo" — [09:21] Bruno) |
| `WEBHOOK_DELIVERY_NOT_FOUND` | API | 404 | Consulta de entrega inexistente | Rejeitar (derivado de [09:34] Marcos) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | API | 404 | Replay de item inexistente na DLQ | Rejeitar (derivado de [09:18] Diego) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Evento | — | Snapshot do evento excede **64KB** | Não enviar; registrar log `error` e mover direto para DLQ com esse motivo ([09:23]–[09:24] Sofia/Diego) |
| `WEBHOOK_DELIVERY_TIMEOUT` | Evento | — | Cliente não respondeu em **10s** | Contar como falha, agendar retry ([09:42] Sofia/Diego) |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Evento | — | Cliente respondeu com HTTP não-2xx | Contar como falha, agendar retry (derivado de [09:34] "sucesso/falha") |

Erros de infra já cobertos pelo middleware centralizado (sem código novo): validação Zod genérica (`VALIDATION_ERROR`), autorização (`UNAUTHORIZED`/`FORBIDDEN`), constraint do Prisma (`CONFLICT`) ([09:29] Bruno).

## 8. Estratégias de Resiliência

- **Timeout de envio: 10 segundos** — cliente lento/não responsivo vira falha e agenda retry ([09:42] Sofia/Diego). Implementado com `AbortSignal.timeout(10_000)` no `fetch` do worker.
- **Retries com backoff exponencial**: agenda 1m/5m/30m/2h/12h (§5.3) — total ~15h de tolerância a indisponibilidade do cliente ([09:17] Diego/Marcos).
- **DLQ como fallback terminal**: evento permanente preservado (payload + motivo + timestamp) e recuperável por replay manual ([09:18] Diego). Sem retry infinito — evita evento pendurado para sempre ([09:15] Diego).
- **Isolamento de processo**: worker fora da instância da API — restart/deploy da API não interrompe entregas ([09:11] Diego); shutdown gracacio do worker segue o padrão SIGINT/SIGTERM de `src/server.ts`.
- **Isolamento transacional**: HTTP nunca dentro da transação de status ([09:04] Bruno/[09:06] Diego).
- **Fallback consciente (não implementado)**: email ao cliente por falhas recorrentes ficou para a próxima fase, "depois que a gente medir o impacto" ([09:37]–[09:38] Marcos/Larissa).

## 9. Observabilidade

**Logs (Pino, reuso integral — [09:29] Bruno — "não vamos botar nada novo"):**

- Worker: `webhook_event_created` (na API, dentro de `publishWebhookEvent`), `webhook_delivery_attempt` (eventId, webhookId, attempt), `webhook_delivery_result` (eventId, success, statusCode, durationMs, error), `webhook_dead_letter_move` (eventId, failureReason, attempts), `webhook_replay` (deadLetterId, eventId, **userId** — auditoria exigida por [09:36] Sofia), `webhook_secret_rotated` (webhookId, sem conteúdo de secret).
- Recomendação: incluir `secret`/`previousSecret` nos redact paths do logger (`src/shared/logger/index.ts` já redacta `*.token` e headers de autorização).
- API: requests de webhook já ganham `X-Request-Id` + log `http_request` pelo `request-logger.middleware` existente, sem mudança.

**Métricas** (não há stack de métricas no projeto; expor via logs estruturados Pino + consulta SQL nas tabelas — decisão de implementação alinhada ao "reuso máximo" [09:30] Larissa e ao "observar e decidir depois" [09:39] Diego):

- Lag da outbox: count de `PENDING`/`PROCESSING` por idade (saúde do worker — atende o "a gente observa" do rate limiting, [09:38]–[09:39] Diego/Larissa).
- Taxa de sucesso/falha por tentativa e por cliente (`webhook_deliveries`).
- Latência de entrega (`durationMs` — "tempo de resposta" pedido por [09:34] Marcos).
- Tamanho da DLQ e tempo até replay (`webhook_dead_letter`).

**Tracing (correlação):** `X-Event-Id` (UUID) propagado em todos os logs do ciclo de vida do evento — do `webhook_event_created` na API até entrega/DLQ/replay — é a chave de correlação entre processos (API e worker são processos separados, ADR-002); na camada HTTP, o `X-Request-Id` existente (`src/middlewares/request-logger.middleware.ts`) continua correlacionando request→resposta. Não introduzimos APM/OTel novo (consistente com [09:30] Larissa — reuso máximo). O cliente pode correlacionar o `X-Event-Id` recebido com seus logs.

## 10. Dependências e Compatibilidade

- **Runtime**: Node.js ≥ 20 (já exigido em `package.json`) — `fetch` global e `AbortSignal.timeout` nativos, sem dependência nova de HTTP client (consistente com [09:29] Bruno).
- **Banco**: MySQL existente via Prisma 5.22; novas tabelas via migração (`prisma migrate dev`). Nenhuma mudança nos models existentes.
- **Bibliotecas**: apenas as já presentes — `uuid` (event_id), `zod` (schemas), `pino` (logs), `@prisma/client`, `jsonwebtoken`/middlewares existentes. **Nenhuma dependência nova** ([09:29] Bruno, [09:30] Larissa; `node:crypto` para HMAC).
- **Compatibilidade com clientes**: carga útil nova e opt-in (cliente cadastra webhook para receber); quem não cadastra não é afetado. `GET /orders/:id` permanece a fonte de detalhes ([09:43] Diego).
- **Deploys**: um artefato novo para orquestrar (processo worker) — mesmo repositório, entry distinta; a API e o worker podem ser deployados independentemente ([09:11] Diego).
- **Coordenação de time**: revisão de segurança (Sofia) com 2 dias úteis reservados antes do deploy, focando HMAC e geração de secret ([09:46] Sofia).

## 11. Integração com o Sistema Existente

Ponto de partida: a aplicação tem padrões consolidados de módulos, erros, validação e composição (descritos por Bruno em [09:27]–[09:29]). A integração toca os seguintes arquivos reais do código base:

1. **`src/modules/orders/order.service.ts`** — alteração crítica ([09:40] Bruno): o método `changeStatus` (linhas ~126–179) ganha, dentro da `prisma.$transaction` existente, a chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` após a gravação do `order_status_history` e antes do refresh final. A função é propositalmente **pura em relação ao tx** — "não precisa injetar repository inteiro" ([09:41] Diego) — evitando acoplar `OrderService` a um repository de webhooks. Se a inserção na outbox falhar, a transação inteira sofre rollback ([09:41] Bruno/Diego).
2. **`src/shared/errors/http-errors.ts`** (e `src/shared/errors/app-error.ts`) — novas classes de erro (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendendo `AppError`/`ConflictError`/`BadRequestError` com códigos `WEBHOOK_*`, seguindo o exemplo de `InvalidStatusTransitionError` e `InsufficientStockError` ([09:28] Bruno — "Quero seguir igual pra webhook").
3. **`src/middlewares/error.middleware.ts`** — reaproveitado **sem alteração**: já trata `AppError` (statusCode + errorCode + details), `ZodError` e erros Prisma ([09:29] Bruno — "Vai pegar nossos erros sem precisar mudar nada").
4. **`src/middlewares/auth.middleware.ts`** — `authenticate` protege as rotas CRUD do módulo; `requireRole('ADMIN')` protege o replay de DLQ ([09:36] Larissa — "reaproveita o requireRole que já existe"; [09:36] Sofia — ADMIN obrigatório).
5. **`src/app.ts` e `src/routes/index.ts`** — `buildControllers` ganha o controller de webhooks (instância de repository/service/controller, como os demais módulos) e `buildApiRouter` registra `router.use('/webhooks', ...)` e a rota admin `router.use('/admin/webhooks/dead-letter/:id/replay', ...)` sob `/api/v1`.
6. **`src/server.ts`** — modelo para a nova entry `src/worker.ts`: mesmo padrão de bootstrap com logger, `SIGINT`/`SIGTERM` com shutdown gracioso e `prisma.$disconnect()` ([09:11] Larissa — "igual o que a gente já tem em src/server.ts").
7. **`src/config/database.ts`** — `createPrismaClient()` reutilizada pelo worker, mas em **instância própria** ("PrismaClient é por processo" — [09:30] Bruno; mesmo banco/`DATABASE_URL` — [09:11] Bruno).
8. **`src/shared/logger/index.ts`** — logger Pino compartilhado por API e worker; recomenda-se apenas estender os `redactPaths` para campos de secret ([09:29] Bruno — Pino permanece o logger único).
9. **`prisma/schema.prisma`** — novos models do §4 seguindo as convenções (UUID `@db.Char(36)`, `@@map` snake_case, índices — [09:51] Diego/Larissa); enum `OrderStatus` existente é a fonte do `statusFilter`.
10. **`src/modules/orders/order.status.ts`** — fonte do enum e das transições válidas (`PENDING→PAID→PROCESSING→SHIPPED→DELIVERED`, `CANCELLED`) que aparecem no payload (`from_status`/`to_status`) e no filtro de eventos ([09:33] Marcos).
11. **`src/middlewares/validate.middleware.ts`** e **`src/modules/orders/order.schemas.ts`** — padrão de validação a espelhar em `webhook.schemas.ts` (incluindo a recusa de URL `http` apontada por Sofia, [09:23]).
12. **`tests/helpers/factories.ts`** e **`tests/orders.test.ts`** — padrão de teste (vitest + supertest + factories com `bootstrapAuthenticatedUser(role)`) a estender para os testes do módulo e do `changeStatus`+outbox.

Novos arquivos (proposta, no padrão citado por Bruno/Diego): `src/modules/webhooks/{webhook.controller,webhook.service,webhook.repository,webhook.routes,webhook.schemas}.ts`, `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`), `src/worker.ts`, `publishWebhookEvent` em `src/modules/webhooks/` ([09:27]–[09:28] Bruno/Diego).

## 12. Critérios de Aceite Técnicos

1. **Atomicidade**: mudança de status commitada ⇒ linhas `PENDING` na outbox para todos os webhooks elegíveis; rollback da transação (ex.: erro simulado após o insert) ⇒ nenhuma linha sobrevive ([09:06], [09:40]–[09:41] Diego/Bruno).
2. **Filtro na inserção**: mudança para status fora do `statusFilter` de todos os webhooks do customer ⇒ nenhuma linha criada ([09:34] Bruno).
3. **Latência**: evento criado ⇒ tentativa de envio iniciada em ≤ 2s + tempo de envio (polling 2s — [09:09]/[09:10]).
4. **Timeout**: endpoint que demora > 10s ⇒ tentativa abortada e falha registrada com `durationMs` ≈ 10000 ([09:42]).
5. **Backoff**: falhas consecutivas ⇒ novas tentativas em 1m, 5m, 30m, 2h, 12h; após a 6ª tentativa, linha em `webhook_dead_letter` com payload, motivo e timestamp ([09:17], [09:18]).
6. **DLQ + replay**: replay por ADMIN volta a linha a `PENDING`/attempts 0 e registra `replayedById`; OPERATOR recebe `403` ([09:18], [09:36] Sofia).
7. **Assinatura**: request recebido pelo endpoint de teste contém `X-Event-Id` (UUID), `X-Signature` = HMAC-SHA256 recomputável com a secret, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` ([09:20], [09:44]–[09:45]).
8. **At-least-once**: reentrega manual (replay) preserva o `event_id`, permitindo dedup no receptor ([09:24]–[09:25] Diego).
9. **Segredos**: cadastro com `http://` ⇒ `400 WEBHOOK_INVALID_URL`; rotação mantém a anterior válida por 24h (verificável em janela de teste) e a devolve apenas na criação/rotação ([09:23], [09:21], [09:31]).
10. **Payload**: snapshot contém exatamente os campos do §5.6; sem items; ≤ 64KB (caso estoure: DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`) ([09:43], [09:24]).
11. **Deliveries**: `GET /webhooks/:id/deliveries` retorna as últimas 100 tentativas com sucesso/falha, payload, response e `durationMs` ([09:34] Marcos).
12. **Isolamento**: `npm run worker` é um processo separado da API (`src/server.ts`); derrubar/reiniciar a API não interrompe o loop do worker ([09:11] Diego).
13. **Compatibilidade de erros**: respostas de erro seguem o envelope `{ error: { code, message, details } }` do middleware atual, sem nenhuma alteração nele ([09:29] Bruno).

## 13. Riscos e Mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Tabela `webhook_outbox` crescer e degradar polling ([09:07] Bruno) | Média | Médio | Índices `(status, nextAttemptAt)` e `createdAt`; batch pequeno; arquivamento ~30 dias registrado como questão em aberto ([09:08] Diego) |
| Secret armazenada/manipida de forma insegura | Média | Alto | Revisão de segurança dedicada (2 dias úteis — [09:46] Sofia); secret por endpoint limita raio de vazamento ([09:21]); rotação 24h; redact de logs |
| Worker virar gargalo em picos (single-worker — [09:12] Diego) | Baixa | Médio | Métricas de lag da outbox (§9); caminho de escala já identificado: particionar por `order_id` ou lock pessimista ([09:13] Diego) |
| Cliente não deduplicar eventos (at-least-once) | Média | Médio | Contrato documentado em destaque no portal ([09:26] Marcos); `X-Event-Id` estável |
| Evento grande demais não detectado antes da entrega | Baixa | Baixo | Checagem de 64KB na geração do snapshot (§5.1.3c) com DLQ imediata ([09:23]–[09:24]) |
| Integração no `changeStatus` introduzir regressão no fluxo de pedidos | Baixa | Alto | `publishWebhookEvent` isolado e testado dentro da transação existente; testes de rollback (critério 1); suite atual de `tests/orders.test.ts` como rede de segurança |

---

**Rastreabilidade**: cada requisito, decisão e restrição deste documento tem linha correspondente no [TRACKER.md](./TRACKER.md), apontando para a fala exata na transcrição ou para o arquivo do código.
