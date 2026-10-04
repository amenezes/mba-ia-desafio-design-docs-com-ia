# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos (outbound) |
| **Autor** | Alexandre Menezes |
| **Data** | 2026-09-28 |
| **Status** | Proposto |
| **Documentos relacionados** | [`PRD.md`](PRD.md) · [`RFC.md`](RFC.md) · [ADRs](adrs/) · [`TRACKER.md`](TRACKER.md) |

> Este FDD detalha o "como implementar". As decisões arquiteturais e seus porquês estão nos ADRs; requisitos de produto, no PRD. Toda informação é rastreável à [`TRANSCRICAO.md`](../TRANSCRICAO.md) ou ao código — ver [`TRACKER.md`](TRACKER.md).

## 1. Contexto e motivação técnica

O OMS precisa notificar clientes B2B quando o status de um pedido muda, com latência < 10s ([09:02] Marcos), sem acoplar o fluxo crítico de pedidos à disponibilidade de endpoints externos ([09:04] Bruno). A solução adotada é o padrão **outbox no MySQL** com **worker separado em polling** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

A aplicação é Node.js ≥ 20 + TypeScript (ESM), Express 4, Prisma 5 (MySQL), Zod, Pino, vitest (`package.json`). As rotas são montadas sob `/api/v1` em `src/app.ts`. O estado do pedido é controlado pela máquina de estados de `src/modules/orders/order.status.ts` (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` acessível de `PENDING/PAID/PROCESSING`).

## 2. Objetivos técnicos

1. Publicar eventos de mudança de status de forma **atômica** com a transação existente do `changeStatus` ([09:40]–[09:41] Bruno/Diego).
2. Entregar eventos via HTTP com **HMAC-SHA256**, headers padronizados e timeout de 10s ([09:20] Sofia, [09:42] Diego, [09:44]–[09:45] Diego/Sofia).
3. **Retry** com backoff exponencial (1m/5m/30m/2h/12h) e **DLQ** com replay manual ADMIN ([09:17]–[09:18] Diego, [09:36] Larissa/Sofia).
4. CRUD de configuração de webhooks com secret por endpoint, rotação (grace 24h) e filtro por status ([09:21] Sofia, [09:31]–[09:33] Marcos/Bruno).
5. Histórico de entregas consultável pelo cliente ([09:34] Marcos).
6. **Reuso máximo** dos padrões existentes — módulo em `src/modules/webhooks/`, `AppError` com códigos `WEBHOOK_*`, Pino, `errorMiddleware`, `requireRole`, Zod ([09:27]–[09:30], [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)).

## 3. Escopo e exclusões

**Escopo de implementação**: módulo `src/modules/webhooks/` (controller, service, repository, routes, schemas, worker/processor), entry-point `src/worker.ts` + script `npm run worker` ([09:11] Larissa, [09:28] Bruno), 4 novas tabelas Prisma, extensão pontual da transação `changeStatus`, novos schemas Zod e classes de erro `WEBHOOK_*`.

**Exclusões** (ver também [PRD §5](PRD.md#5-escopo)): e-mail de alerta ([09:37] Larissa), dashboard visual ([09:39]–[09:40] Larissa/Marcos), rate limiting de saída ([09:38]–[09:39] Diego/Larissa), múltiplos workers/ordering global ([09:12]–[09:13] Diego), arquivamento da outbox após ~30 dias ([09:08] Diego), webhooks inbound ([09:02] Sofia/Marcos).

## 4. Modelo de dados (Prisma)

Novos models em `prisma/schema.prisma`, seguindo as convenções existentes (`id String @id @default(uuid()) @db.Char(36)`, `createdAt`/`updatedAt`, `@@map` snake_case, índices explícitos). UUID é o padrão do projeto ([09:51] Larissa).

```prisma
model WebhookEndpoint {
  id               String   @id @default(uuid()) @db.Char(36)
  customerId       String   @db.Char(36)
  url              String   @db.VarChar(2048)        // https obrigatório ([09:23] Sofia)
  secret           String   @db.VarChar(255)         // secret atual ([09:21] Sofia)
  previousSecret   String?  @db.VarChar(255)         // secret antiga durante grace period
  previousSecretExpiresAt DateTime?                  // now()+24h na rotação ([09:21] Sofia)
  eventStatuses    Json                              // lista de OrderStatus que o endpoint quer ([09:31] Marcos)
  active           Boolean  @default(true)           // estado ativo ([09:21] Bruno)
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId])
  @@map("webhook_endpoints")
}

model WebhookOutboxEvent {
  id            String    @id @default(uuid()) @db.Char(36)  // == event_id ([09:25] Diego)
  webhookId     String    @db.Char(36)
  orderId       String    @db.Char(36)
  eventType     String    @db.VarChar(100)      // "order.status_changed" ([09:43] Diego)
  payload       Json                            // snapshot renderizado na inserção ([09:52] Larissa)
  status        OutboxStatus @default(PENDING)  // pendente/processando/falhou/entregue ([09:08] Diego)
  attempts      Int       @default(0)           // máx. 5 ([09:15] Diego)
  nextAttemptAt DateTime  @default(now())       // agenda o backoff ([09:17] Diego)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([status, createdAt])   // leitura do worker em batch ([09:08] Diego)
  @@index([orderId])             // ordering por order_id ([09:12] Diego)
  @@map("webhook_outbox")
}

enum OutboxStatus {
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model WebhookDeadLetter {
  id         String   @id @default(uuid()) @db.Char(36)
  eventId    String   @unique @db.Char(36)
  webhookId  String   @db.Char(36)
  payload    Json                            // payload ([09:18] Diego)
  reason     String   @db.VarChar(500)       // motivo da falha ([09:18] Diego)
  failedAt   DateTime @default(now())        // timestamp ([09:18] Diego)
  replayedAt DateTime?                       // preenchido no replay ([09:18] Diego)
  replayedById String? @db.Char(36)          // auditoria ([09:36] Sofia)

  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  eventId        String   @db.Char(36)
  webhookId      String   @db.Char(36)
  attempt        Int
  success        Boolean
  requestPayload Json                          // payload enviado ([09:34] Marcos)
  responseStatus Int?
  responseBody   String?  @db.Text             // response ([09:34] Marcos)
  responseTimeMs Int?                          // tempo de resposta ([09:34] Marcos)
  errorCode      String?  @db.VarChar(50)      // WEBHOOK_* quando falha
  createdAt      DateTime @default(now())

  @@index([webhookId, createdAt])   // GET /webhooks/:id/deliveries ([09:34] Marcos)
  @@map("webhook_deliveries")
}
```

Notas:

- Uma linha de outbox por **(evento × webhook endpoint)**: o filtro por status é aplicado na inserção — "se nenhum webhook do customer quer aquele status, nem insere" ([09:34] Bruno).
- `previousSecret`/`previousSecretExpiresAt` implementam o grace period de 24h da rotação ([09:21] Sofia).
- O nome `webhook_dead_letter` é o citado na reunião ([09:18] Diego); `webhook_outbox` idem ([09:06] Diego).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (publicação)

Dentro da transação existente do `changeStatus` (`src/modules/orders/order.service.ts`):

```
changeStatus(id, input, userId):
  $transaction(tx):
    ... validações existentes (canTransition, estoque) ...
    tx.order.update(status: to)
    tx.orderStatusHistory.create(...)
    await publishWebhookEvent(tx, order, from, to)     # NOVO — [09:41] Bruno
    return refreshed
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` (função pura que recebe o `TxClient` — [09:41] Bruno/Diego):

1. Busca `WebhookEndpoint` ativos do `order.customerId` cujo `eventStatuses` contenha `toStatus`.
2. Se nenhum: retorna sem inserir (filtro na inserção — [09:34] Bruno/Diego).
3. Para cada endpoint compatível: gera `event_id` (UUID — [09:25] Diego, [09:51] Larissa), renderiza o **payload snapshot** ([09:52] Larissa) e insere `WebhookOutboxEvent` com `status = PENDING`, `nextAttemptAt = now()`.
4. Se a inserção falhar, a exceção propaga e a transação inteira sofre rollback — "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno, [09:41] Diego).

Payload renderizado (campos definidos em [09:43] Diego; **sem `items`** — cliente consulta `GET /orders/:id` se precisar):

```json
{
  "event_id": "0d6f...uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-10T14:03:22.512Z",
  "order_id": "9b2c...uuid",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "4f1a...uuid",
  "total_cents": 15990
}
```

### 5.2 Processamento pelo worker

Loop em `src/worker.ts` → `webhook.processor.ts` ([09:28] Bruno), a cada **2 segundos** ([09:09] Diego):

1. `SELECT` em batch pequeno dos eventos `PENDING` com `nextAttemptAt <= now()`, ordenados por `createdAt ASC` (single worker ⇒ ordering por `order_id` — [09:12] Diego).
2. Marca `PROCESSING` (evita reprocessamento em caso de reinício).
3. Para cada evento: carrega o `WebhookEndpoint`; serializa o payload; valida tamanho ≤ **64KB** (estouro ⇒ falha `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncar — [09:23]–[09:24]); calcula `X-Signature = HMAC-SHA256(secret, corpo)` ([09:20] Sofia).
4. `POST` na URL do endpoint com timeout de **10s** ([09:42] Diego) e headers:
   - `Content-Type: application/json`
   - `X-Event-Id: <uuid>` ([09:25] Diego)
   - `X-Signature: <hmac hex>` ([09:20] Sofia)
   - `X-Timestamp: <iso8601 do envio>` (deteção de replay attack pelo cliente — [09:44] Diego)
   - `X-Webhook-Id: <id do endpoint>` ([09:44]–[09:45] Sofia/Diego)
5. Resposta `2xx` ⇒ `status = DELIVERED`. Não-2xx, timeout ou erro de rede ⇒ falha (vai para 5.3). Em todo caso, grava `WebhookDelivery` (attempt, success, payload, responseStatus, responseBody, responseTimeMs) para o histórico ([09:34] Marcos).

### 5.3 Retry com backoff

Falha na entrega ([09:15]–[09:17] Diego):

1. `attempts += 1`.
2. Se `attempts < 5`: agenda `nextAttemptAt = now() + backoff[attempts]`, com progressão fixa **1min → 5min → 30min → 2h → 12h** ([09:17] Diego); `status = PENDING` novamente.
3. Se `attempts >= 5`: falha permanente ⇒ fluxo 5.4.

### 5.4 DLQ e replay

1. Esgotadas as 5 tentativas: insere `WebhookDeadLetter` (payload, `reason` com o último erro, `failedAt`) e marca o evento `FAILED` na outbox ([09:15], [09:18] Diego).
2. **Replay manual** por ADMIN: `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18], [09:35] Diego):
   - rota protegida por `authenticate` + `requireRole('ADMIN')` ([09:36] Larissa);
   - recria o evento na outbox como `PENDING` (`attempts = 0`);
   - marca `replayedAt`/`replayedById` na DLQ e **loga o autor** via Pino para auditoria ([09:36] Sofia).

## 6. Contratos públicos

Rotas montadas sob `/api/v1` (padrão de `src/app.ts`/`src/routes/index.ts`); todas autenticadas com JWT Bearer (`authenticate` de `src/middlewares/auth.middleware.ts`); erros no formato do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`. O `customer_id` é passado no body/path, não vem do JWT ([09:32] Larissa).

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook (RF-01/RF-02)

Request:

```json
{
  "customerId": "4f1a...uuid",
  "url": "https://api.atlascomercial.com.br/hooks/pedidos",
  "eventStatuses": ["SHIPPED", "DELIVERED"]
}
```

- `url` obrigatoriamente `https` (Zod; `http` ⇒ 400 `WEBHOOK_INVALID_URL` — [09:23] Sofia).
- `eventStatuses`: subconjunto não vazio do enum `OrderStatus` (inválido ⇒ 400 `WEBHOOK_INVALID_EVENT_FILTER`).
- A **secret é gerada pela plataforma** e devolvida somente na criação ([09:31] Marcos).

Response `201`:

```json
{
  "id": "c0ff...uuid",
  "customerId": "4f1a...uuid",
  "url": "https://api.atlascomercial.com.br/hooks/pedidos",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_9f3a...gerada",
  "active": true,
  "createdAt": "2026-11-01T12:00:00.000Z"
}
```

Erros: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENT_FILTER`, `401 UNAUTHORIZED`, `404 NOT_FOUND` (customer inexistente).

### 6.2 `GET /api/v1/webhooks?customerId=<uuid>` — listar webhooks do customer (RF-04)

Response `200` (paginado via `paginated()` de `src/shared/http/response.ts`):

```json
{
  "data": [
    {
      "id": "c0ff...uuid",
      "customerId": "4f1a...uuid",
      "url": "https://api.atlascomercial.com.br/hooks/pedidos",
      "eventStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-01T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

A `secret` **nunca** é retornada em listagens/edições (só na criação e na rotação). Erros: `401 UNAUTHORIZED`.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar (RF-04)

Request (parcial): `{ "url": "https://...", "eventStatuses": ["DELIVERED"], "active": false }`.
Response `200`: representação atualizada (sem secret). Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENT_FILTER`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover (RF-04)

Response `204` (sem corpo). Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotação de secret (RF-11)

Response `200`:

```json
{
  "id": "c0ff...uuid",
  "secret": "whsec_nova...gerada",
  "previousSecretExpiresAt": "2026-11-02T12:00:00.000Z"
}
```

Semântica: a secret antiga permanece válida por **24h** em paralelo ([09:21] Sofia); o worker assina com a nova e, na verificação pelo cliente, ambas são aceitas dentro da janela. Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas (RF-09)

Response `200` (paginado; padrão "últimas 100" — [09:34] Marcos):

```json
{
  "data": [
    {
      "id": "d311...uuid",
      "eventId": "0d6f...uuid",
      "attempt": 2,
      "success": true,
      "requestPayload": { "event_id": "0d6f...uuid", "event_type": "order.status_changed", "...": "..." },
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "responseTimeMs": 143,
      "errorCode": null,
      "createdAt": "2026-11-10T14:03:24.900Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ (RF-10)

Protegido por `requireRole('ADMIN')` ([09:36] Larissa). Response `202`:

```json
{ "eventId": "0d6f...uuid", "outboxStatus": "PENDING", "replayedById": "a912...uuid" }
```

Erros: `403 FORBIDDEN` (role não-ADMIN), `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.

### 6.8 Contrato outbound (plataforma → cliente)

`POST <url do endpoint>` com o payload do §5.1 e headers do §5.2. Cliente deve responder `2xx` em até **10s** ([09:42] Diego) e **deduplicar por `X-Event-Id`** ([09:25] Diego, documentado no portal — [09:26] Marcos). Verificação de assinatura pelo cliente: `HMAC-SHA256(secret_atual OU secret_anterior_em_grace, corpo_exato_do_request) == X-Signature` ([09:20]–[09:21] Sofia).

## 7. Matriz de erros previstos (prefixo `WEBHOOK_*`)

Padrão decidido em [09:28]–[09:29] Bruno/Larissa, no estilo de `INSUFFICIENT_STOCK`/`INVALID_STATUS_TRANSITION` de `src/shared/errors/http-errors.ts`.

| Código | HTTP | Quando | Origem |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | webhook/endpoint inexistente | [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 | URL inválida ou não-`https` | [09:28] Bruno, [09:23] Sofia |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação exige secret e ela não está presente/válida | [09:28] Bruno |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `eventStatuses` vazio ou com status fora do enum | [09:31]–[09:33] Marcos (filtro por lista de status) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (falha de entrega) | payload do evento > 64KB; erro, não truncamento | [09:23]–[09:24] Sofia/Diego/Larissa |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (falha de entrega) | cliente não respondeu em 10s | [09:42] Diego |
| `WEBHOOK_DELIVERY_FAILED` | — (falha de entrega) | não-2xx/erro de rede; alimenta retry e, no limite, a DLQ | [09:15]–[09:18] Diego |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay de DLQ com id inexistente | [09:18] Diego (endpoint de replay) |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | replay de entrada já reprocessada | [09:18] Diego (replay "recoloca na outbox como pendente") |

Os três últimos códigos (`WEBHOOK_INVALID_EVENT_FILTER`, `WEBHOOK_DEAD_LETTER_*`) não foram citados nominalmente na reunião; derivam diretamente de requisitos decididos (filtro por status, endpoint de replay) e seguem a regra "prefixo `WEBHOOK_` pra tudo do módulo" ([09:29] Larissa).

## 8. Estratégias de resiliência

| Mecanismo | Especificação | Origem |
| --- | --- | --- |
| Timeout de entrega | 10s por chamada HTTP | [09:42] Diego |
| Retry | 5 tentativas no total | [09:15]–[09:16] Diego |
| Backoff | exponencial fixo: 1m, 5m, 30m, 2h, 12h (janela ~15h) | [09:17] Diego |
| Fallback | após 5ª falha ⇒ DLQ persistida com motivo; recuperação via replay manual ADMIN | [09:15], [09:18], [09:35]–[09:36] Diego/Sofia/Larissa |
| Atomicidade | evento e mudança de status na mesma transação; rollback conjunto | [09:06] Diego, [09:40]–[09:41] Bruno/Diego |
| Restart do worker | eventos `PROCESSING` órfãos retornam a `PENDING` na inicialização (at-least-once — duplicatas cobertas por `X-Event-Id`) | [09:11] Diego (processo separado), [09:24]–[09:25] Diego |
| Payload anômalo | > 64KB ⇒ falha imediata sem truncamento | [09:23]–[09:24] |
| Latência de pico | rate limiting de saída **não** implementado nesta fase; observar | [09:38]–[09:39] Diego/Larissa |

## 9. Observabilidade

Padrão: **Pino** (`src/shared/logger/index.ts`), já usado em todo o projeto — "não vamos botar nada novo" ([09:29] Bruno).

**Métricas** (emitidas/derivadas do estado no MySQL, sem stack nova):

- `webhook_outbox_pending` — eventos `PENDING` (tamanho da fila; alerta se crescer).
- `webhook_delivery_total{success}` — entregas ok vs. falhas por período.
- `webhook_delivery_duration_ms` — tempo de resposta das entregas (já persistido em `WebhookDelivery.responseTimeMs` — [09:34] Marcos).
- `webhook_retry_count{attempt}` — tentativas de retry por evento.
- `webhook_dlq_size` — entradas não reprocessadas na DLQ.
- `webhook_payload_rejected_total` — entregas recusadas por > 64KB.

**Logs** (Pino, estruturados):

- Publicação: `event_id`, `order_id`, `webhook_id`, `to_status` (na transação do `changeStatus`).
- Entrega: `event_id`, `webhook_id`, `attempt`, `response_status`, `response_time_ms`, `error_code` quando falha.
- DLQ: entrada (`reason`) e replay (`replayedById` — auditoria exigida em [09:36] Sofia).
- **Redaction**: estender `redact.paths` do logger para incluir `*.secret`, `*.previousSecret` e o header `x-signature`, no padrão dos redactions existentes (`*.password`, `*.token`); segredos nunca aparecem em log ([09:22] Diego relatou vazamento de secret em log de cliente).

**Tracing**:

- O `X-Event-Id` (UUID do evento) é a **chave de correlação ponta a ponta**: publicação na outbox → ciclos de retry → entrega → DLQ → replay. Todos os logs e a `WebhookDelivery` carregam `event_id`, permitindo reconstruir a jornada de um evento ([09:25] Diego define o identificador único; o uso como chave de correlação é a operacionalização dessa decisão).
- Na API HTTP, mantém-se o `requestId` do `request-logger.middleware.ts` (pino-http) existente para os endpoints de configuração.

## 10. Dependências e compatibilidade

- **Sem dependências novas obrigatórias**: Node ≥ 20 tem `crypto` nativo para HMAC-SHA256 e `fetch` global para o HTTP do worker (`package.json` atual: express 4.21, @prisma/client 5.22, zod 3.23, pino 9.5).
- **Banco**: migration Prisma aditiva (4 tabelas novas + relação em `Customer`); nenhum model existente é alterado — compatível com rollback de deploy.
- **Processos**: deploy passa a ter 2 processos (API + worker, `npm run worker`) compartilhando a mesma `DATABASE_URL`, cada um com seu `PrismaClient` ([09:11] Bruno, [09:30] Bruno).
- **Configuração**: novas variáveis no schema Zod de `src/config/env.ts` com defaults decididos na reunião: `WORKER_POLL_INTERVAL_MS=2000` ([09:09]), `WEBHOOK_DELIVERY_TIMEOUT_MS=10000` ([09:42]), `WEBHOOK_MAX_PAYLOAD_KB=64` ([09:24]).
- **Clientes externos**: a deduplicação por `X-Event-Id` e a verificação de assinatura são responsabilidades do cliente, documentadas no portal de desenvolvedor ([09:25]–[09:26] Diego/Marcos).

## 11. Integração com o sistema existente

Seção obrigatória do desafio: como o módulo de webhooks se conecta ao código real do repositório base.

| # | Caminho real | Como integra |
| --- | --- | --- |
| 1 | `src/modules/orders/order.service.ts` | O método `changeStatus` é **estendido**: dentro do `$transaction` existente, após `tx.orderStatusHistory.create(...)`, chama `publishWebhookEvent(tx, order, from, to)` recebendo o `TxClient` da transação ([09:41] Bruno: "função pura recebendo o tx. Não precisa injetar repository inteiro" — [09:41] Diego). Falha na inserção do outbox ⇒ rollback de tudo ([09:40] Bruno). Nenhuma outra parte do serviço muda. |
| 2 | `prisma/schema.prisma` | Recebe os 4 models novos (§4) seguindo as convenções já usadas nos models existentes (`Order`, `OrderStatusHistory`): ids `@default(uuid()) @db.Char(36)`, `@@map` snake_case, `@@index` explícitos; reusa o enum `OrderStatus` existente em `eventStatuses`/`from_status`/`to_status`. |
| 3 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | As classes de erro existentes são **reutilizadas como base**: novos erros (`WebhookNotFoundError`, `WebhookInvalidUrlError`, ...) estendem `AppError`/`NotFoundError`/`ValidationError` com códigos `WEBHOOK_*`, no padrão de `InsufficientStockError` e `InvalidStatusTransitionError` ([09:28] Bruno). |
| 4 | `src/middlewares/error.middleware.ts` | **Nenhuma mudança**: já trata `AppError`, `ZodError` e erros Prisma; os erros `WEBHOOK_*` nascem compatíveis — "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno). |
| 5 | `src/middlewares/auth.middleware.ts` | Reuso de `authenticate` em todas as rotas de webhooks e de `requireRole('ADMIN')` na rota de replay da DLQ ([09:36] Larissa: "a gente reaproveita o requireRole que já existe"). |
| 6 | `src/modules/orders/order.status.ts` | Fonte canônica dos status (`OrderStatus`) usados no filtro de eventos (`eventStatuses`) e no payload (`from_status`/`to_status`); a máquina de estados garante que só transições válidas geram eventos. |
| 7 | `src/routes/index.ts` e `src/app.ts` | `buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(...))` e a rota admin de DLQ, no mesmo padrão de montagem dos módulos existentes sob `/api/v1`. |
| 8 | `src/server.ts` | Modelo para a nova entry-point `src/worker.ts` ([09:11] Larissa: "tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script npm run worker"), usando `createPrismaClient()` de `src/config/database.ts` — instância própria do client por processo ([09:30] Bruno). |
| 9 | `src/shared/logger/index.ts` | Logger Pino reutilizado pelo worker e pelo módulo ([09:29] Bruno); única alteração: estender `redact.paths` para segredos de webhook (§9). |
| 10 | `src/middlewares/validate.middleware.ts` e `src/modules/*/**.schemas.ts` | Validação Zod das rotas de webhooks no padrão existente (`webhook.schemas.ts`); a exigência de `https` entra como refinamento do schema de URL ([09:23] Sofia: "é só uma validação no schema Zod"). |
| 11 | `tests/orders.test.ts` e `tests/helpers/factories.ts` | Padrão de testes (vitest + supertest + factories) a ser seguido pelos novos testes de webhooks; o fluxo de `changeStatus` já testado ganha casos de publicação na outbox. |

## 12. Critérios de aceite técnicos

- **T-01**: rollback da transação do `changeStatus` ⇒ zero linhas na `webhook_outbox`; commit ⇒ exatamente uma linha por endpoint compatível com o filtro ([09:40]–[09:41], [09:34]).
- **T-02**: evento publicado com status fora do `eventStatuses` do endpoint ⇒ nenhuma linha inserida ([09:34] Bruno).
- **T-03**: entrega bem-sucedida em < 10s a partir do polling (pior caso 2s + tempo de HTTP) ([09:02], [09:09]).
- **T-04**: `X-Signature` verificável com HMAC-SHA256(secret, corpo exato); durante grace period, aceite também com `previousSecret` ([09:20]–[09:21]).
- **T-05**: sequência de falhas respeita `nextAttemptAt` ≈ 1m/5m/30m/2h/12h; após 5ª falha há linha na `webhook_dead_letter` com `reason` ([09:17]–[09:18]).
- **T-06**: replay por não-ADMIN ⇒ `403`; por ADMIN ⇒ evento volta `PENDING` com `attempts = 0` e log de auditoria com o autor ([09:36]).
- **T-07**: cadastro com `http://` ⇒ `400 WEBHOOK_INVALID_URL` ([09:23]).
- **T-08**: payload > 64KB ⇒ entrega falha com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncamento ([09:23]–[09:24]).
- **T-09**: `GET /webhooks/:id/deliveries` retorna payload, responseStatus, responseBody e responseTimeMs por tentativa ([09:34]).
- **T-10**: reinício do worker não perde eventos (`PROCESSING` órfãos voltam a `PENDING`; duplicatas dedupáveis pelo cliente via `X-Event-Id`) ([09:11], [09:25]).
- **T-11**: segredos nunca aparecem em logs (redaction Pino) ([09:22], [09:29]).

## 13. Riscos e mitigação (técnicos)

| Risco | Mitigação | Origem |
| --- | --- | --- |
| Crescimento da `webhook_outbox` degrada o polling | Índices `[status, createdAt]`; batch pequeno; arquivamento após 30 dias planejado (fora do escopo atual) | [09:07]–[09:08] Bruno/Diego |
| Worker único vira gargalo/ponto único | Aceito nesta fase (single worker); escala futura com particionamento por `order_id` ou lock pessimista | [09:12]–[09:13] Diego |
| Duplicatas por timeout (cliente processou, resposta perdeu-se) | At-least-once + `X-Event-Id` para dedup no cliente; documentação destacada no portal | [09:24]–[09:26] Diego/Sofia/Marcos |
| Vazamento de secret | Secret por endpoint, rotação com grace 24h, redaction em logs | [09:21]–[09:22] Sofia/Diego |
| Rajadas de eventos para o mesmo cliente (50 pedidos/min) | Não mitigado nesta fase — observação e decisão posterior (rate limiting) | [09:38]–[09:39] Diego/Larissa |
| Replay indevido de DLQ | Role ADMIN + auditoria de autoria | [09:36] Sofia/Larissa |
