# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos (outbound) |
| **Produto** | Order Management System (OMS) |
| **Autor** | Alexandre Menezes |
| **Data** | 2026-09-28 |
| **Status** | Proposto |
| **Origem** | Reunião técnica de ~55 min — [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Stakeholders** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Segurança) |

> Toda informação deste PRD é rastreável à transcrição da reunião ou ao código existente. Ver [`TRACKER.md`](TRACKER.md).

---

## 1. Resumo e contexto da feature

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente para ser notificados em tempo real quando o status dos pedidos deles muda na plataforma ([09:00] Marcos). Hoje eles fazem *polling* no `GET /orders` de tempos em tempos, o que torna a integração lenta e cara para o lado deles ([09:00] Marcos).

A feature é um **sistema de webhooks outbound** (a notificação só sai da plataforma para o cliente; o cliente não envia webhooks para nós — [09:02] Sofia e Marcos) que notifica o cliente quando o status de um pedido dele muda. A aplicação atual **não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks** — este PRD descreve a construção dessa capacidade do zero, reaproveitando os padrões existentes ([ADR-006](adrs/ADR-006-reuso-padroes-existentes.md)).

## 2. Problema e motivação

- **Problema do cliente**: *polling* em `GET /orders` é lento e caro; o cliente não sabe em tempo real quando um pedido muda de status ([09:00] Marcos).
- **Risco de negócio**: a Atlas sinalizou que, se a feature não for entregue até o fim do trimestre, **pode migrar para um concorrente** ([09:00] Marcos).
- **Motivação técnica**: disparar HTTP sincronamente na transação de mudança de status é inviável — a transação já é pesada (atualiza `orders`, insere em `order_status_history`, mexe no estoque) e um cliente lento ou fora do ar travaria o fluxo crítico ([09:04] Bruno). Daí a necessidade de um mecanismo desacoplado e confiável ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 3. Público-alvo e cenários de uso

**Público-alvo**: clientes B2B integrados via API (Atlas Comercial, MaxDistribuição, Nova Cargo — [09:00] Marcos), representados por usuários autenticados por JWT do nosso sistema ([09:32] Marcos e Larissa).

**Cenários de uso**:

1. **Cadastro**: o cliente registra um endpoint de webhook informando a URL e a lista de status que deseja receber; a plataforma gera a secret e a devolve na criação ([09:31] Marcos).
2. **Notificação de mudança de status**: quando um pedido do cliente muda de status, a plataforma entrega um evento assinado no endpoint cadastrado, em menos de 10 segundos ([09:02] Marcos, [09:06] Diego).
3. **Filtro por status**: o cliente que só quer saber de `SHIPPED` e `DELIVERED` recebe apenas esses eventos ([09:33] Marcos).
4. **Consulta de histórico**: o cliente consulta as últimas entregas (sucesso/falha, payload, response, tempo de resposta) ([09:34] Marcos).
5. **Rotação de secret**: o cliente pede uma nova secret; a antiga segue válida por 24h para migração ([09:21] Sofia).
6. **Reprocessamento (admin)**: um ADMIN replaya um evento que caiu na DLQ ([09:18] Diego, [09:36] Sofia).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- |
| Entrega em "tempo real" perceptível | Latência entre mudança de status e entrega no cliente | **< 10 segundos** (pior caso de polling: 2s) | [09:02] Marcos, [09:09]–[09:10] Diego/Larissa |
| Confiabilidade de entrega | Eventos entregues com sucesso na 1ª janela (antes de DLQ) | Cobrir indisponibilidades de até ~15h via retry | [09:15]–[09:17] Diego/Marcos |
| Integridade/autenticidade | Entregas assinadas e validáveis pelo cliente | 100% das entregas com HMAC-SHA256 (`X-Signature`) | [09:20] Sofia |
| Reter a Atlas | Feature entregue dentro do prazo | **Fim de novembro** (~3 sprints) | [09:45]–[09:47] Marcos/Larissa |

Objetivo primário quantitativo: **latência de entrega < 10 segundos** (requisito explícito de "tempo real" dos clientes — [09:02] Marcos).

## 5. Escopo

### Incluso

- CRUD de configuração de webhooks (criar, listar, editar, remover) por customer ([09:31]–[09:33] Marcos/Bruno).
- Geração e rotação de secret por endpoint, com grace period de 24h ([09:21] Sofia).
- Filtro de eventos por status, aplicado na inserção na outbox ([09:33]–[09:34] Marcos/Bruno/Diego).
- Publicação atômica de eventos na outbox dentro da transação de `changeStatus` ([09:40] Bruno).
- Worker em processo separado, polling de 2s ([09:09]–[09:11] Diego/Larissa).
- Entrega HTTP com assinatura HMAC-SHA256 e headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) ([09:20] Sofia, [09:44]–[09:45] Diego/Sofia).
- Retry com backoff exponencial (5 retentativas, ~15h) e DLQ em tabela separada ([09:15]–[09:18] Diego).
- Histórico de entregas (`GET /webhooks/:id/deliveries`) ([09:34] Marcos).
- Endpoint admin de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), role ADMIN, com auditoria ([09:18] Diego, [09:36] Sofia/Larissa).

### Fora de escopo (descartado ou adiado na reunião)

1. **Notificação por e-mail ao cliente quando o webhook falha** — explicitamente **adiado** para a próxima fase, depois de medido o impacto ([09:37] Larissa: "Email tá fora de escopo dessa fase. Talvez próxima fase"; [09:38] Marcos: "anotado como 'futuro'").
2. **Dashboard/painel visual** para o cliente ver os webhooks — **descartado** nesta fase; "painel é projeto separado do time de frontend", a integração é via API documentada no portal ([09:39]–[09:40] Larissa e Marcos).
3. **Rate limiting de saída** (limitar rajadas de envio a um cliente com muitos pedidos mudando) — **adiado**: "a gente observa e implementa se virar problema", registrado como ponto em aberto ([09:38]–[09:39] Diego e Larissa).
4. **Garantia de ordering global / múltiplos workers em paralelo** — **adiado**: single-worker nesta fase; particionamento por `order_id` ou lock pessimista "é problema do futuro" ([09:12]–[09:13] Diego e Larissa).
5. **Webhooks inbound** (cliente enviando webhooks para a plataforma) — **descartado**: escopo é somente outbound ([09:02] Sofia e Marcos).
6. **Arquivamento de linhas entregues na outbox** (após ~30 dias) — **fora do escopo** desta feature ([09:08] Diego).

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **RF-01** | O cliente cadastra um webhook via `POST`, informando a URL; a secret é **gerada pela plataforma** e devolvida na criação | [09:31] Marcos |
| **RF-02** | No cadastro, o cliente informa a **lista de status** que deseja receber (filtro de eventos) | [09:31] Marcos, [09:33] Marcos |
| **RF-03** | O `customer_id` é passado no body ou no path (não vem do JWT, que é do usuário operador) | [09:32] Bruno, Marcos e Larissa |
| **RF-04** | O cliente pode **editar** (`PATCH`), **remover** (`DELETE`) e **listar** (`GET`) os webhooks de um customer | [09:33] Bruno |
| **RF-05** | O filtro de eventos é aplicado **na inserção na outbox**: se nenhum webhook do customer quer aquele status, o evento nem é inserido | [09:34] Bruno, [09:34] Diego |
| **RF-06** | Quando o status de um pedido muda, um evento é publicado na outbox **na mesma transação** (atomicidade com `changeStatus`) | [09:40] Bruno, [09:41] Diego |
| **RF-07** | O worker entrega o evento ao endpoint do cliente com payload JSON assinado e headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`) | [09:43]–[09:45] Diego e Sofia |
| **RF-08** | Em falha de entrega, o sistema faz **retry com backoff exponencial** (5 retentativas após o envio original, intervalos 1m/5m/30m/2h/12h) e, esgotadas, move o evento para a **DLQ** | [09:15]–[09:18] Diego |
| **RF-09** | O cliente consulta o **histórico de entregas** (`GET /webhooks/:id/deliveries`): últimas ~100, com sucesso/falha, payload, response e tempo de resposta | [09:34] Marcos |
| **RF-10** | Um **ADMIN** pode **replayar** um evento da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), que volta à outbox como pendente; a ação é logada para auditoria | [09:18] Diego, [09:36] Sofia e Larissa |
| **RF-11** | O cliente pode **rotacionar a secret**; durante a rotação, a secret antiga permanece válida por **24h** em paralelo | [09:21] Sofia |
| **RF-12** | Garantia **at-least-once**: o cliente deduplica eventos repetidos pelo `X-Event-Id` (UUID único por evento) | [09:24]–[09:25] Diego |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **RNF-01** | Latência de entrega < 10 segundos (pior caso de polling: 2s) | [09:02] Marcos, [09:09]–[09:10] Diego/Larissa |
| **RNF-02** | **TLS obrigatório**: URL do webhook deve ser `https`; cadastro com `http` é recusado com erro de validação (schema Zod) | [09:23] Sofia |
| **RNF-03** | Autenticidade/integridade: toda entrega assinada com **HMAC-SHA256** sobre o corpo do request | [09:20] Sofia |
| **RNF-04** | **Limite de payload de 64KB**; acima disso a entrega falha (não trunca) | [09:23]–[09:24] Sofia/Diego/Larissa |
| **RNF-05** | **Timeout de 10s** na chamada HTTP do worker; estouro é tratado como falha e vai para retry | [09:42] Sofia/Diego |
| **RNF-06** | Consistência: se a transação de mudança de status sofre rollback, o evento não é publicado (e vice-versa) | [09:06] Diego, [09:40]–[09:41] Bruno/Diego |
| **RNF-07** | Segurança de segredos: secret única por endpoint; segredos não são logados (redaction Pino) | [09:21] Sofia, [09:22] Diego |
| **RNF-08** | Autorização: replay de DLQ exige role `ADMIN` (reuso do `requireRole`); demais endpoints de configuração exigem autenticação | [09:36] Larissa/Sofia, [09:36]–[09:37] Marcos/Sofia |
| **RNF-09** | Resiliência da outbox: índice em status e `created_at`; worker lê pendentes em batch pequeno | [09:08] Diego |
| **RNF-10** | Prazo de entrega: fim de novembro (~3 sprints, incluindo revisão de segurança de Sofia) | [09:45]–[09:47] Marcos/Larissa/Sofia |

## 8. Decisões e trade-offs principais

As decisões arquiteturais fechadas estão registradas como ADRs; o resumo dos trade-offs de produto:

- **Outbox no MySQL vs. Redis/fila externa**: optou-se pelo MySQL existente para não subir infra nova; trade-off é a latência mínima de ~2s (polling) em vez de push reativo ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).
- **5 tentativas em ~15h vs. retry agressivo/indeterminado**: cobre manutenções reais (2h) sem deixar evento pendurado para sempre; trade-off é que indisponibilidade > ~15h exige replay manual ([ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)).
- **At-least-once vs. exactly-once**: simplicidade e padrão de mercado (Stripe/GitHub); trade-off é jogar a deduplicação para o cliente ([ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)).
- **Secret por endpoint vs. global**: limita o blast radius de vazamento; trade-off é a complexidade de rotação com grace period ([ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).

## 9. Dependências

- **Código existente**: `src/modules/orders/order.service.ts` (transação `changeStatus` a ser estendida), `src/shared/errors/*` (`AppError` e subclasses), `src/middlewares/auth.middleware.ts` (`requireRole`), `src/middlewares/error.middleware.ts` (tratamento centralizado), `src/shared/logger/index.ts` (Pino), `src/config/database.ts` (PrismaClient por processo), `prisma/schema.prisma` (novas tabelas) — ver [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) e [`FDD.md`](FDD.md).
- **Banco**: MySQL via Prisma (novas tabelas `webhook_outbox`, `webhook_dead_letter`, configuração de webhook e histórico de entregas).
- **Novo processo**: `src/worker.ts` + script `npm run worker` ([09:11] Larissa).
- **Portal de desenvolvedor** (fora do repo): documentação da deduplicação por `X-Event-Id` e da integração via API ([09:26] Marcos, [09:40] Marcos).
- **Revisão de segurança**: ~2 dias úteis reservados para Sofia revisar HMAC e geração de secret antes do deploy ([09:46] Sofia).

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
| --- | --- | --- | --- | --- | --- |
| **R-01** | Vazamento de secret no lado do cliente (já ocorreu antes) | Média | Alto | Secret única por endpoint + rotação com grace period de 24h + redaction de segredos no Pino | [09:21]–[09:22] Sofia/Diego |
| **R-02** | Cliente indisponível por mais de ~15h perde eventos na DLQ | Média | Médio | DLQ persistida com evidência + endpoint admin de replay manual | [09:16]–[09:18] Diego/Marcos |
| **R-03** | Crescimento da tabela `webhook_outbox` degrada o worker | Baixa | Médio | Índice em status/`created_at` + leitura em batch pequeno; arquivamento (adiado) | [09:07]–[09:08] Bruno/Diego |
| **R-04** | Perda de ordering ao escalar para múltiplos workers | Baixa | Médio | Single-worker nesta fase; particionamento/lock documentados como evolução futura | [09:12]–[09:13] Diego/Larissa |
| **R-05** | Cliente não implementa deduplicação e processa evento duplicado | Média | Médio | `X-Event-Id` + documentação destacada no portal de desenvolvedor | [09:25]–[09:26] Sofia/Marcos |
| **R-06** | Prazo apertado (fim de novembro) comprometer a revisão de segurança | Média | Alto | Estimativa de 3 sprints já inclui os ~2 dias de revisão da Sofia | [09:46]–[09:47] Sofia/Larissa |

## 11. Critérios de aceitação

- **CA-01**: ao mudar o status de um pedido com webhook cadastrado e filtro compatível, um evento é publicado na outbox na mesma transação; se a transação sofre rollback, nenhum evento é publicado ([09:40]–[09:41] Bruno/Diego).
- **CA-02**: o worker entrega o evento ao endpoint em < 10s (respeitado o polling de 2s), com payload JSON e os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` ([09:09]–[09:10], [09:44]–[09:45]).
- **CA-03**: uma entrega que falha é retentada com backoff 1m/5m/30m/2h/12h (5 retentativas, quase 15h entre a 1ª falha e a última tentativa); se a 5ª retentativa também falhar, o evento vai para a DLQ com payload, motivo e timestamp ([09:17]–[09:18] Diego).
- **CA-04**: cadastro com URL `http://` é recusado com erro de validação ([09:23] Sofia).
- **CA-05**: a assinatura `X-Signature` é um HMAC-SHA256 verificável do corpo com a secret do endpoint; após uma rotação, a secret antiga continua válida por 24h em paralelo e depois deixa de valer ([09:20]–[09:22] Sofia). O mecanismo de assinatura durante a janela está em aberto ([RFC Q-06](RFC.md#questões-em-aberto)).
- **CA-06**: `POST /admin/webhooks/dead-letter/:id/replay` exige role ADMIN, recoloca o evento como pendente e loga o autor ([09:36] Sofia/Larissa).
- **CA-07**: `GET /webhooks/:id/deliveries` retorna as últimas entregas com status, payload, response e tempo de resposta ([09:34] Marcos).
- **CA-08**: payload acima de 64KB falha (não trunca) ([09:23]–[09:24]).
- **CA-09**: eventos duplicados carregam o mesmo `X-Event-Id`, permitindo dedup pelo cliente ([09:25] Diego).

## 12. Estratégia de testes e validação

- **Testes unitários** da máquina de estados/filtro de eventos e da função `publishWebhookEvent(tx, ...)` (no padrão dos testes existentes em `tests/`).
- **Testes de integração** do fluxo completo `changeStatus` → outbox → worker → entrega, cobrindo atomicidade (rollback não publica) e retry/DLQ.
- **Testes de contrato** da assinatura HMAC-SHA256 (verificação com a secret do endpoint) e dos headers.
- **Testes de validação Zod**: URL `http://` recusada, payload > 64KB rejeitado.
- **Testes de autorização**: replay de DLQ bloqueado para role não-ADMIN.
- **Revisão de segurança** manual de Sofia (HMAC e geração de secret) antes do deploy, ~2 dias úteis ([09:46] Sofia).
- **Validação de latência**: confirmar entrega < 10s em cenário de polling de 2s ([09:09]–[09:10]).

> Observação: a entrega deste desafio é **puramente documental** — nenhum código da aplicação (`src/`, `prisma/`, `tests/`) foi alterado. A estratégia acima descreve o plano de validação para a fase de implementação.
