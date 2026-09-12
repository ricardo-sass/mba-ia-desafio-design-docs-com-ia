<a id="adr-006"></a>

# ADR-006 — Reuso dos padrões do OMS

## Status

Aceito.

## Contexto

O OMS organiza seus domínios sob `src/modules/` e compõe manualmente controllers, services e repositories em `src/app.ts`. As rotas sob `/api/v1` são registradas em `src/routes/index.ts`; validação usa Zod por meio de `src/middlewares/validate.middleware.ts`; autenticação e autorização usam `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.

Erros de domínio derivam de `AppError` (`src/shared/errors/app-error.ts`) e são serializados por `src/middlewares/error.middleware.ts`. Logs estruturados usam Pino e redação de campos sensíveis em `src/shared/logger/index.ts`; requisições recebem correlação em `src/middlewares/request-logger.middleware.ts`. A persistência usa Prisma/MySQL e UUIDs em `prisma/schema.prisma`.

A reunião decidiu evitar uma arquitetura paralela e reutilizar esses padrões ([09:27] Bruno (TRANSCRICAO.md:L162); [09:30] Larissa (TRANSCRICAO.md:L180)).

## Decisão

Especificar a futura feature como novo domínio sob `src/modules/`, seguindo controller, service, repository, routes e schemas. A lógica do consumidor ficará no domínio como processor/worker e será iniciada por um entry point separado equivalente a `src/server.ts` ([09:27] Bruno (TRANSCRICAO.md:L162); [09:28] Bruno (TRANSCRICAO.md:L166); [09:30] Larissa (TRANSCRICAO.md:L180)).

Reutilizar:

- Zod e `validate` para contratos de entrada;
- `authenticate` para o CRUD e `requireRole('ADMIN')` para replay da DLQ;
- subclasses de `AppError`, com códigos do domínio prefixados por `WEBHOOK_`;
- o error middleware e seu envelope `{ error: { code, message, details? } }`;
- Pino e correlação HTTP existentes; ampliar redação e propagação para secrets/eventos do worker é extensão proposta;
- Prisma/MySQL, com `PrismaClient` próprio do processo worker.

A integração de pedido deve ocorrer na transação já existente de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), recebendo o transaction client atual em vez de abrir uma transação independente ([09:40] Bruno (TRANSCRICAO.md:L238); [09:41] Diego (TRANSCRICAO.md:L244)).

Reusar o error middleware não implica que o contrato específico surja automaticamente: FDD-PROP-08 propõe estender `validate` com um mapper Zod opcional para as rotas de webhook. O default `VALIDATION_ERROR` e todos os callers existentes permanecem inalterados.

## Alternativas Consideradas

- <a id="adr-006-alt-01"></a>[ADR-006-ALT-01] **Análise derivada/proposta:** **Novo framework ou serviço independente com stack própria:** não escolhido; aumentaria superfície operacional e divergência sem necessidade registrada.
- <a id="adr-006-alt-02"></a>[ADR-006-ALT-02] **Repository inteiro de webhooks injetado em `OrderService`:** a reunião preferiu uma função de enqueue que receba o transaction client, reduzindo o acoplamento ([09:41] Bruno (TRANSCRICAO.md:L242); [09:41] Diego (TRANSCRICAO.md:L244)).
- <a id="adr-006-alt-03"></a>[ADR-006-ALT-03] **Análise derivada/proposta:** **Novo mecanismo de autenticação, erro ou log:** descartado pela decisão explícita de reuso máximo ([09:29] Larissa (TRANSCRICAO.md:L172); [09:29] Bruno (TRANSCRICAO.md:L174); [09:30] Larissa (TRANSCRICAO.md:L180)).

## Consequências

### Positivas

- <a id="adr-006-pos-01"></a>[ADR-006-POS-01] **Análise derivada/proposta:** Menor curva de aprendizado e maior consistência com os módulos existentes.
- <a id="adr-006-pos-02"></a>[ADR-006-POS-02] Autenticação, autorização, validação, erros e logs mantêm contratos já conhecidos.
- <a id="adr-006-pos-03"></a>[ADR-006-POS-03] O transaction client preserva a atomicidade definida no ADR-001.
- <a id="adr-006-pos-04"></a>[ADR-006-POS-04] Não há necessidade confirmada de nova infraestrutura ou framework.

### Negativas

- <a id="adr-006-neg-01"></a>[ADR-006-NEG-01] **Análise derivada/proposta:** `src/app.ts` e `src/routes/index.ts` precisarão crescer para compor o novo módulo na implementação futura.
- <a id="adr-006-neg-02"></a>[ADR-006-NEG-02] **Análise derivada/proposta:** O logger precisará ampliar a lista de campos redigidos para cobrir secrets do webhook.
- <a id="adr-006-neg-03"></a>[ADR-006-NEG-03] O worker terá ciclo de vida e conexão Prisma próprios, exigindo observabilidade e shutdown específicos.
- <a id="adr-006-neg-04"></a>[ADR-006-NEG-04] **Análise derivada/proposta:** Os testes de integração precisarão incluir e limpar futuras tabelas, exigindo banco isolado selecionado pelo executor; `tests/setup.ts` não verifica isolamento de `DATABASE_URL`.
