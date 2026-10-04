# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature
**Sistema de Webhooks de Notificação de Pedidos**, registrados a partir da reunião
técnica transcrita em [`TRANSCRICAO.md`](../../TRANSCRICAO.md).

Cada decisão arquitetural relevante é registrada em arquivo individual, no formato
MADR, nomeado como `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para eventos de mudança de status | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e Dead Letter Queue em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação | Aceito |
| [ADR-005](ADR-005-at-least-once-x-event-id.md) | Garantia at-least-once com deduplicação por X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto | Aceito |

Rastreabilidade de cada item à transcrição ou ao código: [`docs/TRACKER.md`](../TRACKER.md).
