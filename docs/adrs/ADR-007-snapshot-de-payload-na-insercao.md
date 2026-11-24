# ADR-007 — Snapshot do Payload Renderizado na Inserção da Outbox

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — "Bom, snapshot. Decidido." [09:52] Bruno, após proposta de Larissa endossada por Diego)

## Contexto

Encerrando a reunião, Bruno levantou a dúvida de modelagem que faltava: o evento da outbox guarda o **payload renderizado** ou apenas `order_id` para renderizar na hora do envio? ([09:51] Bruno). A diferença é semântica: entre a mudança de status e a entrega, o pedido pode mudar de novo (PENDING → PAID → PROCESSING em sequência rápida — o próprio cenário discutido em [09:12] Larissa/Diego). Renderizar no envio faria o evento refletir um estado posterior ao da transição que o gerou — "senão tem caso esquisito" ([09:52] Larissa).

Na mesma conversa final, Diego confirmou com Larissa que o id da outbox será **UUID**, seguindo o padrão do restante do projeto ([09:51] Diego/Larissa — "Tudo é uuid").

## Decisão

A linha da `webhook_outbox` armazena o **payload do evento já renderizado no momento da inserção** (snapshot), dentro da mesma transação da mudança de status ([09:52] Larissa/Diego/Bruno). O id do evento é um **UUID** gerado na inserção — o mesmo valor enviado depois como `X-Event-Id` (ADR-005).

Formato do payload fechado na reunião ([09:43] Diego, endossado por Bruno [09:44] "mantém payload enxuto"):

```json
{
  "event_id": "<uuid>",
  "event_type": "order.status_changed",
  "timestamp": "<ISO 8601>",
  "order_id": "<uuid>",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "<uuid>",
  "total_cents": 12345
}
```

Sem items no payload: "Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois." ([09:43] Diego) — o que alinha com o limite de 64KB (ADR-004).

## Alternativas Consideradas

1. **Guardar apenas `order_id` (e status) e renderizar o payload na hora do envio** — descartada: o payload passaria a refletir o estado do pedido no momento da entrega, não o da transição; se o pedido mudar depois, o evento conta uma história diferente da que disparou ([09:52] Larissa). Trade-off: linha menor na outbox vs. semântica do evento incorreta e leitura extra de `orders` no worker para cada envio.
2. **Id auto incremental na outbox** — descartada: contraria o padrão do projeto, em que todas as entidades usam UUID ([09:51] Diego/Larissa). Trade-off: ordenação natural por chave e chave menor vs. exposição de sequência identificável externamente e inconsistência com o restante do schema.

## Consequências

**Positivas:**

- O evento entregue é imutável e reflete exatamente o estado no instante da mudança de status — auditoria e debug têm o dado fiel ([09:52] Larissa).
- O worker não precisa reler `orders` para entregar: menos queries no caminho quente do polling.
- UUID como `event_id` serve simultaneamente de chave de dedup (`X-Event-Id`) e de identificador em conversas de suporte (ADR-005).

**Negativas:**

- Cada linha da outbox carrega o payload completo: tabela maior, e o limite de 64KB (ADR-004) precisa ser verificado na inserção, quando o snapshot é montado.
- O formato do payload fica congelado no snapshot: mudanças de schema de eventos não se aplicam retroativamente a eventos pendentes/dormindo em retry.

## Referências

- Transcrição: [09:51]–[09:52] (Diego, Larissa, Bruno) e [09:43]–[09:44] (formato do payload).
- Código: `prisma/schema.prisma` (campos `orderNumber`, `totalCents`, `customerId`, `status` do model `Order` que alimentam o snapshot), `src/modules/orders/order.service.ts` (`changeStatus` monta o snapshot dentro da transação via `publishWebhookEvent(tx, order, fromStatus, toStatus)` — [09:41] Bruno).

## Relacionados

- ADR-001 (outbox e transação de inserção)
- ADR-004 (limite de 64KB do payload)
- ADR-005 (event_id/X-Event-Id)
