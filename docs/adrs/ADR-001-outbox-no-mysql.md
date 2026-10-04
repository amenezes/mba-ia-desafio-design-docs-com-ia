# ADR-001 — Padrão Outbox no MySQL para eventos de mudança de status

## Status

Aceito — decidido em reunião técnica ([09:08] Larissa: "Tá decidido então: outbox em MySQL"; confirmado no resumo [09:48]).

## Contexto

A feature de Webhooks de Notificação de Pedidos precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) quando o status de um pedido muda, com latência abaixo de 10 segundos ([09:00] e [09:02] Marcos). Hoje a aplicação não tem nenhum mecanismo de notificação externa, eventos ou filas.

A mudança de status já ocorre dentro de uma transação SQL pesada em `src/modules/orders/order.service.ts` (método `changeStatus`): atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity` dos produtos ([09:04] Bruno). Disparar chamadas HTTP dentro dessa transação acoplaria a disponibilidade do cliente externo ao fluxo crítico de pedidos: um cliente lento travaria mudanças de status de outros pedidos, e um cliente fora do ar forçaria a decisão impossível entre rollback do status ou perda da notificação ([09:04] Bruno).

O time é pequeno e a infra atual é apenas a aplicação Node.js + MySQL ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: quando o status do pedido muda, dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`, o sistema insere uma linha numa tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego).

Propriedades da decisão:

- **Atomicidade**: se a transação principal commitou, o evento foi registrado; se deu rollback, o evento some junto — não há inconsistência possível ([09:06] Diego). Se a inserção na outbox falhar, a mudança de status também falha (rollback); "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno, [09:41] Diego).
- **Acesso eficiente**: índice nos campos de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`; o worker lê apenas pendentes em batch pequeno ([09:08] Diego).
- **Snapshot na inserção**: o evento é gravado com o payload já renderizado no momento da mudança de status — se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou ([09:52] Larissa, [09:52] Diego).
- **IDs em UUID**, seguindo o padrão do restante do projeto ([09:51] Larissa).
- **Arquivamento** de linhas entregues após ~30 dias fica **fora do escopo** desta feature ([09:08] Diego).
- A inserção na outbox respeita o filtro de eventos: se nenhum webhook do customer quer aquele status, nem insere ([09:34] Bruno, [09:34] Diego).

## Alternativas Consideradas

1. **Disparo síncrono no service de orders** (chamada HTTP dentro da transação de mudança de status). Descartada: transação já é pesada; cliente lento trava o fluxo crítico e cliente offline cria dilema de rollback ([09:03] Larissa levantou, [09:04] Bruno argumentou contra, [09:06] Diego: "Síncrono está fora de questão").
2. **Redis Streams ou fila externa semelhante**. Descartada: exigiria subir mais infra; para um time pequeno, "subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Larissa e Diego).
3. **Publicar fora da transação** (inserir o evento após o commit). Descartada: "se ficar fora da transação, perde a garantia toda" ([09:41] Diego) — reabre a inconsistência que o outbox atômico elimina.

## Consequências

**Positivas**

- Zero infra nova: usa o MySQL e o Prisma já existentes ([09:07] Diego).
- Garantia forte de consistência entre mudança de status e evento registrado ([09:06] Diego).
- Desacoplamento total entre o fluxo de pedidos e a disponibilidade do cliente externo ([09:04] Bruno).

**Negativas / trade-offs**

- Latência mínima de entrega: o evento só sai no próximo ciclo de polling do worker (2s no pior caso — ver ADR-002), em vez de envio imediato ([09:10] Larissa: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos").
- A transação de `changeStatus` fica ligeiramente maior (uma inserção extra) ([09:40] Bruno).
- A tabela `webhook_outbox` cresce com o tempo e exigirá estratégia de arquivamento — explicitamente adiada ([09:08] Diego).
- `OrderService` passa a depender de uma função de publicação de eventos (`publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o client da transação) ([09:41] Bruno, [09:41] Diego).

## Referências

- Transcrição: [09:03]–[09:08], [09:40]–[09:41], [09:48], [09:51]–[09:52].
- Código: `src/modules/orders/order.service.ts` (`changeStatus`, `$transaction`), `prisma/schema.prisma` (modelos `Order`, `OrderStatusHistory`).
- ADRs relacionados: [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md).
