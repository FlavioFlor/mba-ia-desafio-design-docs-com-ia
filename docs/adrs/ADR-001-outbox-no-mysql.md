# ADR-001 — Padrão Outbox no MySQL

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — confirmado por Larissa em [09:07] e no resumo final [09:48])

## Contexto

A feature de webhooks precisa disparar notificações HTTP quando o status de um pedido muda. Hoje a transação de mudança de status em `src/modules/orders/order.service.ts` (`changeStatus`) já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos do pedido ([09:04] Bruno). Adicionar uma chamada HTTP síncrona dentro dessa transação foi descartado: um cliente lento travaria mudanças de status para outros pedidos, e um cliente fora do ar colocaria a questão impossível de "dar rollback na mudança de status?" ([09:04] Bruno, [09:06] Diego — "Síncrono está fora de questão").

A alternativa levantada na reunião foi uma fila externa (Redis Streams), mas exigiria subir infraestrutura nova para um time pequeno — classificada como overengineering ([09:07] Larissa, [09:07] Diego). A aplicação já roda MySQL via Prisma (`prisma/schema.prisma`), e o padrão transacional com `prisma.$transaction` já é usado em todo o fluxo de pedidos.

## Decisão

Adotar o **padrão Outbox no MySQL existente**: dentro da mesma transação SQL que atualiza `orders` e insere em `order_status_history`, o `changeStatus` também insere uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego).

Propriedades resultantes:

- Se a transação principal comita, o evento foi registrado; se deu rollback, o evento some junto. Não há inconsistência possível ([09:06] Diego).
- A tabela terá índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`; o worker lê apenas os pendentes em batch pequeno, processa e marca como entregue ([09:08] Diego).
- Se nenhuma configuração de webhook do cliente interessa para aquele status, a linha nem é inserida (filtro na inserção — [09:34] Bruno, confirmado por Diego).
- Arquivamento de linhas entregues após ~30 dias fica **fora do escopo** desta fase ([09:08] Diego).

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro de `OrderService.changeStatus`** — descartada: qualquer cliente lento ou offline bloquearia ou invalidaria a transação de mudança de status, que já é pesada (orders + histórico + estoque) ([09:04] Bruno). Trade-off: menor latência pontual vs. acoplamento da operação crítica de pedidos à disponibilidade de terceiros.
2. **Redis Streams (ou fila externa equivalente)** — descartada: exigiria subir e operar infraestrutura adicional (Redis Cluster) para um time pequeno, além de introduzir uma segunda tecnologia de persistência de eventos ([09:07] Larissa/Diego). Trade-off: maior reatividade vs. custo operacional e complexidade nova.

## Consequências

**Positivas:**

- Garantia transacional de consistência entre mudança de status e registro do evento, sem infraestrutura nova.
- Reuso do MySQL/Prisma já operados pelo time; o modelo transacional de `order.service.ts` permanece o mesmo, apenas com um insert a mais.
- A leitura do outbox é barata: índices em status e `created_at` e processamento em batch pequeno.

**Negativas:**

- O MySQL não notifica processos externos; o worker precisa fazer polling (ver ADR-002), com latência mínima na casa de segundos.
- A tabela `webhook_outbox` cresce continuamente; a estratégia de arquivamento (30 dias) foi adiada, então o crescimento é um risco operacional monitorável ([09:07] Bruno perguntou exatamente isso).
- O payload do evento fica materializado no banco (ver ADR-007), aumentando o volume por linha.

## Referências

- Transcrição: [09:03]–[09:08] (Larissa, Bruno, Diego).
- Código: `src/modules/orders/order.service.ts` (transação de `changeStatus`), `prisma/schema.prisma` (modelos `Order` e `OrderStatusHistory` transacionados hoje).

## Relacionados

- ADR-002 (worker que consome o outbox)
- ADR-003 (retry e DLQ)
- ADR-007 (snapshot do payload na inserção)
