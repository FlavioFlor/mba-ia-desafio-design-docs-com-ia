# ADR-005 — Garantia At-least-once com Idempotência via X-Event-Id

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." [09:26] Larissa; confirmado no resumo final [09:48])

## Contexto

Com retry automático (ADR-003), o mesmo evento pode ser entregue mais de uma vez: o timeout do worker estoura, o evento é reagendado, mas o cliente na verdade já havia recebido e processado a notificação. A reunião assumiu isso como consequência natural do desenho: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado." ([09:24] Diego). A alternativa teórica, entrega exactly-once, foi avaliada e descartada: "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo." ([09:25] Diego). Sofia apontou o efeito colateral: "Isso joga responsabilidade pro cliente." ([09:25] Sofia) — aceito como trade-off, com o respaldo de que é o padrão de mercado: Stripe e GitHub fazem assim ([09:25] Diego).

## Decisão

Adotar **garantia at-least-once** para a entrega, com **deduplicação do lado do cliente via `X-Event-Id`**: um UUID único por evento, gerado quando o evento entra na outbox e enviado no header `X-Event-Id` ([09:24]–[09:25] Diego). Se o cliente recebeu o mesmo `X-Event-Id` duas vezes, descarta a duplicata.

A medida complementar é de documentação: Marcos documentará o contrato (at-least-once + dedup) "bem destacado no portal de desenvolvedor" para os clientes ([09:26] Marcos).

## Alternativas Consideradas

1. **Exactly-once com coordenação bidirecional** (ex.: protocolo de ack/commit em duas fases com o cliente) — descartada: exigiria que os endpoints dos clientes implementassem coordenação de entrega, multiplicando a complexidade da integração para todos os clientes ([09:25] Diego). Trade-off: sem duplicatas vs. protocolo proprietário complexo que nenhum cliente B2B típico aceitaria implementar.
2. **Deduplicação no lado do servidor com tabela de acks por evento** — variante de coordenação: o worker só marcaria "entregue" após confirmação inequívoca, mas confirmar processamento (vs. recebimento) continua impossível sem cooperação do cliente; aproxima-se da alternativa 1 com o mesmo custo. Trade-off: aparente garantia forte vs. mesmo problema fundamental deslocado para o banco de dados.

## Consequências

**Positivas:**

- Desenho simples e resiliente: retries são seguros, nenhuma coordenação entre as partes.
- Padrão consolidado de mercado (Stripe, GitHub — [09:25] Diego), familiar para times de integração dos clientes.
- `X-Event-Id` dá ao cliente uma chave de idempotência direta e ao suporte um identificador rastreável em conversas de debug.

**Negativas:**

- Os clientes precisam implementar dedup por `X-Event-Id`; a responsabilidade é deles ([09:25] Sofia).
- Duplicatas realmente ocorrerão em produção (timeout pós-processamento é inevitável) — a documentação no portal precisa ser explícita para evitar incidentes de "evento processado duas vezes" ([09:26] Marcos).

## Referências

- Transcrição: [09:24]–[09:26] (Diego, Bruno, Sofia, Marcos, Larissa).
- Código: `prisma/schema.prisma` (UUID como padrão de id do projeto — ver ADR-007), `package.json` (dependência `uuid` já presente).

## Relacionados

- ADR-003 (retry, causa raiz das duplicatas)
- ADR-007 (UUID gerado na inserção da outbox)
