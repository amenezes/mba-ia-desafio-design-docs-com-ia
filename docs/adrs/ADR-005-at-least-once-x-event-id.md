# ADR-005 — Garantia at-least-once com deduplicação por X-Event-Id

## Status

Aceito — decidido em reunião técnica ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."; confirmado no resumo [09:48]).

## Contexto

Com outbox + retry com backoff ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md)), pode acontecer de o cliente receber o mesmo evento duas vezes — por exemplo, quando o cliente processa a entrega mas a resposta não chega ao worker dentro do timeout de 10s ([09:42] Diego), que então retenta. Era preciso escolher o nível de garantia de entrega ([09:24] Diego: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado").

## Decisão

1. **Garantia at-least-once**: todo evento pendente é tentado até ser entregue com sucesso ou esgotar as 5 tentativas e ir para a DLQ ([09:24] Diego, [09:15] Diego).
2. **`X-Event-Id` no header**: um **UUID gerado quando o evento entra na outbox**, único por evento, é enviado em cada entrega ([09:25] Diego).
3. **Deduplicação do lado do cliente**: se o cliente receber o mesmo `event_id` duas vezes, ele deduplica ([09:25] Diego). Essa responsabilidade será **documentada bem destacada no portal de desenvolvedor** ([09:26] Marcos).
4. O `event_id` também compõe o payload JSON do evento (`event_id`, ao lado de `event_type`, `timestamp` etc.) ([09:43] Diego).

## Alternativas Consideradas

1. **Garantia exactly-once**. Descartada: "exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" ([09:25] Diego). Além disso, é o padrão de mercado — "Stripe faz assim, GitHub faz assim" ([09:25] Diego).
2. **Deduplicação do lado da plataforma** (não reenviar evento já entregue). Descartada implicitamente pela decisão: a plataforma não tem como saber se o cliente processou uma entrega cuja resposta se perdeu — daí a responsabilidade de dedup ser do cliente, via `X-Event-Id` ([09:25] Diego, [09:25] Sofia: "Isso joga responsabilidade pro cliente").

## Consequências

**Positivas**

- Nenhuma mudança de status é perdida sem rastro: ou entrega, ou DLQ com evidência ([09:15] Diego, [09:18] Diego).
- Simplicidade: sem coordenação distribuída entre plataforma e cliente ([09:25] Diego).
- Alinhado ao padrão de mercado (Stripe, GitHub), o que reduz a surpresa na integração ([09:25] Diego).

**Negativas / trade-offs**

- Clientes **precisam** implementar deduplicação por `event_id`; quem não implementar pode processar o mesmo evento duas vezes ([09:25] Sofia: "joga responsabilidade pro cliente"). Mitigação: documentação destacada no portal ([09:26] Marcos).
- Entregas duplicadas aparecem no histórico de entregas e podem gerar confusão em auditorias se o `X-Event-Id` não for considerado ([09:34] Marcos).

## Referências

- Transcrição: [09:24]–[09:26], [09:43], [09:48].
- Código: `prisma/schema.prisma` (padrão de ids `@default(uuid()) @db.Char(36)` que o `event_id` seguirá, [09:51] Larissa).
- ADRs relacionados: [ADR-003](ADR-003-retry-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md).
