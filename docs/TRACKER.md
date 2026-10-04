# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Referência cruzada entre cada item registrado nos documentos do pacote e sua **origem na transcrição** ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) **ou no código** do repositório base. Nenhum requisito, decisão ou restrição foi inventado: itens sem origem identificável foram ajustados ou removidos durante a produção.

**Formato**: `Fonte = TRANSCRICAO` ⇒ `Localização = [hh:mm] Nome`; `Fonte = CODIGO` ⇒ `Localização = caminho do arquivo`.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação em tempo real de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara; Atlas pode migrar para concorrente até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 segundos já é considerada "tempo real" pelos clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-01 | docs/PRD.md | Escopo | Webhooks são outbound apenas (cliente recebe, não envia) | TRANSCRICAO | [09:02] Sofia; [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Meta quantitativa: entrega < 10s; pior caso de polling 2s aceito | TRANSCRICAO | [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Prazo | Prazo fim de novembro; estimativa de 3 sprints com revisão de segurança inclusa | TRANSCRICAO | [09:45] Marcos; [09:46] Larissa |
| PRD-FESC-01 | docs/PRD.md | Fora de Escopo | E-mail de alerta para webhook com problema adiado para próxima fase | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-FESC-02 | docs/PRD.md | Fora de Escopo | Dashboard visual descartado; painel é projeto separado do time de frontend | TRANSCRICAO | [09:39] Marcos; [09:40] Larissa |
| PRD-FESC-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída adiado: "observa e implementa se virar problema" | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| PRD-FESC-04 | docs/PRD.md | Fora de Escopo | Ordering global/múltiplos workers adiados; limitação conhecida documentada | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| PRD-FESC-05 | docs/PRD.md | Fora de Escopo | Arquivamento de linhas entregues após ~30 dias fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Filtro de eventos: lista dos status que o webhook quer ouvir | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | customer_id passado no body/path; não vem do JWT (JWT é do usuário operador) | TRANSCRICAO | [09:32] Bruno; [09:32] Larissa |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | PATCH para editar, DELETE para remover, GET para listar webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro aplicado na inserção na outbox; sem webhooks interessados, nem insere | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Evento publicado na outbox dentro da mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Entrega HTTP com payload JSON assinado e headers padronizados | TRANSCRICAO | [09:43] Diego; [09:44] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry com backoff exponencial (5 tentativas) e DLQ após falha permanente | TRANSCRICAO | [09:15] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico de entregas: últimas ~100, sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via endpoint admin; log de quem fez para auditoria | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret via API; antiga válida por 24h em paralelo | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Garantia at-least-once; cliente deduplica por X-Event-Id (UUID único por evento) | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório: URL https; http recusado com erro de validação no schema Zod | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 sobre o corpo do request; assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB; ultrapassou ⇒ erro (não trunca) | TRANSCRICAO | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s na chamada HTTP do worker; estouro ⇒ falha e retry | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Atomicidade: commitou ⇒ evento registrado; rollback ⇒ evento some junto | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint (não global); segredos não vazam em log | TRANSCRICAO | [09:21] Sofia; [09:22] Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Replay de DLQ exige role ADMIN reaproveitando requireRole existente | TRANSCRICAO | [09:36] Larissa |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Outbox com índice em status e created_at; worker lê pendentes em batch pequeno | TRANSCRICAO | [09:08] Diego |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Prazo de entrega fim de novembro; ~3 sprints incluindo a revisão de segurança | TRANSCRICAO | [09:45] Marcos; [09:47] Larissa |
| PRD-R-01 | docs/PRD.md | Risco | Vazamento de secret no lado do cliente (precedente real em log de aplicação) | TRANSCRICAO | [09:22] Diego |
| PRD-R-02 | docs/PRD.md | Risco | Indisponibilidade longa do cliente; precedente de 2h em manutenção planejada; janela de retry ~15h | TRANSCRICAO | [09:16] Diego; [09:17] Marcos |
| PRD-R-03 | docs/PRD.md | Risco | Acúmulo de eventos na tabela pode deixar o worker lento | TRANSCRICAO | [09:07] Bruno |
| PRD-R-04 | docs/PRD.md | Risco | Perda de ordering ao escalar para múltiplos workers | TRANSCRICAO | [09:12] Diego |
| PRD-R-05 | docs/PRD.md | Risco | Dedup é responsabilidade do cliente; exige documentação destacada no portal | TRANSCRICAO | [09:25] Sofia; [09:26] Marcos |
| PRD-R-06 | docs/PRD.md | Risco | Prazo (3 sprints) deve absorver a revisão de segurança de 2 dias | TRANSCRICAO | [09:46] Sofia; [09:47] Larissa |
| PRD-DEC-01 | docs/PRD.md | Decisão | Trade-off aceito: latência mínima de 2s (polling) em vez de push reativo | TRANSCRICAO | [09:10] Larissa |
| PRD-CA-09 | docs/PRD.md | Critério de Aceite | Duplicatas carregam o mesmo X-Event-Id para dedup | TRANSCRICAO | [09:25] Diego |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Confiabilidade: retry cobre indisponibilidade do cliente de até ~15h antes da DLQ | TRANSCRICAO | [09:17] Diego; [09:17] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | 100% das entregas assinadas com HMAC-SHA256 (X-Signature) | TRANSCRICAO | [09:20] Sofia |
| PRD-FESC-06 | docs/PRD.md | Fora de Escopo | Webhooks inbound (cliente enviando para a plataforma) descartados | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário de Uso | Cliente cadastra endpoint via API autenticada por JWT de usuário que representa o cliente | TRANSCRICAO | [09:32] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário de Uso | Cliente filtra para receber só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Portal de desenvolvedor documenta dedup por X-Event-Id e integração via API | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança de ≥2 dias úteis (HMAC e geração de secret) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Nova entry-point src/worker.ts (a criar) + script npm run worker | TRANSCRICAO | [09:11] Larissa |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Evento publicado na outbox na mesma transação; rollback não publica | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | Entrega em < 10s com payload JSON e headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id | TRANSCRICAO | [09:10] Larissa; [09:44] Diego; [09:45] Diego |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | Retry 1m/5m/30m/2h/12h (5 retentativas, ~15h); falha da 5ª retentativa ⇒ DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Cadastro com URL http:// recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | X-Signature HMAC-SHA256 verificável; secret antiga válida por 24h após rotação | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Replay de DLQ exige ADMIN, recoloca como pendente e loga o autor | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Histórico de entregas com status, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Payload acima de 64KB falha, sem truncamento | TRANSCRICAO | [09:24] Larissa |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Validação de latência < 10s com polling de 2s | TRANSCRICAO | [09:09] Diego; [09:10] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono no service: transação pesada; cliente lento trava o fluxo; dilema de rollback | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams/fila externa: subir Redis Cluster para time pequeno é overengineering | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco: MySQL não tem NOTIFY/LISTEN; trigger não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Exactly-once: exigiria coordenação dos dois lados; muito mais complexo | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | 3 tentativas: pouco; cliente com 2h de manutenção ficaria descoberto | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | DLQ como flag "failed" na outbox: tabela separada é mais limpa e vira evidência | TRANSCRICAO | [09:18] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Diego; [09:39] Larissa |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | Autorização do CRUD de webhooks pode ser endurecida no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Escala futura do worker: particionamento por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Arquivamento da outbox após ~30 dias ficou para depois | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | Novo processo (worker) para deploy/monitorar; polling constante no MySQL | TRANSCRICAO | [09:11] Diego; [09:09] Diego |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | E-mail de alerta de webhook com problema adiado até medir o impacto | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| RFC-Q-06 | docs/RFC.md | Questão em Aberto | Mecanismo de assinatura outbound durante o grace period de 24h não foi definido (decidida só a janela) | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| RFC-IMP-02 | docs/RFC.md | Limitação Conhecida | Sem ordering global; só por order_id enquanto single-worker; clientes nunca pediram ordering global | TRANSCRICAO | [09:13] Larissa; [09:14] Marcos |
| RFC-PROP-01 | docs/RFC.md | Decisão | Interpretação: "5 tentativas" = 5 retentativas, pois os 5 intervalos somam ~15h entre 1ª falha e última tentativa | TRANSCRICAO | [09:17] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL: evento inserido na mesma transação SQL de orders/history | TRANSCRICAO | [09:06] Diego; [09:08] Larissa |
| ADR-001-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Payload renderizado em snapshot no momento da inserção | TRANSCRICAO | [09:52] Larissa; [09:52] Diego |
| ADR-001-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Falha na inserção do outbox ⇒ rollback da mudança de status | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling de 2s buscando pendentes mais antigos | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| ADR-002-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado; entry-point src/worker.ts + script npm run worker | TRANSCRICAO | [09:11] Diego; [09:11] Larissa |
| ADR-002-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single worker nesta fase; ordering implícita por order_id via created_at | TRANSCRICAO | [09:12] Diego |
| ADR-002-04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Clientes nunca pediram ordering global | TRANSCRICAO | [09:14] Marcos |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h (~15h de janela) | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| ADR-003-02 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-03 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual: POST /admin/webhooks/dead-letter/:id/replay recoloca como pendente | TRANSCRICAO | [09:18] Diego; [09:35] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo; assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Tabela de configuração armazena url + secret + customer_id + ativo | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| ADR-004-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | X-Timestamp permite ao cliente detectar replay attack | TRANSCRICAO | [09:44] Diego |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na entrada do outbox) | TRANSCRICAO | [09:25] Diego; [09:26] Larissa |
| ADR-005-02 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Trade-off | Padrão de mercado: Stripe e GitHub fazem assim | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Módulo src/modules/webhooks seguindo o padrão controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Códigos de erro com prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:28] Bruno; [09:29] Larissa |
| ADR-006-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Reuso de Pino, error middleware centralizado e AppError sem mudança | TRANSCRICAO | [09:29] Bruno; [09:30] Larissa |
| ADR-006-04 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | Worker usa instância própria de PrismaClient (client é por processo), mesmo DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-006-05 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Decisão | IDs UUID: "segue o padrão do resto do projeto. Tudo é uuid" | TRANSCRICAO | [09:51] Larissa; [09:51] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono dentro da transação de mudança de status | TRANSCRICAO | [09:03] Larissa; [09:04] Bruno; [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams/fila externa: infra nova, overengineering para time pequeno | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Publicar o evento fora da transação: "perde a garantia toda" | TRANSCRICAO | [09:41] Diego |
| ADR-001-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Latência mínima de 2s no pior caso aceita em troca de não subir infra | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger de banco para reatividade (MySQL sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Bruno; [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker dentro do processo da API: reinício da API derruba o worker | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Adiada | Múltiplos workers em paralelo: perde ordering; particionamento/lock no futuro | TRANSCRICAO | [09:12] Diego; [09:13] Diego |
| ADR-002-05 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Lógica do worker em src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts (a criar) | TRANSCRICAO | [09:28] Bruno |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Alternativa Descartada | Retry indefinido: evento fica pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Alternativa Descartada | 3 tentativas: pouco para indisponibilidades de 2h | TRANSCRICAO | [09:16] Bruno; [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Alternativa Descartada | Flag "failed" na própria outbox em vez de tabela separada | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| ADR-003-ALT-04 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Alternativa Descartada | Reprocessamento automático da DLQ (plausível; decidido replay manual) | TRANSCRICAO | [09:18] Diego |
| ADR-003-04 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay exige role ADMIN e loga quem fez para auditoria | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| ADR-003-05 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Timeout de 10s tratado como falha e marcado para retry | TRANSCRICAO | [09:42] Diego |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma: "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Truncar payload acima do limite em vez de errar | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Enviar sem assinatura confiando só em TLS (cliente não valida autoria/integridade) | TRANSCRICAO | [09:19] Sofia |
| ADR-004-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| ADR-004-05 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | TLS obrigatório (https) como validação no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Alternativa Descartada | Exactly-once: exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Alternativa Descartada | Dedup no lado da plataforma; responsabilidade fica com o cliente | TRANSCRICAO | [09:25] Sofia; [09:25] Diego |
| ADR-005-03 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | event_id também compõe o payload JSON | TRANSCRICAO | [09:43] Diego |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Alternativa Descartada | Módulo de webhooks com estrutura própria: "fica como módulo igual aos outros" | TRANSCRICAO | [09:27] Bruno; [09:30] Larissa |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Alternativa Descartada | Logging/erro específico de webhook: "não vamos botar nada novo" | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-padroes-existentes.md | Alternativa Descartada | Códigos de erro sem prefixo dedicado | TRANSCRICAO | [09:29] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | Headers de envio: X-Event-Id, X-Signature, X-Timestamp, Content-Type application/json | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | Header X-Webhook-Id com o id do endpoint webhook | TRANSCRICAO | [09:44] Sofia; [09:45] Diego |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | Payload: event_id, event_type "order.status_changed", timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | Função publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o client da transação | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | CRUD de configuração pode ser usado por qualquer role autenticada (por enquanto) | TRANSCRICAO | [09:36] Marcos; [09:37] Sofia |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Status da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-ERRO-01 | docs/FDD.md | Erro | Códigos citados nominalmente: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Histórico guarda tempo de resposta por entrega (responseTimeMs) | TRANSCRICAO | [09:34] Marcos |
| FDD-SEC-01 | docs/FDD.md | Segurança | Replay de DLQ loga o autor para auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-RES-01 | docs/FDD.md | Resiliência | Retry cobre janela de até 12–24h de indisponibilidade do cliente | TRANSCRICAO | [09:15] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | POST /api/v1/webhooks: url https + eventStatuses; secret gerada e devolvida só na criação (201) | TRANSCRICAO | [09:31] Marcos; [09:23] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId=: lista webhooks do customer (customer_id na query, não do JWT) | TRANSCRICAO | [09:32] Larissa; [09:33] Bruno |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id: edita url, eventStatuses, active | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id: remove (204) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret: nova secret; antiga válida 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries: últimas 100 entregas paginadas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay: ADMIN, 202, evento volta a PENDING | TRANSCRICAO | [09:35] Diego; [09:36] Larissa |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Outbound: cliente responde 2xx em até 10s e deduplica por X-Event-Id | TRANSCRICAO | [09:42] Diego; [09:25] Diego |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | Proposta (a validar): duas assinaturas em X-Signature durante a janela de rotação — derivação da RFC Q-06 | TRANSCRICAO | [09:21] Sofia; [09:46] Sofia |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE: payload > 64KB, erro sem truncar | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT: cliente não respondeu em 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED: não-2xx/erro de rede alimenta retry e DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENT_FILTER (derivado): eventStatuses vazio/inválido; regra do prefixo | TRANSCRICAO | [09:33] Marcos; [09:29] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND / _ALREADY_REPLAYED (derivados do endpoint de replay) | TRANSCRICAO | [09:18] Diego; [09:29] Larissa |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker a cada 2s lê PENDING mais antigos em batch, ordenados por created_at | TRANSCRICAO | [09:09] Diego; [09:12] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry: 5 retentativas após o envio original (1m/5m/30m/2h/12h ≈ 14h36min); 6ª falha ⇒ DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ grava payload, motivo e timestamp; evento FAILED na outbox | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Publicação: filtra endpoints do customer pelo status; sem interessados não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de Dados | IDs UUID @db.Char(36) nas novas tabelas | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-02 | docs/FDD.md | Modelo de Dados | Outbox guarda payload renderizado (snapshot na inserção) | TRANSCRICAO | [09:52] Larissa; [09:52] Bruno |
| FDD-DADOS-03 | docs/FDD.md | Modelo de Dados | Tabela de configuração: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-RES-02 | docs/FDD.md | Resiliência | Timeout de 10s por chamada HTTP | TRANSCRICAO | [09:42] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Restart: PROCESSING órfãos voltam a PENDING (derivação de estado processando + at-least-once) | TRANSCRICAO | [09:08] Diego; [09:24] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Rate limiting de saída não implementado nesta fase; observar | TRANSCRICAO | [09:39] Larissa |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Pino como único logger, sem stack nova; redaction de secrets | TRANSCRICAO | [09:29] Bruno; [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | X-Event-Id como chave de correlação ponta a ponta (tracing) | TRANSCRICAO | [09:25] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Deploy com 2 processos (API + worker) no mesmo banco, PrismaClient por processo | TRANSCRICAO | [09:11] Diego; [09:30] Bruno |
| FDD-T-01 | docs/FDD.md | Critério de Aceite Técnico | Rollback do changeStatus ⇒ zero linhas na outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-T-02 | docs/FDD.md | Critério de Aceite Técnico | Status fora do filtro ⇒ nenhuma linha inserida | TRANSCRICAO | [09:34] Bruno |
| FDD-T-03 | docs/FDD.md | Critério de Aceite Técnico | Entrega < 10s (polling 2s + HTTP) | TRANSCRICAO | [09:02] Marcos; [09:09] Diego |
| FDD-T-04 | docs/FDD.md | Critério de Aceite Técnico | X-Signature verificável; secret anterior verificável por 24h após rotação | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia |
| FDD-T-05 | docs/FDD.md | Critério de Aceite Técnico | Backoff 1m/5m/30m/2h/12h; falha da 5ª retentativa ⇒ DLQ | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-T-06 | docs/FDD.md | Critério de Aceite Técnico | Replay por não-ADMIN ⇒ 403; ADMIN ⇒ PENDING + log do autor | TRANSCRICAO | [09:36] Sofia |
| FDD-T-07 | docs/FDD.md | Critério de Aceite Técnico | Cadastro http:// ⇒ 400 WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-T-08 | docs/FDD.md | Critério de Aceite Técnico | Payload > 64KB ⇒ WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:24] Larissa |
| FDD-T-09 | docs/FDD.md | Critério de Aceite Técnico | Deliveries retorna payload, status, body e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-T-10 | docs/FDD.md | Critério de Aceite Técnico | Reinício do worker não perde eventos; duplicatas dedupáveis | TRANSCRICAO | [09:11] Diego; [09:25] Diego |
| FDD-T-11 | docs/FDD.md | Critério de Aceite Técnico | Segredos nunca aparecem em logs | TRANSCRICAO | [09:22] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus executa $transaction com update do order, insert no history e débito/reposição de estoque — ponto de extensão do outbox | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados canTransition e enum OrderStatus — fonte dos status do filtro e do payload | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Convenções de schema: id uuid @db.Char(36), @@map snake_case, @@index; models Order/OrderStatusHistory | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração | AppError com statusCode/errorCode/details; subclasses InsufficientStockError, InvalidStatusTransitionError — padrão para erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts; src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN'/'OPERATOR') reaproveitados nas rotas de webhooks | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | errorMiddleware trata AppError, ZodError e Prisma sem necessidade de mudança | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Logger Pino com redact de segredos (*.password, *.token) — estendido para secrets de webhook | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Rotas montadas sob /api/v1 via buildApiRouter — mesmo padrão para /webhooks e /admin/webhooks | CODIGO | src/app.ts; src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Entry-point existente que inspira src/worker.ts; createPrismaClient para instância por processo | CODIGO | src/server.ts; src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Stack e versões (Node ≥20, express 4.21, @prisma/client 5.22, zod 3.23, pino 9.5, vitest); scripts npm | CODIGO | package.json |
| FDD-INT-11 | docs/FDD.md | Integração | Helper paginated()/PaginatedResponse usado nas listagens de webhooks e deliveries | CODIGO | src/shared/http/response.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Padrão de testes vitest + supertest + factories a ser seguido pelos testes de webhooks | CODIGO | tests/orders.test.ts; tests/helpers/factories.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Schema Zod de env (src/config/env.ts) recebe as novas variáveis do worker/webhook | CODIGO | src/config/env.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Middleware validate() com schemas Zod (body/query/params) — onde entra a exigência de https | CODIGO | src/middlewares/validate.middleware.ts |

## Resumo de cobertura

- **Total de linhas**: 179 — **TRANSCRICAO**: 165 (92%) · **CODIGO**: 14 (8%).
- Cobertura por documento: PRD (RFs, RNFs, objetivos, fora de escopo, riscos, dependências, critérios de aceitação), RFC (alternativas, questões em aberto, impacto), ADRs (decisões e alternativas consideradas de cada ADR) e FDD (contratos de cada endpoint, matriz de erros, fluxos, modelo de dados, resiliência, observabilidade, critérios técnicos T-01 a T-11 e integração com o código).
- Todas as linhas TRANSCRICAO usam timestamp válido no formato `[hh:mm] Nome`.
- Todas as linhas CODIGO referenciam caminhos reais existentes no repositório base.
- Itens derivados sem citação nominal na reunião (ex.: códigos `WEBHOOK_INVALID_EVENT_FILTER`, `WEBHOOK_DEAD_LETTER_*`) e propostas ainda não decididas (assinatura durante a rotação de secret — RFC Q-06; retorno de eventos `PROCESSING` órfãos a `PENDING`) estão explicitamente sinalizados como derivação no [`FDD.md` §7](FDD.md#7-matriz-de-erros-previstos-prefixo-webhook_) — a regra que os origina (`WEBHOOK_` pra tudo do módulo) está em [09:29] Larissa.
