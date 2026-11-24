# ADR-002 — Worker em Processo Separado com Polling de 2 Segundos

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — registrado por Larissa em [09:10] e confirmado no resumo final [09:48])

## Contexto

Definido o outbox no MySQL (ADR-001), restava definir como o evento chega ao cliente. O requisito de negócio é percebido como "tempo real": qualquer latência abaixo de 10 segundos atende os clientes B2B ([09:01]–[09:02] Bruno/Marcos). A reunião discutiu ser mais reativo usando trigger do banco; MySQL não possui listener nativo tipo NOTIFY/LISTEN do Postgres, e triggers executam apenas SQL — não notificam processos externos. Notificar o worker exigiria improvisos (escrever em arquivo ou bater num endpoint), considerados esquisitos ([09:09] Bruno/Diego).

Além disso, o worker não pode rodar dentro da instância da API: se a API reinicia, o worker seria perdido junto ([09:11] Diego). O projeto já tem o padrão de entry-point em `src/server.ts`, o que sugere um `src/worker.ts` análogo ([09:11] Larissa).

## Decisão

O worker roda como **processo Node separado**, consumindo o outbox por **polling em loop a cada 2 segundos**: busca os eventos pendentes mais antigos, processa e marca como entregue ([09:09] Diego, [09:10] Marcos — "2 segundos serve, perfeito").

Detalhes:

- Entry-point nova `src/worker.ts` e script `npm run worker`, seguindo o modelo de `src/server.ts` ([09:11] Larissa).
- Mesmo banco, mesma stack (Prisma), mas **PrismaClient em instância nova**, pois é outro processo Node — "PrismaClient é por processo" ([09:11] Bruno, [09:29]–[09:30] Diego/Bruno).
- Processamento em batch pequeno de eventos pendentes ([09:08] Diego).
- **Single-worker**: com um único worker, o processamento segue a ordem de `created_at` do outbox, garantindo ordering implícito por `order_id`. Escala para múltiplos workers perderia essa garantia — documentado como limitação conhecida, não bug ([09:12]–[09:13] Diego/Larissa). Caminho futuro: particionar por `order_id` ou lock pessimista ([09:13] Diego).
- Latência mínima no pior caso: 2 segundos — aceita explicitamente ([09:10] Larissa).

## Alternativas Consideradas

1. **Trigger de banco para notificar o worker** — descartada: MySQL não tem NOTIFY/LISTEN nativo; trigger executa apenas SQL dentro do banco e não avisa processo externo. Fazer ponte via arquivo ou endpoint HTTP foi considerado improvável/esquisito ([09:09] Diego). Trade-off: reatividade total vs. dependência de mecanismo que não existe no MySQL.
2. **Worker embutido na instância da API** (ex.: `setInterval` dentro do processo Express) — descartada: reinício/deploys da API derrubariam o consumidor de eventos junto, e a proposta explícita da reunião foi processo separado ([09:11] Diego). Trade-off: um deploy a menos vs. acoplamento de ciclo de vida entre API e entrega de webhooks.

## Consequências

**Positivas:**

- Simplicidade operacional: polling de 2s cumpre o requisito "abaixo de 10 segundos" com folga ([09:09] Diego).
- Ciclo de vida independente: deploys/reinícios da API não interrompem a entrega de eventos; deploy do worker não reinicia a API.
- Ordering por `order_id` preservado enquanto single-worker ([09:12] Diego).
- Sem infraestrutura nova (nenhum broker, nenhum listener externo).

**Negativas:**

- Latência mínima de 2 segundos no pior caso — aceita, mas é um piso ([09:10] Larissa).
- Polling gera queries periódicas no banco mesmo sem eventos (custo fixo baixo, porém contínuo).
- Single-worker é um limite de throughput conhecido: escalar exige particionamento por `order_id` ou lock pessimista, problema remanejado para o futuro ([09:13] Diego).
- Ordering global não é garantido — apenas por `order_id` e apenas enquanto single-worker ([09:13] Larissa). Os clientes nunca pediram ordering global ([09:14] Marcos).

## Referências

- Transcrição: [09:08]–[09:13] (Diego, Bruno, Marcos, Larissa).
- Código: `src/server.ts` (modelo de entry-point), `src/config/database.ts` (fábrica de PrismaClient), `package.json` (scripts existentes).

## Relacionados

- ADR-001 (outbox que o worker consome)
- ADR-003 (retry e DLQ no processamento)
- ADR-006 (reuso de padrões — PrismaClient por processo)
