# ADR-006 — Reuso dos Padrões Existentes do Projeto

## Status

Aceito (decidido na reunião técnica de novembro de 2026 — "Decisão: reuso máximo do que já existe." [09:30] Larissa; confirmado no resumo final [09:48])

## Contexto

A codebase tem padrões consolidados, descritos por Bruno na reunião ([09:27]–[09:29]): cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas; erros herdam de `AppError` com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`; o logger Pino cobre o projeto inteiro; o middleware de erro centralizado já trata `AppError`, Zod e Prisma sem configuração extra.

A feature de webhooks é o primeiro mecanismo assíncrono da aplicação (hoje não há eventos, filas ou notificações externas), o que abriria espaço para introduzir bibliotecas ou convenções novas. A reunião decidiu o contrário: integrar a feature nos padrões existentes em vez de criar paralelos.

## Decisão

O módulo de webhooks segue **exatamente o padrão de módulos existente** ([09:27] Bruno — "Webhook vai seguir igual", aprovado por Diego; [09:30] Larissa):

- **Estrutura**: pasta `src/modules/webhooks` com controller, service, repository, routes e schemas, igual aos módulos `orders`, `customers`, `products`, `users` e `auth` existentes em `src/modules/`.
- **Entry do worker**: `src/worker.ts` como entry separada (modelo: `src/server.ts`) e a lógica de processamento dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno, aprovado por Diego).
- **Erros**: novas classes seguindo o padrão de `AppError`/`http-errors.ts`, com código e **prefixo `WEBHOOK_`** para todo o módulo — exemplos citados na reunião: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
- **Logger**: Pino, o já usado no projeto — "Não vamos botar nada novo" ([09:29] Bruno).
- **Middleware de erro centralizado**: reaproveitado sem alteração — "Vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno).
- **Schemas Zod**: validações no mesmo padrão dos módulos existentes (ex.: recusa de URL `http` — [09:23] Sofia).
- **PrismaClient**: instância nova no worker, pois é processo separado; mesmo banco e mesma `DATABASE_URL` ([09:29]–[09:30] Diego/Bruno).
- **Autorização**: reuso de `authenticate` para os endpoints e de `requireRole` para o replay de DLQ com role ADMIN ([09:36] Larissa — "a gente reaproveita o requireRole que já existe").
- **UUID como id** das novas entidades, seguindo o padrão do restante do projeto ([09:51] Diego/Larissa).

## Alternativas Consideradas

1. **Introduzir biblioteca/framework externo de webhooks (ou job queue como BullMQ/Agenda)** — não adotada: a reunião foi explícita em evitar stack nova para o problema (o mesmo raciocínio que descartou Redis Streams em ADR-001 — [09:07] Diego: time pequeno, infra nova é overengineering). Trade-off: recursos prontos de scheduling/UI vs. dependência operacional e padrões paralelos aos do projeto.
2. **Módulo fora do layout padrão** (ex.: pasta `src/webhooks/` com estrutura própria) — descartada pela proposta de Bruno, endossada por Diego e formalizada por Larissa ([09:27]–[09:28], [09:30]). Trade-off: liberdade estrutural vs. consistência de navegação e onboarding no código.

## Consequências

**Positivas:**

- Curva de aprendizado zero para o time: qualquer dev que já leu `src/modules/orders` navega em `src/modules/webhooks` sem custo ([09:27] Bruno).
- Middleware de erro, logger, validação e autorização funcionam sem alteração — menos superfície de mudança e menos risco de regressão ([09:29] Bruno).
- Erros com prefixo `WEBHOOK_` são identificáveis na resposta HTTP pela mesma estrutura `{ error: { code, message } }` já consumida pelos clientes.

**Negativas:**

- Nenhuma ferramenta especializada de entrega/fila: tudo (retry, agendamento, polling) é construído à mão sobre MySQL e timers do Node — mais código próprio para manter.
- O middleware de erro permanece acoplado: se o worker (fora do Express) precisar de formato de erro diferente, o reuso não se aplica do mesmo jeito — o worker loga via Pino diretamente, sem passar pelo middleware ([09:29] Bruno menciona o middleware apenas para a API).
- Padrões de módulo pensados para CRUD síncrono precisam ser estendidos (repository com queries de polling, processor sem controller) — o padrão é seguido, mas extrapolado.

## Referências

- Transcrição: [09:27]–[09:30] (Bruno, Diego, Larissa) e [09:36] (requireRole), [09:51] (UUID).
- Código:
  - `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — classes `AppError`, `ConflictError`, `InvalidStatusTransitionError`, `InsufficientStockError` com códigos `INVALID_STATUS_TRANSITION`/`INSUFFICIENT_STOCK` citados por Bruno como modelo ([09:28]).
  - `src/middlewares/error.middleware.ts` — tratamento centralizado de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` ([09:29]).
  - `src/shared/logger/index.ts` — logger Pino com redaction de headers sensíveis ([09:29]).
  - `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole('ADMIN')` ([09:36]).
  - `src/app.ts` e `src/routes/index.ts` — composição de módulos (`buildControllers`/`buildApiRouter`) a estender com webhooks.
  - `src/server.ts` — entry-point a espelhar em `src/worker.ts` ([09:11] Larissa).
  - `prisma/schema.prisma` — todos os models usam `id String @id @default(uuid()) @db.Char(36)` ([09:51]).

## Relacionados

- ADR-002 (processo separado e PrismaClient por processo)
- ADR-001 (outbox como tabela Prisma adicional)
