# ADR-002 — Worker em processo separado com polling de 2 segundos

## Status

Aceito — decidido em reunião técnica ([09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s"; [09:11] Diego: processo separado; confirmados no resumo [09:48]).

## Contexto

Com o padrão Outbox escolhido ([ADR-001](ADR-001-outbox-no-mysql.md)), algo precisa ler a tabela `webhook_outbox` e disparar as chamadas HTTP. A questão é como o worker é notificado e onde ele roda ([09:08] Larissa: "como o worker lê isso?").

O requisito de latência do cliente é "qualquer coisa abaixo de 10 segundos já é 'tempo real'" ([09:02] Marcos). O banco é MySQL, que não tem listener nativo tipo o `NOTIFY/LISTEN` do Postgres; trigger de banco só executa SQL e não notifica processo externo ([09:09] Diego).

## Decisão

1. **Polling em loop**: a cada **2 segundos**, o worker busca os eventos pendentes mais antigos na outbox, processa e marca ([09:09] Diego). Atende o requisito de "abaixo de 10 segundos" com folga ([09:09] Diego, [09:10] Marcos: "2 segundos serve, perfeito"). A latência mínima será de 2 segundos no pior caso — trade-off aceito explicitamente ([09:10] Larissa).
2. **Processo separado da API**: o worker não roda dentro da mesma instância da API; se a API reinicia, o worker não é afetado ([09:11] Diego).
3. **Nova entry-point no projeto**: `src/worker.ts`, seguindo o que já existe em `src/server.ts`, com um script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
4. **Mesmo banco, mesma stack, PrismaClient próprio**: o worker conecta no mesmo MySQL com a mesma `DATABASE_URL`, mas com uma **instância nova** do `PrismaClient`, porque `PrismaClient` é por processo ([09:11] Bruno, [09:29]–[09:30] Diego e Bruno).
5. **Single worker** nesta fase: um único worker processa em ordem de `created_at`, o que dá ordering implícita por `order_id` ([09:12] Diego). Escalar para múltiplos workers (particionamento por `order_id` ou lock pessimista) é problema do futuro ([09:13] Diego) e fica registrado como **limitação conhecida**: não há garantia de ordering global, só por `order_id` e enquanto for single-worker ([09:13] Larissa). Os clientes nunca pediram ordering global ([09:14] Marcos).

## Alternativas Consideradas

1. **Trigger de banco para reatividade**. Descartada: MySQL não tem `NOTIFY/LISTEN`; trigger só executa SQL e não notifica processo externo — para avisar o worker seria preciso improvisar (escrever em arquivo, bater num endpoint), "fica esquisito" ([09:09] Bruno levantou, Diego descartou).
2. **Worker dentro do processo da API**. Descartada: se a API reinicia, perde o worker ([09:11] Diego).
3. **Múltiplos workers em paralelo desde já**. Descartada/adiada: perde a garantia de ordering ([09:12] Diego); particionamento por `order_id` ou lock pessimista ficam para o futuro ([09:13] Diego).

## Consequências

**Positivas**

- Simplicidade operacional: nenhuma tecnologia ou infra nova; mesmo Node, mesmo MySQL, mesmo Prisma ([09:11] Diego).
- Ciclo de vida independente da API ([09:11] Diego).
- Ordering por `order_id` gratuita enquanto single worker ([09:12] Diego).

**Negativas / trade-offs**

- Latência mínima de até 2s por evento (aceita explicitamente, [09:10] Larissa).
- Polling gera consultas periódicas ao banco mesmo sem eventos pendentes.
- Ordering global não garantida; escalar horizontalmente o worker exige resolver particionamento ou lock antes ([09:13] Larissa: "documentamos como limitação conhecida").

## Referências

- Transcrição: [09:08]–[09:14], [09:28], [09:29]–[09:30], [09:48].
- Código: `src/server.ts` (entry-point existente que inspira o `src/worker.ts`), `src/config/database.ts` (`createPrismaClient`), `package.json` (scripts).
- ADRs relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md).
