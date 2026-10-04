# Desafio 5 — Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

O desafio consiste em transformar a transcrição de uma reunião técnica de ~55 minutos ([`TRANSCRICAO.md`](TRANSCRICAO.md)) — na qual tech lead, PM, dois engenheiros e uma engenheira de segurança decidem como construir um **Sistema de Webhooks de Notificação de Pedidos** para um Order Management System (OMS) já em produção — em um pacote completo de design docs: PRD, RFC, FDD, 6 ADRs e um Tracker de rastreabilidade.

A regra de ouro é que **nenhuma informação pode ser inventada**: cada requisito, decisão ou restrição precisa ter origem identificável na transcrição (timestamp + falante) ou no código existente (`src/`, `prisma/`, `tests/`). A entrega é puramente documental — o código da aplicação não foi alterado; ele serve de contexto e referência. O enunciado original do desafio está preservado no repositório base (`mba-ia-desafio-design-docs-com-ia`).

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Pi (coding agent)** | Ferramenta principal de produção: leitura integral da transcrição e do código, extração de decisões/requisitos/descartes com timestamp, redação de PRD, RFC, FDD, ADRs e Tracker, e verificação de conformidade contra os critérios de aceite |
| **Bash/grep/find (via agente)** | Verificação mecânica de evidências: contagem de linhas do Tracker, validação de formato `[hh:mm] Nome`, confirmação de existência de todo caminho de código citado nos documentos |

## Workflow adotado

1. **Análise antes de escrever**: leitura completa da `TRANSCRICAO.md` e mapeamento do código (`src/modules/orders/order.service.ts`, `order.status.ts`, `src/shared/errors/*`, `src/middlewares/*`, `prisma/schema.prisma`, `src/app.ts`, `src/server.ts`, `src/config/*`) para saber exatamente o que existia e podia ser citado.
2. **Extração dirigida com rastreio**: em vez de "gere um PRD da transcrição", a extração foi fatiada por categoria — decisões fechadas, requisitos funcionais, restrições não funcionais, **itens descartados/adiados** (escopo negativo), ganchos com o código — sempre anotando `[hh:mm] Nome` de cada trecho.
3. **Ordem de produção**: **ADRs primeiro** (as decisões fechadas são o esqueleto do pacote) → **PRD** (produto/negócio) → **RFC** (proposta de arquitetura, concisa, linkando os ADRs) → **FDD** (implementação detalhada, ancorada nos arquivos reais) → **TRACKER** (referência cruzada de tudo) → **README** (este documento, por último, relatando o processo real).
4. **Fronteiras entre documentos**: cada documento opera numa altura (produto × arquitetura × decisão × implementação); conteúdo de implementação foi mantido fora do RFC e conteúdo de produto fora do FDD, para evitar duplicação.
5. **Validação final contra os critérios de aceite**: checklist item a item, com verificação mecânica (grep) de contagens, formatos de timestamp e existência dos caminhos de código citados.

## Prompts customizados

**Prompt 1 — Extração com escopo negativo (anti-alucinação)**:

```text
Leia TRANSCRICAO.md integralmente. Extraia 4 listas separadas, cada item com
[hh:mm] + nome do falante:
1. DECISÕES FECHADAS (alguém diz "decidido/anotado/tá decidido" ou a decisão
   é confirmada no resumo final [09:47]-[09:48]);
2. REQUISITOS FUNCIONAIS explícitos (o que o sistema deve fazer);
3. RESTRIÇÕES/RNFs (valores numéricos: timeouts, limites, intervalos, prazos);
4. FORA DE ESCOPO: itens explicitamente descartados ("não", "fora de escopo")
   ou adiados ("próxima fase", "problema do futuro", "observar e decidir depois").
Regra: se um trecho não se encaixa com certeza numa lista, não inclua.
Não resuma falas — cite o timestamp exato de cada item.
```

**Prompt 2 — Ancoragem no código antes do FDD**:

```text
Antes de escrever o FDD, liste os caminhos reais do repositório que a feature
de webhooks vai tocar, lendo o código (não suponha):
- onde fica a transação de mudança de status do pedido e o que ela faz hoje;
- o padrão de classes de erro e códigos (AppError/errorCode) já existente;
- onde estão authenticate/requireRole e o error middleware centralizado;
- o logger e suas regras de redaction;
- as convenções do schema.prisma (ids, @@map, índices) e de montagem de rotas.
Para cada caminho, diga COMO o módulo de webhooks se integra a ele.
Todo caminho citado será verificado com `ls` antes da entrega — caminho
inexistente é erro.
```

**Prompt 3 — Auditoria do Tracker**:

```text
Para cada requisito, decisão, restrição e alternativa dos documentos (PRD,
RFC, FDD, ADRs), verifique se existe linha no TRACKER.md com Fonte e
Localização válidas. Para itens TRANSCRICAO, confirme que o timestamp existe
no arquivo e que a fala atribuída àquele participante realmente contém o item.
Para itens CODIGO, confirme com `ls` que o caminho existe.
Itens sem origem: ou remova do documento, ou sinalize explicitamente como
derivação de uma decisão rastreada.
```

## Iterações e ajustes

Foram necessárias **3 iterações principais** (extração → redação → auditoria), com correções concretas:

1. **Contagens do Tracker estavam erradas no primeiro rascunho**: o resumo de cobertura afirmava "92 linhas (78 TRANSCRICAO)" com base em estimativa. A verificação mecânica com `grep -c` mostrou **95 linhas (81 TRANSCRICAO / 14 CODIGO)** — e revelou que o padrão de contagem incluía indevidamente o cabeçalho da tabela. O resumo foi corrigido pelos números medidos, não pelos supostos.
2. **Vazamento de texto estranho e link quebrado**: a primeira versão do PRD continha uma palavra em outro idioma no §1 ("能力"), corrigida em revisão; no RFC, a tabela de decisões relacionadas apontava para `adrs/ATR-005-...` (typo) em vez de `ADR-005`, o que quebraria o link exigido pelos critérios de aceite. Ambos corrigidos antes do stage.
3. **Boilerplate contraditório em `docs/adrs/README.md`**: o README original do diretório de ADRs prescrevia a nomenclatura `0001-titulo.md`, incompatível com o formato exigido pelo desafio (`ADR-NNN-titulo-em-kebab-case.md`). O arquivo foi substituído por um índice dos 6 ADRs produzidos, eliminando a contradição.
4. **Derivações sinalizadas em vez de disfarçadas**: a matriz de erros do FDD precisava de códigos não citados nominalmente na reunião (ex.: `WEBHOOK_DEAD_LETTER_NOT_FOUND`). Em vez de apresentá-los como se tivessem sido ditos, o FDD (§7) e o Tracker marcam explicitamente que derivam da regra "prefixo `WEBHOOK_` pra tudo do módulo" ([09:29] Larissa) — mantendo a integridade da rastreabilidade.

## Como navegar a entrega

Ordem sugerida de leitura:

| # | Arquivo | O que é |
| --- | --- | --- |
| 0 | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária: a reunião literal (não alterada) |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, público, escopo (incl. fora de escopo), requisitos, métricas, riscos |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Como pretendemos resolver: proposta técnica concisa, alternativas descartadas, questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | As 6 decisões fechadas, uma por ADR (outbox, worker/polling, retry+DLQ, HMAC, at-least-once, reuso de padrões) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: modelo de dados, fluxos, contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e integração com o código existente |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | Rastreabilidade: cada item do pacote → origem na transcrição (`[hh:mm] Nome`) ou no código (caminho do arquivo) |

O código da aplicação (`src/`, `prisma/`, `tests/`) não foi modificado — serve de contexto e referência, como exige o enunciado.
