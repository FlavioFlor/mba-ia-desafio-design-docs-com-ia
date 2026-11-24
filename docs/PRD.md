# PRD — Product Requirements Document

## Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks de notificação de mudança de status de pedidos (outbound) |
| **Autor** | Flavio Flor (elaboração assistida por IA, a partir da reunião técnica) |
| **Data** | 2026-11-24 |
| **Stakeholders de origem** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Segurança) |
| **Documentos irmãos** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

---

## 1. Resumo e Contexto

Este PRD define o Sistema de Webhooks de Notificação de Pedidos para o Order Management System (OMS) em produção. A feature notifica clientes B2B, via HTTP, sempre que o status de um pedido deles muda — substituindo o modelo atual em que cada cliente consulta periodicamente o `GET /orders` para descobrir mudanças.

A demanda nasceu de pedido formal de três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo — [09:00] Marcos). A decisão técnica completa foi discutida em reunião com tech lead, PM, engenharia e segurança (`TRANSCRICAO.md`), cujo conteúdo este documento consolida em linguagem de produto. O detalhamento técnico de implementação está no [FDD](./FDD.md); a proposta de arquitetura submetida à revisão, no [RFC](./RFC.md).

Contexto relevante do sistema: o OMS é uma API Node.js/TypeScript com máquina de estados de pedido (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, com CANCELLED), controle transacional de estoque e auditoria de status — e **não possui hoje nenhum mecanismo de notificação externa**, o vácuo que esta feature preenche.

## 2. Problema e Motivação

- **Integração lenta e cara para os clientes**: hoje eles "ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa" ([09:00] Marcos). O resultado é atraso na detecção de mudanças e desperdício de chamadas de API.
- **Risco de churn**: a Atlas Comercial sinalizou que, sem entrega até fim do trimestre, pode migrar para o concorrente ([09:00] Marcos).
- **Pressão de prazo**: a Atlas quer a feature até **fim de novembro** ([09:45] Marcos); a estimativa interna fechou em **3 sprints** ([09:46]–[09:47] Larissa).
- **Definição de sucesso do cliente**: para os três clientes, "tempo real" é qualquer latência **abaixo de 10 segundos** — o que importa é não ficar pendurado atualizando manualmente ([09:02] Marcos).

## 3. Público-alvo e Cenários de Uso

**Público-alvo:**

- **Primário — times de integração/TI dos clientes B2B** (Atlas Comercial, MaxDistribuição, Nova Cargo): recebem os eventos, verificam a assinatura HMAC e reagem à mudança de status (disparar separata, faturamento, logística) ([09:19] Sofia — o cliente valida origem e integridade).
- **Secundário — operadores e admins do OMS**: cadastram e mantêm webhooks via API ([09:31]–[09:33] Marcos/Bruno) e reprocessam eventos mortos via endpoint admin ([09:18] Diego, [09:36] Sofia).
- **Terciário — time de suporte/devrel**: documenta o contrato no portal de desenvolvedores ([09:26] Marcos, [09:40] Marcos).

**Cenários de uso:**

1. **Atlas quer saber quando o pedido vira SHIPPED e DELIVERED**: cadastra um webhook com filtro de status `["SHIPPED", "DELIVERED"]`; recebe POST assinado no endpoint dela a cada transição elegível, sem polling ([09:33] Marcos).
2. **MaxDistribuição em manutenção planejada de 2h** (caso real citado — [09:16] Diego): entregas falham; o sistema retenta com backoff crescente e, quando o endpoint volta, a entrega converge — o evento não se perde.
3. **Nova Cargo precisa provar que o evento veio da plataforma**: valida o header `X-Signature` (HMAC-SHA256) com a secret do endpoint e usa `X-Event-Id` para deduplicar reentregas ([09:20] Sofia, [09:25] Diego).
4. **Operador investiga integração quebrada**: consulta as últimas 100 entregas com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos); se o evento morreu, um ADMIN executa o replay pela DLQ ([09:18] Diego, [09:36] Sofia).
5. **Cliente troca de secret por segurança**: pede rotação pela API; a secret antiga continua válida por 24h enquanto ele migra os sistemas ([09:21] Sofia).

## 4. Objetivos e Métricas de Sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| O1 | Notificar mudanças de status em tempo real percebido | Latência entre commit da mudança e 1ª tentativa de entrega | **< 10 s para 100% dos eventos** (desenho: polling de 2 s — [09:02] Marcos, [09:09]–[09:10] Diego/Marcos) |
| O2 | Eliminar o polling manual dos clientes que solicitaram | Clientes B2B migrados de polling para webhook | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo — [09:00] Marcos) |
| O3 | Não perder eventos durante indisponibilidade do cliente | Eventos entregues dentro da janela de retry após falha transitória | **100%** dos eventos recuperáveis na janela de ~15 h ([09:17] Diego: 5 retentativas 1m/5m/30m/2h/12h) |
| O4 | Preservar eventos não entregues para recuperação | Eventos permanentemente falhos disponíveis para replay | **100%** retidos na DLQ com payload, motivo e timestamp ([09:18] Diego) |
| O5 | Entregar dentro do prazo acordado | Sprints consumidos até deploy | **≤ 3 sprints**, incluindo 2 dias úteis de revisão de segurança ([09:46]–[09:47] Larissa/Sofia) |
| O6 | Adesão dos clientes com verificação de integridade | Webhooks ativos com secret própria por endpoint | **1 secret por endpoint**, rotação suportada ([09:21] Sofia) |

## 5. Escopo

### 5.1 Incluído

- Webhooks **outbound** de mudança de status de pedidos ([09:02] Sofia/Marcos).
- CRUD de configuração de webhooks por cliente (url https, secret, filtro de status, estado ativo) ([09:31]–[09:33]).
- Entrega assíncrona transacional (outbox no MySQL) com worker em processo separado ([09:06], [09:11]).
- Retry com backoff exponencial e DLQ com replay manual admin ([09:15]–[09:18]).
- Segurança: HMAC-SHA256 por endpoint, rotação de secret com grace period de 24 h, TLS obrigatório ([09:19]–[09:24]).
- Histórico de entregas por webhook (últimas 100) ([09:34] Marcos).
- Documentação do contrato para os clientes no portal de desenvolvedores ([09:26], [09:40] Marcos).

### 5.2 Fora de Escopo

1. **Email de alerta ao cliente quando o webhook falha 3 vezes seguidas** — pedido de Marcos, recusado por Larissa: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." ([09:37]–[09:38]; anotado como "futuro" por Marcos).
2. **Dashboard visual para o cliente acompanhar webhooks** — "não, agora não. Só endpoints. Painel é projeto separado do time de frontend." ([09:39]–[09:40] Marcos/Larissa).
3. **Rate limiting de envio por cliente** — ponto em aberto: "a gente observa e implementa se virar problema" ([09:38]–[09:39] Diego/Larissa).
4. **Arquivamento/purga da outbox (~30 dias)** — "fora do escopo dessa feature" ([09:08] Diego).
5. **Escala para múltiplos workers** (particionamento por `order_id`, lock pessimista) — "problema do futuro, não agora" ([09:13] Diego).
6. **Webhooks inbound** (cliente enviando eventos para nós) — explicitamente excluído: "Só saindo da gente pra eles." ([09:02] Marcos/Sofia).

## 6. Requisitos Funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| FR-01 | O cliente pode **cadastrar um webhook** via endpoint POST, informando URL e lista de status de interesse; a **secret é gerada pelo sistema e devolvida na criação**; o `customerId` é informado no body/path (não derivado do JWT) | [09:31] Marcos, [09:32] Bruno/Larissa/Marcos |
| FR-02 | O cliente pode **editar** um webhook (PATCH) — URL, filtro de status, estado ativo | [09:33] Bruno |
| FR-03 | O cliente pode **remover** um webhook (DELETE) | [09:33] Bruno |
| FR-04 | O cliente pode **listar os webhooks de um customer** (GET) | [09:33] Bruno |
| FR-05 | Ao mudar o status de um pedido, o sistema **registra o evento na outbox dentro da mesma transação SQL** da mudança — status mudado sem evento registrado é impossível; se a inserção falhar, rollback | [09:06] Diego, [09:40]–[09:41] Bruno/Diego |
| FR-06 | O **filtro de eventos é aplicado na inserção**: se nenhum webhook do customer quer aquele status, a linha nem é criada | [09:33]–[09:34] Marcos/Bruno/Diego |
| FR-07 | A entrega é **assíncrona**, executada por worker em processo separado com polling de 2 s; latência mínima de 2 s no pior caso é aceita | [09:09]–[09:11] Diego/Marcos/Larissa |
| FR-08 | Entregas com falha são **retentadas com backoff exponencial** 1m/5m/30m/2h/12h (5 retentativas, ~15 h de janela) | [09:15]–[09:17] Diego/Bruno/Larissa |
| FR-09 | Esgotadas as tentativas, o evento vai para **DLQ persistida** (payload, motivo, timestamp) e pode voltar via **replay manual** em endpoint admin restrito a role ADMIN, com registro de quem executou | [09:18] Diego, [09:35]–[09:36] Larissa/Sofia |
| FR-10 | Toda entrega é **assinada com HMAC-SHA256** sobre o corpo, com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` | [09:20] Sofia, [09:44]–[09:45] Diego/Sofia |
| FR-11 | O cliente pode **rotacionar a secret** pela API; a anterior permanece válida por 24 h em paralelo (grace period) | [09:21] Sofia |
| FR-12 | O cliente pode **consultar o histórico de entregas** (últimas 100: sucesso/falha, payload, response, tempo de resposta) via `GET /webhooks/:id/deliveries` | [09:34] Marcos |
| FR-13 | O payload do evento contém `event_id`, `event_type` "order.status_changed", timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents` — **sem items** (detalhes via `GET /orders/:id`) | [09:43] Diego |
| FR-14 | CRUD de configuração acessível a **qualquer role autenticada** (por enquanto); replay de DLQ **exclusivo de ADMIN** | [09:36]–[09:37] Sofia/Larissa/Marcos |

## 7. Requisitos Não Funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| NFR-01 | **Latência**: 1ª tentativa de entrega em < 10 s após a mudança de status (polling de 2 s) | [09:02] Marcos, [09:09]–[09:10] Diego/Marcos |
| NFR-02 | **TLS obrigatório**: URL de webhook deve ser `https`; cadastro com `http` é recusado com erro de validação | [09:23] Sofia |
| NFR-03 | **Limite de payload: 64 KB** — eventos que excederem não são enviados (erro, não truncamento) | [09:23]–[09:24] Sofia/Diego/Larissa |
| NFR-04 | **Timeout HTTP de 10 s** por tentativa de entrega; estouro conta como falha e agenda retry | [09:42] Sofia/Diego |
| NFR-05 | **Semântica at-least-once**: duplicatas são possíveis; clientes deduplicam por `X-Event-Id` | [09:24]–[09:26] Diego/Sofia/Marcos |
| NFR-06 | **Ordering implícito por `order_id`** enquanto single-worker; ordering global não é garantido (limitação documentada) | [09:12]–[09:14] Diego/Larissa/Marcos |
| NFR-07 | **Isolamento operacional**: worker em processo separado da API; restart da API não interrompe entregas | [09:11] Diego |
| NFR-08 | **Consistência transacional**: registro do evento atômico com a mudança de status | [09:06] Diego, [09:40]–[09:41] Bruno |
| NFR-09 | **Segregação de segredos**: secret única por endpoint (não global), suporte a rotação com grace period | [09:21] Sofia |
| NFR-10 | **Reuso de stack**: MySQL/Prisma, Pino, Zod, `AppError` e middleware de erro existentes; códigos de erro com prefixo `WEBHOOK_`; nenhuma dependência/infra nova | [09:27]–[09:30] Bruno/Diego/Larissa |
| NFR-11 | **Auditoria**: replay de DLQ registra o operador que executou | [09:36] Sofia |
| NFR-12 | **Secret nunca exposta em listagens**; devolvida apenas na criação/rotação | [09:31] Marcos, [09:21] Sofia |

## 8. Decisões e Trade-offs Principais

| Decisão | Alternativa descartada | Trade-off aceito |
| --- | --- | --- |
| Outbox no MySQL, evento na mesma transação do status ([09:06] Diego) | HTTP síncrono no service ([09:04] Bruno); Redis Streams ([09:07] Larissa/Diego) | Sem infra nova e sem risco de inconsistência, ao custo de polling (latência mínima de 2 s) |
| Worker em processo separado, polling de 2 s ([09:09]–[09:11] Diego) | Trigger de banco (MySQL sem NOTIFY/LISTEN — [09:09]); worker embutido na API ([09:11] Diego) | Simplicidade e isolamento de ciclo de vida, ao custo de um processo a mais para operar e throughput limitado ao single-worker |
| 5 retentativas com backoff até ~15 h, depois DLQ ([09:15]–[09:17] Diego/Marcos) | 3 tentativas ([09:16] Bruno); retry indefinido ([09:15] Diego) | Cobre manutenções reais de clientes sem fila infinita, ao custo de até ~15 h de atraso até o descarte |
| DLQ em tabela separada ([09:18] Diego) | Marcar "failed" na própria outbox | Outbox limpa e evidência auditável, ao custo de mais uma tabela |
| HMAC-SHA256 com secret por endpoint ([09:20]–[09:21] Sofia) | Secret global da plataforma ([09:21] Sofia) | Vazamento isolado a um endpoint, ao custo de gestão de N secrets + endpoint de rotação |
| At-least-once + `X-Event-Id` ([09:25] Diego) | Exactly-once com coordenação dos dois lados ([09:25] Diego) | Simplicidade e padrão de mercado (Stripe/GitHub), ao custo de responsabilidade de dedup no cliente |
| Payload snapshot sem items ([09:43] Diego, [09:52] Larissa) | Renderizar no envio; incluir items | Evento fiel ao instante da transição e enxuto, ao custo de o cliente buscar detalhes em `GET /orders/:id` |
| E-mail de alerta e dashboard **excluídos** desta fase ([09:37]–[09:40] Larissa) | — | Escopo cabível em 3 sprints; alerta e painel avaliados após medir impacto |

## 9. Dependências

- **Banco MySQL existente** via Prisma — novas tabelas (outbox, DLQ, configuração, deliveries) ([09:06], [09:18], [09:21] Bruno).
- **Autenticação JWT e roles existentes** (`ADMIN`/`OPERATOR`) para CRUD e replay admin ([09:32], [09:36] Bruno/Sofia/Larissa).
- **Endpoint `GET /orders/:id`** permanece como fonte de detalhes do pedido (payload do webhook não inclui items) ([09:43] Diego).
- **Portal de desenvolvedores** (fora do repositório): Marcos documenta contrato de integração, dedup e assinatura em destaque ([09:26] Marcos, [09:40] Marcos).
- **Revisão de segurança**: Sofia reserva 2 dias úteis para revisar HMAC e geração de secret antes do deploy ([09:46] Sofia).
- **Infra de deploy**: capacidade de rodar um processo adicional (worker) junto à API, mesmo banco ([09:11] Bruno/Diego).
- **Clientes implementarem** verificação HMAC e dedup por `X-Event-Id` ([09:20] Sofia, [09:25] Diego).

## 10. Riscos e Mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R1 | **Vazamento de secret de cliente** (precedente real: cliente vazou secret em log de aplicação — [09:22] Diego) | Média | Alto | Secret por endpoint (não global) limitando raio de explosão ([09:21] Sofia); rotação com grace period de 24 h; secret devolvida uma única vez ([09:31]); revisão de segurança dedicada ([09:46] Sofia) |
| R2 | **Cliente não tratar duplicatas** (at-least-once) e processar pedido duas vezes | Média | Médio | Contrato documentado em destaque no portal ([09:26] Marcos); `X-Event-Id` estável por evento ([09:25] Diego) |
| R3 | **Prazo estourar** frente à expectativa da Atlas (fim de novembro) | Média | Alto | Escopo fechado com exclusões explícitas (email, dashboard, rate limiting); estimativa de 3 sprints com revisão de segurança embutida ([09:46]–[09:47] Larissa/Sofia) |
| R4 | **Acúmulo de eventos na outbox** degradar o worker (preocupação de [09:07] Bruno) | Média | Médio | Índices em status/`created_at` e batch pequeno ([09:08] Diego); arquivamento ~30 dias registrado como questão em aberto |
| R5 | **Cliente fora do ar por longa indisponibilidade** (> janela de ~15 h) | Baixa | Médio | DLQ preserva o evento para replay manual ([09:18] Diego); histórico de entregas permite diagnóstico ([09:34] Marcos) |
| R6 | **Replay indevido** comprometer fila de notificações | Baixa | Médio | Restrição a role ADMIN e log de auditoria de quem executou ([09:36] Sofia/Larissa) |

## 11. Critérios de Aceitação

1. Cliente cadastra webhook com URL `https` e filtro de status; recebe a secret na resposta ([09:31] Marcos).
2. Cadastro com URL `http` é recusado com erro de validação ([09:23] Sofia).
3. Mudança de status elegível gera evento entregue ao endpoint do cliente **em menos de 10 s**, com headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e payload conforme especificado ([09:02] Marcos, [09:43]–[09:45] Diego/Sofia).
4. Mudança de status cujo `toStatus` não interessa a nenhum webhook do customer **não gera evento** ([09:34] Bruno).
5. Mudança de status com falha forçada na inserção da outbox **não é aplicada** (rollback) — nunca existe status mudado sem evento ([09:40]–[09:41] Bruno/Diego).
6. Endpoint offline retorna à vida dentro da janela de retry ⇒ evento entregue sem intervenção ([09:15]–[09:17] Diego).
7. Evento com 6 tentativas falhadas aparece na DLQ com payload, motivo e timestamp; ADMIN executa replay e o cliente o recebe; OPERATOR recebe 403 no replay ([09:18] Diego, [09:36] Sofia).
8. Rotação de secret devolve nova secret; a anterior valida assinaturas por 24 h ([09:21] Sofia).
9. `GET /webhooks/:id/deliveries` retorna as últimas 100 entregas com sucesso/falha, payload, response e tempo de resposta ([09:34] Marcos).
10. Reiniciar a API **não interrompe** a entrega de eventos (worker é processo separado) ([09:11] Diego).
11. Assinatura HMAC-SHA256 verificável com a secret do endpoint ([09:20] Sofia).
12. Nenhum item dos "Fora de escopo" (§5.2) aparece na entrega.

## 12. Estratégia de Testes e Validação

**Funcional (vitest + supertest, padrão já usado em `tests/`):**

- CRUD de webhooks: criação com devolução única de secret, edição, remoção, listagem por customer; recusa de URL `http` e de status inválidos no filtro.
- Autorização: CRUD com qualquer role autenticada; replay permitido apenas a ADMIN, com log de auditoria ([09:36] Sofia).
- Filtro na inserção: transições não inscritas não geram linhas ([09:34] Bruno).

**Transacional (o teste mais crítico da feature):**

- Mudança de status commitada ⇒ linhas na outbox; falha forçada na inserção ⇒ rollback da mudança ([09:40]–[09:41] Bruno/Diego).
- Caso de regressão: débito de estoque + histórico + outbox atômicos (suite existente de `tests/orders.test.ts` como rede de segurança).

**Worker/entrega (teste de processo isolado, endpoint mockado):**

- Polling de 2 s, timeout de 10 s, agenda de backoff 1m/5m/30m/2h/12h (relógio controlado), move para DLQ após esgotar tentativas, replay volta a PENDING ([09:09], [09:42], [09:17]–[09:18]).
- Assinatura HMAC-SHA256 recomputável pelo receptor com a secret; headers completos ([09:20], [09:44]–[09:45]).
- At-least-once: replay preserva `event_id` ([09:25] Diego).

**Segurança (revisão humana + testes):**

- 2 dias úteis de revisão da Sofia focados em HMAC e geração de secret antes do deploy ([09:46] Sofia).
- Testes negativos: assinatura inválida rejeitada pelo receptor; secret antiga aceita dentro de 24 h e rejeitada após ([09:21] Sofia).
- Rotação com cliente em migration (grace period).

**Validação com clientes (piloto):**

- Portal de desenvolvedores com o contrato documentado em destaque — dedup por `X-Event-Id` e verificação de assinatura ([09:26] Marcos).
- Integração assistida dos 3 clientes B2B, com histórico de entregas como ferramenta de debug compartilhada ([09:34] Marcos).

---

**Rastreabilidade**: cada requisito e decisão deste PRD tem linha correspondente no [TRACKER.md](./TRACKER.md), com a fala exata da transcrição ou o arquivo do código como origem.
