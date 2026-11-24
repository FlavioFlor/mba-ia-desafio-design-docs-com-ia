# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto — especificamente, as decisões da feature **Sistema de Webhooks de Notificação de Pedidos**, registradas a partir da reunião técnica documentada em `TRANSCRICAO.md`.

Formato de nomenclatura: `ADR-NNN-titulo-em-kebab-case.md` (numeração sequencial a três dígitos).

Cada ADR segue o formato MADR simplificado com as seções: **Status, Contexto, Decisão, Alternativas Consideradas, Consequências** (positivas e negativas, com trade-off explícito), além de Referências e Relacionados.

## Índice

| ADR | Decisão |
| --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos |
| [ADR-003](./ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada |
| [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) | Garantia at-least-once com idempotência via X-Event-Id |
| [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](./ADR-007-snapshot-de-payload-na-insercao.md) | Snapshot do payload renderizado na inserção |
