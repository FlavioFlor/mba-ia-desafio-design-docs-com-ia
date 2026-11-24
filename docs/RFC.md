# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Flavio Flor (elaboração assistida por IA, a partir da reunião técnica de novembro de 2026) |
| **Status** | Draft — aguardando revisão da equipe |
| **Data** | 2026-11-24 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Feature** | Webhooks de notificação de mudança de status de pedidos (outbound) |
| **Discussão de origem** | Reunião técnica de ~55 min (`TRANSCRICAO.md`) |

> Revisores: conforme combinado em [09:50] Larissa, a proposta passará por sessão de revisão com Bruno e Diego antes do início da codação; Sofia reserva dois dias úteis para revisão de segurança antes do deploy ([09:46] Sofia).

## TL;DR

Propomos um sistema de **webhooks outbound** que notifica clientes B2B quando o status de seus pedidos muda, substituindo o polling manual deles no `GET /orders`. A entrega é assíncrona via **padrão Outbox no MySQL existente**: o evento é gravado na mesma transação SQL da mudança de status (sem risco de inconsistência) e um **worker em processo separado** faz polling de 2 segundos e dispara HTTP assinado com **HMAC-SHA256** (secret por endpoint, rotação com grace period de 24h). Falhas retentam com **backoff exponencial 1m/5m/30m/2h/12h (5 tentativas)** e, esgotadas, vão para uma **DLQ** com replay manual via endpoint admin. A entrega é **at-least-once**, com `X-Event-Id` (UUID) para deduplicação no cliente — o mesmo modelo de Stripe e GitHub ([09:25] Diego). Reuso integral dos padrões da codebase (módulos, `AppError`, Pino, Zod). Prazo estimado: **3 sprints** ([09:46] Larissa), contemplando a revisão de segurança.

## Contexto e Problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente notificação em tempo real de mudança de status de pedidos ([09:00] Marcos). Hoje eles consultam o `GET /orders` periodicamente para detectar mudanças, o que torna a integração deles lenta e cara. A Atlas sinalizou risco de migração para o concorrente caso não haja entrega até fim do trimestre ([09:00] Marcos). Para esses clientes, "tempo real" significa qualquer latência **abaixo de 10 segundos** ([09:02] Marcos).

Escopo direcional fechado no início: **outbound apenas** — os webhooks saem de nós para eles; eles não enviam para nós ([09:02] Sofia/Marcos).

Restrição técnica central: a transação de mudança de status (`OrderService.changeStatus`) já grava `orders`, `order_status_history` e débito/reposição de estoque; qualquer entrega HTTP síncrona ali é inviável ([09:04] Bruno) — um cliente lento travaria o fluxo de pedidos e um cliente fora do ar colocaria em risco a própria mudança de status.

## Proposta Técnica

**1. Outbox transacional no MySQL (ADR-001).** Dentro da mesma transação que muda o status do pedido, inserimos a linha de evento em `webhook_outbox` (payload snapshot, UUID, índices em status e `created_at`). Commit da transação ⇒ evento garantido; rollback ⇒ evento desaparece junto — consistência sem infraestrutura adicional ([09:06] Diego). O filtro de status por cliente é aplicado **na inserção**: se nenhum webhook do customer quer aquele status, a linha nem nasce ([09:34] Bruno).

**2. Worker separado em polling (ADR-002).** Processo Node distinto (`src/worker.ts`, script `npm run worker`), consumindo a outbox em polling de 2s, batch pequeno, single-worker com ordering implícito por `order_id` ([09:09]–[09:13] Diego). Latência mínima de 2s no pior caso — aceita contra o teto de 10s do negócio ([09:10] Larissa).

**3. Retry com DLQ (ADR-003).** Backoff exponencial 1m/5m/30m/2h/12h, 5 tentativas (~15h de janela — [09:17] Diego/Marcos). Esgotadas, o evento vai para a tabela `webhook_dead_letter` (payload, motivo, timestamp) e pode voltar via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role ADMIN com log de auditoria ([09:18] Diego, [09:36] Sofia/Larissa).

**4. Segurança (ADR-004).** HMAC-SHA256 sobre o corpo do request no header `X-Signature`; secret única por endpoint (não global); rotação de secret com grace period de 24h; URL obrigatoriamente `https` (recusa `http` via Zod); limite de payload de 64KB com erro ([09:20]–[09:24] Sofia/Diego).

**5. Semântica de entrega (ADR-005).** At-least-once com `X-Event-Id` (UUID por evento) para dedup no cliente — padrão de mercado, documentado no portal de desenvolvedores ([09:24]–[09:26] Diego/Marcos).

**6. Integração com a codebase (ADR-006/007).** Módulo `src/modules/webhooks` no layout padrão (controller/service/repository/routes/schemas); erros `WEBHOOK_*` herdando `AppError`; Pino e middleware de erro sem alteração; `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamado dentro da transação do `changeStatus` ([09:41] Bruno/Diego).

O detalhamento de implementação (schemas de dados, contratos HTTP completos, matriz de erros, fluxo do worker passo a passo) fica no [FDD](./FDD.md).

## Alternativas Consideradas

1. **Entrega HTTP síncrona dentro de `OrderService.changeStatus`** — descartada. A transação de status já é pesada (orders + histórico + estoque); uma chamada HTTP no meio faria um cliente lento travar mudanças de status de outros pedidos, e um cliente fora do ar criaria o dilema insolúvel do rollback da mudança de status ([09:04] Bruno, [09:06] Diego: "Síncrono está fora de questão"). *Trade-off: latência mínima vs. acoplamento da operação crítica de pedidos à disponibilidade de terceiros.*
2. **Redis Streams (fila externa)** — descartada. Exigiria subir e operar infraestrutura nova (Redis Cluster) para um time pequeno: overengineering, na avaliação de Diego ([09:07] Larissa/Diego). *Trade-off: reatividade e escalabilidade futura vs. custo operacional de uma segunda infraestrutura.*
3. **Notificação do worker via trigger de banco (reativo, sem polling)** — descartada. MySQL não tem NOTIFY/LISTEN nativo; triggers executam apenas SQL e não avisam processos externos — contornar isso via arquivo/endpoint foi considerado esquisito ([09:09] Bruno/Diego). *Trade-off: zero latência de polling vs. mecanismo inexistente no MySQL.*
4. **Exactly-once na entrega** — descartada. Exigiria coordenação dos dois lados (cliente e OMS) para confirmar processamento, muito mais complexa que at-least-once + dedup por `X-Event-Id` ([09:25] Diego). *Trade-off: sem duplicatas vs. protocolo proprietário pesado para os clientes.*

## Questões em Aberto

1. **Rate limiting de saída por cliente** — se um cliente tem 50 pedidos mudando de status em um minuto, o enviamos 50 chamadas? Diego propôs não tratar agora: observar em produção e implementar se virar problema. Registrado como "observar e decidir depois" ([09:38]–[09:39] Diego/Larissa).
2. **Escalonamento do worker para múltiplas instâncias** — a ordering por `order_id` só vale enquanto single-worker; escalar exige particionamento por `order_id` ou lock pessimista, explicitamente remanejado para o futuro ([09:13] Diego, limitação documentada por Larissa).
3. **Endurecimento de permissões no CRUD de webhooks** — hoje qualquer role autenticada opera o CRUD; Sofia sinalizou que "mais pra frente a gente pode endurecer" ([09:36]–[09:37] Sofia). Critério para o endurecimento não definido.
4. **Arquivamento da outbox** — linhas entregues seriam arquivadas após ~30 dias; Diego posicionou fora do escopo da feature ([09:08] Diego). Política de retenção e mecanismo a definir antes que a tabela cresça.

## Impacto e Riscos

**Impacto:**

- **Código**: novo módulo `src/modules/webhooks`, entry `src/worker.ts`, três tabelas novas (configuração, `webhook_outbox`, `webhook_dead_letter`) e um ponto de alteração em código existente: `OrderService.changeStatus` ganha a chamada a `publishWebhookEvent` dentro da transação ([09:40]–[09:41] Bruno). Nenhuma outra rota/módulo é tocada.
- **Operação**: um processo novo para deployar e monitorar (`npm run worker`), e um endpoint admin novo (replay de DLQ).
- **Clientes**: mudança de integração — de polling para recebimento push; precisam implementar verificação HMAC e dedup por `X-Event-Id`. Documentação no portal de desenvolvedores cobre isso ([09:26] Marcos).
- **Prazo**: 3 sprints, incluindo 2 dias úteis de revisão de segurança da Sofia ([09:46] Larissa/Sofia). Atlas espera entrega até fim de novembro ([09:45] Marcos).

**Riscos e mitigação:**

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente não implementar dedup e processar evento duplicado | Média | Médio | Contrato at-least-once documentado em destaque no portal ([09:26] Marcos); `X-Event-Id` estável por evento |
| Vazamento de secret do cliente (já ocorreu — [09:22] Diego) | Média | Alto | Secret por endpoint (não global), rotação com grace period de 24h ([09:21] Sofia) |
| Acúmulo de linhas na outbox degradando o polling | Média | Médio | Índices em status/`created_at`, batch pequeno ([09:08] Diego); arquivamento listado como questão em aberto |
| Requisição maliciosa/replay contra o cliente | Baixa | Médio | Assinatura HMAC + `X-Timestamp` para detecção de replay pelo cliente ([09:44] Diego/Sofia) |
| Prazo estourar frente à expectativa da Atlas | Média | Alto | Escopo duro com exclusões explícitas (email de alerta, dashboard, rate limiting fora); revisão de security agendada no fim das 3 sprints ([09:46]–[09:47]) |

## Decisões Relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em Processo Separado com Polling de 2 Segundos](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com Backoff Exponencial e DLQ](./adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com Secret por Endpoint](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](./adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos Padrões do Projeto](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Snapshot do Payload na Inserção](./adrs/ADR-007-snapshot-de-payload-na-insercao.md)

Documentos irmãos: [PRD](./PRD.md) (problema e escopo de produto) e [FDD](./FDD.md) (especificação de implementação).
