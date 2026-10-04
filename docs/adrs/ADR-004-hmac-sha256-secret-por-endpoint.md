# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação

## Status

Aceito — decidido em reunião técnica ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h"; confirmado no resumo [09:48]).

## Contexto

A feature expõe eventos com dados de pedidos para endpoints **fora da nossa infraestrutura** ([09:19] Sofia). O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que ninguém adulterou o payload no meio do caminho ([09:19] Sofia).

Há precedente de segurança real: um cliente já vazou secret em log da própria aplicação dele ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**: a plataforma assina o payload com uma secret compartilhada com o cliente e envia a assinatura no header `X-Signature`; o cliente verifica do lado dele ([09:20] Sofia). SHA-256 é o padrão de mercado e "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
2. **Secret única por endpoint de webhook** — não existe secret global da plataforma: "se vaza uma, vaza tudo" ([09:21] Sofia). A tabela de configuração de webhook armazena `url` + `secret` + `customer_id` + estado ativo ([09:21] Bruno, [09:21] Sofia).
3. **Rotação de secret com grace period de 24h**: endpoint na API permite ao cliente pedir nova secret; durante a rotação, a secret antiga permanece válida por **24 horas em paralelo** para o cliente migrar os sistemas dele; depois disso, a antiga morre ([09:21] Sofia). Como a verificação é feita pelo cliente, o *mecanismo* de assinatura durante essa janela não foi definido na reunião e fica como questão em aberto ([RFC Q-06](../RFC.md#questões-em-aberto)); a proposta está no [FDD §6.5](../FDD.md#65-post-apiv1webhooksidrotate-secret--rotação-de-secret-rf-11).
4. **TLS obrigatório**: a URL do webhook tem que ser `https`; cadastro com `http` é recusado com erro de validação — implementado como validação no schema Zod, não como decisão arquitetural separada ([09:23] Sofia).
5. **Limite de payload de 64KB, com erro (não truncamento)**: se o evento ultrapassar o teto, a entrega falha; "se chegou nesse tamanho, tem algo errado" ([09:23] Sofia, [09:24] Diego: "64KB já é um teto generoso", [09:24] Larissa: "64KB de limite, erro caso ultrapasse").
6. **Headers de envio associados**: `X-Signature` (HMAC), `X-Timestamp` (timestamp do envio, para o cliente detectar replay attack se quiser) e `X-Webhook-Id` (id do endpoint, para clientes com vários cadastros saberem qual originou o envio), além de `X-Event-Id` (ver [ADR-005](ADR-005-at-least-once-x-event-id.md)) e `Content-Type: application/json` ([09:44] Diego, [09:44]–[09:45] Sofia e Diego).

## Alternativas Consideradas

1. **Secret global da plataforma**. Descartada: um único vazamento compromete todos os clientes ([09:21] Sofia). O precedente de vazamento em log de cliente ([09:22] Diego) reforça a escolha.
2. **Truncar payload acima do limite** em vez de rejeitar. Descartada: payload de 500KB indica que algo está errado; a preferência explícita foi errar, não truncar ([09:23] Sofia).
3. **Enviar sem assinatura** (confiar apenas em TLS). Descartada: TLS protege o transporte, mas não permite ao cliente validar autoria e integridade do payload ([09:19] Sofia).

## Consequências

**Positivas**

- Cliente consegue verificar autoria e integridade de cada entrega com biblioteca padrão de mercado ([09:20] Sofia).
- Blast radius de vazamento limitado a um endpoint, com caminho de rotação em 24h ([09:21] Sofia, [09:22] Diego).
- `X-Timestamp` permite ao cliente proteger-se contra replay attacks ([09:44] Diego).

**Negativas / trade-offs**

- Responsabilidade operacional no cliente: ele precisa implementar a verificação do HMAC ([09:20] Sofia).
- Rotação exige manter duas secrets válidas simultaneamente por 24h (janela de aceite duplicada) ([09:21] Sofia).
- HMAC é calculado sobre o corpo exato do request — qualquer transformação de bytes no caminho invalida a verificação, exigindo cuidado na serialização.

## Referências

- Transcrição: [09:19]–[09:24], [09:44]–[09:45], [09:48].
- Código: `src/middlewares/validate.middleware.ts` (validação Zod onde o `https` será exigido), `src/shared/logger/index.ts` (redaction Pino, padrão para não logar secrets), `prisma/schema.prisma` (nova tabela de configuração de webhook).
- ADRs relacionados: [ADR-005](ADR-005-at-least-once-x-event-id.md), [ADR-006](ADR-006-reuso-padroes-existentes.md).
