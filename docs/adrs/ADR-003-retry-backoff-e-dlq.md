# ADR-003 — Retry com Backoff Exponencial e DLQ em Tabela Separada

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h" [09:17] Larissa; confirmado no resumo final [09:48])

## Contexto

Quando o endpoint do cliente está offline ou lento, o evento não pode simplesmente ser descartado — o cliente perde a notificação que motivou a feature. Mas também não pode ficar pendurado para sempre. A reunião debateu o número de tentativas: Bruno propôs 3 (mais agressivo), Diego defendeu 5, argumentando que 3 tentativas cobririam só ~30 minutos — insuficiente porque já houve cliente com indisponibilidade de 2 horas em manutenção planejada ([09:15]–[09:16] Bruno/Diego). Retry indefinido com backoff foi mencionado e rejeitado por deixar eventos pendurados para sempre caso o cliente desapareça ([09:15] Diego).

Marcos validou a janela final: um cliente fora do ar por 15 horas "já tá com problema sério dele" — aceitável ([09:17] Marcos). Sobre onde parar os eventos mortos, Diego propôs tabela separada contra a alternativa de marcar "failed" na própria outbox ([09:18] Larissa/Diego).

## Decisão

**Retry com backoff exponencial: 5 tentativas** com progressão **1 min → 5 min → 30 min → 2 h → 12 h**, totalizando ~15 horas entre a primeira falha e a última tentativa ([09:15]–[09:17] Diego/Larissa). Esgotadas as tentativas, o evento é considerado **falha permanente** e movido para a **DLQ: tabela `webhook_dead_letter` separada**, com payload, motivo da falha e timestamp ([09:18] Diego) — mantendo a outbox principal limpa para leitura e servindo de evidência para debug e reprocessamento.

**Replay manual**: endpoint admin `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como pendente ([09:18] Diego). O endpoint exige role ADMIN do JWT e registra quem fez o replay para auditoria ([09:36] Sofia/Larissa — reusando o `requireRole` existente).

## Alternativas Consideradas

1. **3 tentativas (backoff mais agressivo)** — descartada: retentaria 3 vezes em ~30 minutos e "mataria" o evento; já existiu cliente com 2 horas de indisponibilidade em manutenção planejada ([09:16] Diego). Trade-off: notificação de falha permanente mais rápida vs. taxa de falsos "mortos" durante manutenções longas de clientes.
2. **Retry indefinido com backoff crescente** — descartada: evento fica pendurado para sempre se o cliente sumiu; o custo de armazenar e reprocessar cresce sem controle ([09:15] Diego). Trade-off: nunca perder evento vs. fila imprevisível e acumulação infinita.
3. **Marcar como "failed" na própria `webhook_outbox` (sem tabela de DLQ)** — descartada: mistura eventos vivos e mortos na tabela que o worker faz polling, sujando a leitura dos pendentes; a tabela separada é mais limpa e serve de evidência para debug/reprocessamento ([09:18] Diego). Trade-off: menos uma tabela vs. queries de polling mais caras e difícil auditoria do que falhou.

## Consequências

**Positivas:**

- Janela de ~15h cobre manutenções planejadas reais de clientes (caso citado: 2h) com margem ([09:16]–[09:17]).
- DLQ separada mantém o polling do worker enxuto (só pendentes) e vira fonte de evidência para debug ([09:18] Diego).
- Replay manual via endpoint admin com auditoria de quem executou ([09:36] Sofia).
- Evento permanente não é descartado: pode ser recuperado por decisão humana.

**Negativas:**

- Um evento problemático pode levar ~15 horas até ser declarado morto e chegar à DLQ — diagnóstico de integração quebrada é lento se ninguém observar as métricas.
- Replay é operação manual: sem alerta proativo (email de falha ficou fora de escopo — [09:37]–[09:38] Marcos/Larissa), a descoberta depende de monitoramento ou reclamação do cliente.
- Mais uma tabela para modelar, migrar e manter (`webhook_dead_letter`).

## Referências

- Transcrição: [09:14]–[09:18] (Larissa, Diego, Bruno, Marcos) e [09:35]–[09:36] (Sofia/Larissa sobre ADMIN e auditoria).
- Código: `src/middlewares/auth.middleware.ts` (`requireRole` a reaproveitar para o replay), `src/shared/errors/http-errors.ts` (padrão de classes de erro para falhas de validação).

## Relacionados

- ADR-001 (outbox onde o evento nasce)
- ADR-002 (worker que executa as tentativas)
- ADR-006 (reuso do `requireRole`)
