# ADR-003 — Retry com backoff exponencial e Dead Letter Queue em tabela separada

## Status

Aceito — decidido em reunião técnica ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"; [09:18] Diego: DLQ em tabela separada; confirmados no resumo [09:48]).

## Contexto

Clientes B2B podem estar temporariamente indisponíveis quando o webhook é disparado. É preciso definir o que acontece quando a entrega falha: quantas tentativas, com que intervalo, e para onde vai um evento que falhou definitivamente ([09:14] Larissa: "Se o cliente tá offline, o que a gente faz?").

Há precedente real: um cliente já teve indisponibilidade de **duas horas** em manutenção planejada ([09:16] Diego). Por outro lado, retry indefinido deixa evento pendurado para sempre se o cliente sumiu ([09:15] Diego).

## Decisão

1. **Backoff exponencial com 5 tentativas** ([09:15] Diego, [09:16] Diego: "Cinco já dá pra cobrir uma janela de até 12 ou 24 horas").
2. **Progressão fixa**: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas — total de quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego). Aceitável do ponto de vista de produto: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).
3. **Falha permanente → Dead Letter Queue**: esgotadas as tentativas, o evento é considerado falha permanente e movido para a DLQ ([09:15] Diego).
4. **DLQ em tabela separada** (`webhook_dead_letter`), contendo o payload, o motivo da falha e o timestamp — mantém a leitura da outbox principal limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).
5. **Reprocessamento manual via endpoint admin**: `POST /admin/webhooks/dead-letter/:id/replay` recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego). O endpoint exige role `ADMIN` do JWT e reaproveita o `requireRole` já existente ([09:36] Larissa), e **loga quem fez o replay** para auditoria ([09:36] Sofia).
6. **Timeout de entrega de 10 segundos**: cliente que não responde em 10s é tratado como falha e marcado para retry ([09:42] Diego).

## Alternativas Consideradas

1. **Retry indefinido com backoff**. Descartada: evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego).
2. **3 tentativas (mais agressivo)**. Descartada: "3 é pouco" — com indisponibilidade de manhã, o sistema retentaria três vezes em 30 minutos e mataria o evento; já houve cliente com duas horas de manutenção planejada ([09:16] Bruno levantou, Diego descartou).
3. **Marcar como `failed` na própria outbox, sem tabela separada**. Descartada: a tabela separada deixa a leitura da outbox principal mais limpa e funciona como evidência para debug e reprocessamento ([09:17] Larissa levantou, [09:18] Diego justificou).
4. **Reprocessamento automático da DLQ**. Descartada: o replay é manual, via endpoint admin com auditoria ([09:18] Diego, [09:36] Sofia) — reprocessar automaticamente um evento que falhou 5 vezes ao longo de ~15h sem avaliação humana não foi proposto nem aceito.

## Consequências

**Positivas**

- Cobre indisponibilidades reais de até ~15 horas, incluindo manutenções planejadas de 2h ([09:16]–[09:17] Diego).
- Outbox principal permanece limpa e indexada; DLQ vira fonte de auditoria e debug ([09:18] Diego).
- Replay manual com controle de acesso ADMIN e trilha de auditoria ([09:36] Larissa e Sofia).

**Negativas / trade-offs**

- Eventos de clientes com indisponibilidade superior a ~15h só são recuperados por intervenção manual ([09:17]–[09:18]).
- O reprocessamento exige operação humana — não há replay automático ([09:18] Diego).
- Mais uma tabela para manter e consultar (`webhook_dead_letter`) ([09:18] Diego).

## Referências

- Transcrição: [09:14]–[09:18], [09:35]–[09:36], [09:42], [09:48].
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`), `prisma/schema.prisma` (`UserRole.ADMIN`), `src/shared/logger/index.ts` (Pino para auditoria).
- ADRs relacionados: [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md).
