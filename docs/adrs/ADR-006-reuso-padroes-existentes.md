# ADR-006 — Reuso máximo dos padrões existentes do projeto

## Status

Aceito — decidido em reunião técnica ([09:30] Larissa: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros"; confirmado no resumo [09:48]).

## Contexto

A codebase já tem um padrão claro e consistente: cada domínio é um módulo em `src/modules/` com `controller`, `service`, `repository`, `routes` e `schemas` ([09:27] Bruno). Há também infraestrutura compartilhada madura — erros tipados, logger, validação e tratamento centralizado de erros. A questão era se a feature de webhooks introduziria estruturas próprias ou seguiria os padrões existentes ([09:27] Bruno: "Webhook vai seguir igual").

Este ADR é o que referencia explicitamente os arquivos, módulos e classes do código existente que serão reaproveitados.

## Decisão

O módulo de webhooks reaproveita os padrões e a infraestrutura existente, sem introduzir mecanismos paralelos:

1. **Estrutura de módulo** — `src/modules/webhooks/` com os cinco arquivos do padrão (`webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`), espelhando `src/modules/orders/` ([09:27] Bruno, [09:28] Bruno). A lógica de processamento do worker fica em `webhook.worker.ts`/`webhook.processor.ts` dentro do módulo ([09:28] Bruno).
2. **Classes de erro** — reuso de `AppError` (`src/shared/errors/app-error.ts`) e das subclasses de `src/shared/errors/http-errors.ts`. Novos erros de webhook estendem `AppError` com códigos no prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), exatamente como `InsufficientStockError`/`InvalidStatusTransitionError` já fazem ([09:28] Bruno, [09:29] Larissa: "Prefixo WEBHOOK_ pra tudo do módulo").
3. **Tratamento centralizado de erros** — o `errorMiddleware` (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros do Prisma; vai pegar os erros de webhook **sem precisar mudar nada** ([09:29] Bruno).
4. **Logger Pino** — o logger Pino de `src/shared/logger/index.ts` já está no projeto inteiro; nada novo de logging será introduzido ([09:29] Bruno).
5. **Autorização `requireRole`** — o replay de DLQ reaproveita o `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
6. **Schemas Zod e `validate`** — validação de entrada com Zod via `src/middlewares/validate.middleware.ts`, no mesmo padrão dos `*.schemas.ts` existentes; a exigência de `https` entra aqui ([09:23] Sofia).
7. **Respostas HTTP paginadas** — `paginated()` de `src/shared/http/response.ts` para os endpoints de listagem, como os demais módulos.
8. **PrismaClient por processo** — o worker usa `createPrismaClient()` de `src/config/database.ts` (instância própria, mesmo `DATABASE_URL`), no padrão já existente ([09:30] Bruno).
9. **IDs UUID** — o `event_id` e as novas tabelas seguem o padrão `@default(uuid()) @db.Char(36)` já usado em `prisma/schema.prisma` ([09:51] Larissa: "UUID, segue o padrão do resto do projeto").

## Alternativas Consideradas

1. **Módulo de webhooks com estrutura própria** (fora do padrão `controller/service/repository/routes/schemas`). Descartada: quebraria a consistência da codebase; "webhook fica como módulo igual aos outros" ([09:30] Larissa, [09:27] Bruno).
2. **Novo mecanismo de logging/erro específico de webhook**. Descartada: Pino e o error middleware centralizado já cobrem o caso; "não vamos botar nada novo" ([09:29] Bruno).
3. **Códigos de erro sem prefixo dedicado**. Descartada: o prefixo `WEBHOOK_` foi definido explicitamente para tudo do módulo ([09:29] Larissa), no mesmo estilo de `INSUFFICIENT_STOCK`/`INVALID_STATUS_TRANSITION` ([09:28] Bruno).

## Consequências

**Positivas**

- Código novo indistinguível do existente em estilo e estrutura; curva de aprendizado mínima para o time ([09:30] Larissa).
- Erros de webhook já nascem compatíveis com o error middleware, o logger e o formato de resposta — sem retrabalho ([09:29] Bruno).
- Menos superfície de código novo para revisar (inclusive na revisão de segurança de Sofia, [09:46]).

**Negativas / trade-offs**

- Acoplamento aos padrões atuais: se um padrão do projeto mudar, o módulo de webhooks muda junto.
- O error middleware centralizado não diferencia, por padrão, erros de webhook de outros `AppError` — qualquer tratamento específico precisaria ser adicionado pontualmente.

## Referências

- Transcrição: [09:27]–[09:30], [09:36], [09:46], [09:48], [09:51].
- Código: `src/modules/orders/` (padrão de módulo), `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/config/database.ts`, `prisma/schema.prisma`.
- ADRs relacionados: [ADR-003](ADR-003-retry-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md).
