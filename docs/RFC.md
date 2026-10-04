# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Alexandre Menezes |
| **Status** | Em revisão |
| **Data** | 2026-09-28 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Segurança) |
| **ADRs relacionados** | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) · [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) · [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) · [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |

## TL;DR

Propomos um sistema de **webhooks outbound** para notificar clientes B2B quando o status de um pedido muda. A arquitetura é **outbox no MySQL existente**: o evento é inserido na tabela `webhook_outbox` dentro da mesma transação do `changeStatus`; um **worker em processo separado** faz polling a cada 2s e entrega via HTTP com **HMAC-SHA256**, **retry com backoff (5 tentativas, ~15h)** e **DLQ** em tabela própria com replay manual por ADMIN. Garantia **at-least-once** com dedup por `X-Event-Id`. Nenhum componente de infra novo; reuso máximo dos padrões do projeto.

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem polling em `GET /orders` para detectar mudanças de status, o que é lento e caro; a Atlas ameaça migrar para concorrente se não receber a feature até fim do trimestre ([09:00] Marcos). O requisito de latência é < 10s ([09:02] Marcos). A aplicação não possui eventos, filas ou notificações externas, e a transação de mudança de status em `src/modules/orders/order.service.ts` já é pesada (orders + `order_status_history` + estoque), inviabilizando chamadas HTTP síncronas ([09:04] Bruno).

## Proposta técnica (visão geral)

Quatro componentes, todos sobre a stack existente (Node.js + TypeScript + Express + Prisma/MySQL):

1. **Publicação (outbox atômico)** — dentro da transação do `changeStatus`, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` insere o evento (payload renderizado em snapshot, `event_id` UUID) na tabela `webhook_outbox`, aplicando o filtro de status dos webhooks do customer no momento da inserção ([09:41] Bruno/Diego, [09:34] Bruno, [09:52] Larissa). Se a inserção falhar, a transação inteira sofre rollback ([09:40] Bruno). → [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
2. **Worker de entrega** — novo processo (`src/worker.ts`, script `npm run worker`), com PrismaClient próprio, que a cada 2s lê pendentes em batch pequeno (índices em status/`created_at`) e dispara o HTTP com timeout de 10s, assinando com HMAC-SHA256 (`X-Signature`) e enviando `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` ([09:09]–[09:11] Diego/Larissa, [09:42]–[09:45]). Single worker nesta fase; ordering por `order_id` implícita. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md), [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
3. **Retry e DLQ** — falhou: backoff exponencial 1m/5m/30m/2h/12h (5 tentativas, janela ~15h); esgotou: move para `webhook_dead_letter` (payload, motivo, timestamp). Replay manual via `POST /admin/webhooks/dead-letter/:id/replay` com `requireRole('ADMIN')` e log de auditoria ([09:15]–[09:18], [09:35]–[09:36]). → [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)
4. **API de configuração** — CRUD de webhooks (módulo `src/modules/webhooks/` no padrão existente): cadastro com URL `https` obrigatória e secret gerada pela plataforma (única por endpoint, rotação com grace period de 24h), filtro por status, histórico de entregas `GET /webhooks/:id/deliveries` ([09:21]–[09:23], [09:31]–[09:34]). → [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)

O detalhamento de implementação (contratos, payloads, matriz de erros, fluxos, observabilidade) está no [`FDD.md`](FDD.md).

## Alternativas consideradas

| Alternativa | Trade-off que levou ao descarte | Origem |
| --- | --- | --- |
| **Disparo síncrono no `OrderService`** (HTTP dentro da transação de mudança de status) | Cliente lento trava mudanças de status de outros pedidos; cliente offline cria dilema impossível de rollback; transação já é pesada | [09:03]–[09:04] Larissa/Bruno, [09:06] Diego |
| **Redis Streams / fila externa** | Exige subir infra nova (Redis Cluster) para um time pequeno — "overengineering"; outbox no MySQL existente resolve | [09:07] Larissa/Diego |
| **Trigger de banco para reatividade** | MySQL não tem `NOTIFY/LISTEN`; trigger só executa SQL, não notifica processo externo; improvisos (arquivo/endpoint) "fica esquisito" | [09:09] Bruno/Diego |
| **Garantia exactly-once** | Exigiria coordenação dos dois lados, muito mais complexo; at-least-once + `X-Event-Id` resolve 99% dos casos (padrão Stripe/GitHub) | [09:25] Diego |
| **Retry com 3 tentativas** | Pouco: indisponibilidade de manhã esgotaria as tentativas em ~30min; já houve cliente com 2h de manutenção planejada | [09:16] Bruno/Diego |
| **DLQ como flag `failed` na própria outbox** | Tabela separada mantém a outbox limpa e serve de evidência para debug/reprocessamento | [09:17]–[09:18] Larissa/Diego |

## Questões em aberto

1. **Rate limiting de saída**: se um cliente tiver 50 pedidos mudando de status em um minuto, vamos bombardeá-lo com 50 chamadas? Ficou decidido **observar e decidir depois** — não entra nesta fase, mas é ponto em aberto registrado ([09:38]–[09:39] Diego/Larissa).
2. **Endurecimento de autorização do CRUD de configuração**: por enquanto qualquer role autenticada pode gerenciar webhooks; "mais pra frente a gente pode endurecer" ([09:36]–[09:37] Marcos/Sofia).
3. **Escala do worker (múltiplos processos)**: como particionar por `order_id` ou usar lock pessimista sem perder ordering — "problema do futuro, não agora" ([09:12]–[09:13] Diego).
4. **Arquivamento da outbox**: linhas entregues seriam arquivadas após ~30 dias, "fora do escopo dessa feature" — quando e como fazer permanece aberto ([09:08] Diego).
5. **Notificação por e-mail de webhook com problema** (ex.: 3 falhas seguidas): adiada para a próxima fase, "depois que a gente medir o impacto" ([09:37]–[09:38] Marcos/Larissa).

## Impacto e riscos

- **Impacto no código existente**: pontual e cirúrgico — a transação `changeStatus` ganha uma chamada de publicação; rotas ganham o módulo `webhooks`; nenhuma alteração nos middlewares (`errorMiddleware` já cobre `AppError`). Novas tabelas no schema Prisma ([09:29]–[09:30], [09:40]–[09:41]).
- **Impacto operacional**: um processo novo para deploy/monitorar (worker); polling constante (consulta a cada 2s) no MySQL ([09:09], [09:11]).
- **Riscos principais**: vazamento de secret no lado do cliente (mitigado por secret por endpoint + rotação 24h, [09:21]–[09:22]); crescimento da outbox (mitigado por índices e batch, arquivamento adiado, [09:07]–[09:08]); cliente não deduplicar eventos (mitigado por documentação destacada no portal, [09:25]–[09:26]); prazo de 3 sprints com revisão de segurança inclusa ([09:46]–[09:47]). Detalhes no [`PRD.md`](PRD.md#10-riscos-e-mitigação).
- **Limitação conhecida**: sem garantia de ordering global; ordering apenas por `order_id` enquanto single-worker ([09:13] Larissa). Clientes nunca pediram ordering global ([09:14] Marcos).

## Decisões relacionadas

| Decisão | ADR |
| --- | --- |
| Outbox no MySQL, publicação atômica na transação do `changeStatus` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker em processo separado, polling de 2s, single-worker | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| Retry 5x com backoff 1m/5m/30m/2h/12h, DLQ separada, replay ADMIN | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação com grace period 24h | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com dedup por `X-Event-Id` | [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md) |
| Reuso dos padrões existentes (módulo, AppError/`WEBHOOK_*`, Pino, requireRole, Zod) | [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |
