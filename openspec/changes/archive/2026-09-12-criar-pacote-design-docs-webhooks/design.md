## Context

O repositório contém um OMS funcional em Node.js, TypeScript, Express, Prisma e MySQL, mas não contém webhooks, eventos, outbox, worker ou DLQ. A mudança atual é exclusivamente documental: converter `ENUNCIADO.md`, `TRANSCRICAO.md` e o código existente em um pacote coerente que permita uma implementação futura, sem modificar a aplicação.

Na reunião, Marcos registra a necessidade de três clientes B2B e a expectativa de notificação abaixo de 10 segundos ([09:00]–[09:03]). O grupo fecha a arquitetura-base entre [09:06] e [09:48]: transactional outbox no MySQL, worker Node separado em polling de 2 segundos, entrega at-least-once, retry limitado com cinco marcos temporais, DLQ, HMAC-SHA256 por endpoint e reuso dos padrões do projeto. A quantidade total de envios do retry permanece ambígua. Larissa encerra assumindo a criação e a revisão do design antes do início do código ([09:50]).

O estado atual que ancora o desenho foi conferido em:

- `src/modules/orders/order.service.ts`: `changeStatus` concentra alteração do pedido, histórico e estoque em `prisma.$transaction`;
- `src/modules/orders/order.status.ts`: transições permitidas e efeitos de estoque;
- `prisma/schema.prisma`: MySQL, UUIDs, modelos e índices existentes;
- `src/app.ts` e `src/routes/index.ts`: composição manual de dependências e registro sob `/api/v1`;
- `src/middlewares/auth.middleware.ts`: autenticação JWT e `requireRole` para `ADMIN`/`OPERATOR`;
- `src/middlewares/error.middleware.ts` e `src/shared/errors/`: `AppError` e envelope de erro centralizado;
- `src/shared/logger/index.ts` e `src/middlewares/request-logger.middleware.ts`: Pino, correlação e redação de dados sensíveis;
- `tests/orders.test.ts` e `tests/setup.ts`: padrão de testes de integração e dependência de MySQL isolado.

Stakeholders da documentação: Larissa (Tech Lead), Marcos (Produto), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), o time que implementará a feature e os revisores do desafio.

## Goals / Non-Goals

**Goals:**

- Entregar PRD, RFC, FDD, de cinco a oito ADRs, Tracker e README, todos em pt-BR e nos caminhos exigidos pelo `ENUNCIADO.md`.
- Fazer cada documento operar na altura correta: PRD em produto, RFC em proposta arquitetural, ADR em decisão isolada, FDD em implementação e Tracker em evidência.
- Transformar decisões confirmadas em conteúdo acionável, sem apresentar recomendações ou lacunas como fatos.
- Rastrear requisitos, decisões, restrições e trade-offs a timestamps ou caminhos reais, atingindo as metas quantitativas do Tracker.
- Validar de forma não destrutiva estrutura, links, caminhos, seções obrigatórias, consistência terminológica e checklist de aceite.

**Non-Goals:**

- Implementar ou alterar qualquer comportamento em `src/`, `prisma/`, `tests/` ou configurações.
- Alterar `TRANSCRICAO.md` ou usar os documentos vazios/preexistentes em `docs/` como fontes primárias.
- Fechar silenciosamente batch, locking, respostas retentáveis, proteção das secrets, canonicalização da assinatura, contrato final dos endpoints, posição de `customer_id`, retenção, SLOs ou operação de deploy/rollback.
- Incluir na primeira fase inbound webhooks, exactly-once, Redis, múltiplos workers, email de fallback, dashboard, rate limiting de saída ou arquivamento da outbox entregue.

## Decisions

### 1. Usar uma matriz de evidências como base comum

**Decisão documental.** Antes e durante a redação, cada afirmação verificável será associada a `ENUNCIADO.md`, a um timestamp/falante de `TRANSCRICAO.md` ou a um caminho real do código. O Tracker final será derivado dessa matriz, não reconstruído por memória.

**Motivação.** O enunciado proíbe requisitos sem origem e exige cobertura mínima de 80%, predominância de fontes da transcrição e ao menos cinco caminhos de código. A transcrição também mistura decisão, sugestão, descarte e dúvida; por exemplo, Redis é alternativa descartada ([09:07] Diego), enquanto rate limiting fica para observação futura ([09:38]–[09:39] Diego/Larissa).

**Alternativa considerada.** Redigir todos os documentos e montar o Tracker apenas ao final. É mais rápido no início, mas aumenta a chance de perder origem, promover hipótese a decisão e exigir retrabalho.

### 2. Produzir na ordem ADRs → RFC → FDD → PRD → Tracker → README

**Decisão documental.** As decisões fechadas serão isoladas primeiro; o RFC as consolidará em nível arquitetural; o FDD especificará a futura implementação; o PRD fechará a visão de produto sem herdar detalhe técnico indevido. O Tracker será consolidado a partir dos identificadores estáveis usados durante a redação, e o README relatará o processo que efetivamente ocorreu.

**Motivação.** Essa ordem segue a recomendação do `ENUNCIADO.md` e reduz inconsistências. ADRs formam o esqueleto decisório; o README precisa ser último para não inventar iterações ainda não realizadas.

**Alternativa considerada.** Começar pelo PRD e gerar todos os arquivos em uma única passagem. A abordagem favorece superficialidade, duplicação e um relato de processo artificial.

### 3. Separar os documentos por pergunta e controlar referências cruzadas

**Decisão documental.** O PRD responderá “por quê e o quê”; o RFC, “qual arquitetura propomos e por quê”; cada ADR, “por que esta decisão isolada”; o FDD, “como construir”; e o Tracker, “qual é a origem”. Detalhes como contratos, payloads, matriz de erros e fluxos ficam no FDD. O RFC fará links, em vez de copiar ADRs ou o FDD.

**Motivação.** Essa fronteira é requisito explícito do enunciado. A repetição integral dificultaria revisão e criaria versões divergentes.

**Alternativa considerada.** Criar um único documento técnico longo. Isso não atende aos artefatos obrigatórios nem preserva as alturas de decisão e implementação.

### 4. Documentar a arquitetura futura como fluxo transacional e assíncrono

**Decisão técnica confirmada a registrar.** O pacote descreverá o seguinte estado futuro, sempre marcado como proposta e não como código existente:

```text
API / mudança de status
  -> transação Prisma/MySQL
       -> pedido + histórico + estoque
       -> snapshot JSON na outbox para endpoints ativos compatíveis com o status
  -> commit

worker Node.js separado (single-worker, polling 2 s)
  -> lê pendências elegíveis
  -> valida limite, assina e envia via HTTPS (timeout 10 s)
  -> registra tentativa/latência/resposta
  -> entregue | agenda o próximo marco da política | move para DLQ

ADMIN autenticado -> replay manual da DLQ -> nova pendência + auditoria
```

A inserção do snapshot na outbox deve ocorrer na mesma transação de `changeStatus`; se falhar, toda a mudança faz rollback ([09:06] Diego; [09:40]–[09:41] Bruno/Diego). O snapshot representa o estado no instante da transição ([09:51]–[09:52] Larissa/Diego). Filtrar os status no enqueue evita linhas sem consumidor ([09:31]–[09:34] Marcos/Bruno/Diego).

O worker separado, com `PrismaClient` próprio por processo, usa o mesmo MySQL e polling de 2 segundos ([09:09]–[09:12] Diego/Larissa). Nesta fase há um único worker e não há garantia global ([09:12]–[09:14]). O single-worker sozinho não impede overtaking durante backoff. O FDD propõe uma sequência monotônica por transição e gating pelo predecessor do mesmo (`webhookId`, `orderId`), para que um destino lento não bloqueie outro; após DLQ, um replay posterior ainda pode chegar fora de ordem.

Como defaults operacionais sujeitos a aprovação, o FDD mantém os parâmetros de janela/claim/lease somente no documento de implementação. A conclusão de sucesso, retry ou DLQ grava delivery e estado em uma transação protegida por token de claim; uma falha nessa transação deixa o item recuperável sem histórico parcial.

**Alternativas descartadas.** HTTP síncrono dentro de `changeStatus` prenderia a transação a endpoints externos ([09:03]–[09:06]); Redis adicionaria infraestrutura desnecessária ([09:07]); trigger MySQL não notificaria o processo externo sem improviso ([09:09]).

### 5. Documentar entrega segura, at-least-once e recuperável

**Decisão técnica confirmada a registrar.** Cada entrega usará HTTPS, timeout de 10 segundos, limite de 64 KB sem truncamento e os headers `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` ([09:20]–[09:25], [09:42]–[09:45]). O corpo será o snapshot enxuto de `order.status_changed`, sem itens ([09:43]–[09:44]).

O contrato confirmado é HMAC-SHA256 sobre os bytes do corpo efetivamente enviado, com secret única por endpoint; `X-Timestamp` não foi confirmado como parte do cálculo ([09:20]–[09:23], [09:44] Sofia/Diego). O FDD separa desse baseline a extensão proposta `<timestamp>.<rawBody>`, que muda o algoritmo do consumidor e exige aprovação/versionamento. A rotação manterá a anterior válida por 24 horas; mantendo apenas secret atual/anterior, nova rotação durante o grace period é rejeitada como proposta para não encurtar a validade prometida.

A entrega será at-least-once, e o UUID de `X-Event-Id` será a chave para deduplicação do consumidor; exactly-once foi descartado pela coordenação e complexidade ([09:24]–[09:26] Diego/Larissa). A política de falha usa os marcos 1 min, 5 min, 30 min, 2 h e 12 h, seguida de DLQ e replay manual por `ADMIN` ([09:15]–[09:19], [09:35]–[09:36]). A reunião também chama a política de “5 tentativas”, portanto ainda é preciso decidir se são cinco envios totais ou uma tentativa inicial mais cinco retries. A semântica de respostas retentáveis continua aberta.

### 6. Especificar gestão autenticada e aderente ao OMS

**Decisão técnica confirmada a registrar.** O pacote definirá CRUD autenticado de configurações, geração e retorno da secret na criação, rotação, filtro de status, histórico das 100 entregas mais recentes e replay administrativo. Dois paths relativos foram verbalizados: `GET /webhooks/:id/deliveries` ([09:34] Marcos) e `POST /admin/webhooks/dead-letter/:id/replay` ([09:18] Diego e [09:35] Diego). Os paths do CRUD, o prefixo completo e a posição de `customer_id` continuam sujeitos a revisão ([09:31]–[09:33]).

Como defaults de produto sujeitos a aprovação, o FDD propõe que desativar/remover um endpoint cancele pendências sem apagar histórico e que repetir o replay da mesma entrada de DLQ reutilize uma única outbox. A reunião confirma as operações, mas não essas semânticas.

O futuro módulo seguirá controller/service/repository/routes/schemas sob `src/modules/webhooks`, Zod, `AppError`, erros `WEBHOOK_*`, Pino, error middleware e `requireRole('ADMIN')` no replay ([09:27]–[09:36]). Isso é coerente com `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts` e `src/shared/logger/index.ts`.

Como `src/middlewares/validate.middleware.ts` converte atualmente todo `ZodError` em `VALIDATION_ERROR`, o FDD propõe um segundo argumento opcional para mapear issues nas rotas de webhook. URL e filtro produzem suas subclasses de `AppError`; callers existentes e demais issues mantêm o default.

**Alternativa considerada.** Introduzir novos frameworks, mensageria ou mecanismos paralelos de autenticação/log. Foi rejeitada em favor do “reuso máximo” confirmado em [09:27]–[09:30].

### 7. Tratar observabilidade, testes e operação em níveis de evidência

**Decisão documental.** O FDD cobrirá métricas, logs e tracing; testes unitários, integração transacional, worker, segurança e ponta a ponta; e uma estratégia prospectiva de migração/deploy/rollback. Como a reunião não fechou nomes de métricas, SLOs, propagação de tracing nem operação de rollout, o texto distinguirá:

- **confirmado:** Pino existente, histórico de entrega com resultado, resposta e latência, meta de produto abaixo de 10 segundos e revisão de segurança;
- **recomendado para revisão:** métricas e correlação necessárias para validar o fluxo;
- **em aberto:** SLOs finais, retenção, deploy, rollback e detalhes operacionais.

A meta inferior a 10 segundos será acompanhada de cenário nominal explícito de carga, backlog e latência do destino, além de teste que demonstra a violação possível quando uma chamada lenta antecede outra no worker sequencial. Critérios derivados de defaults serão condicionais à aprovação, não requisitos confirmados.

**Alternativa considerada.** Omitir tópicos não decididos. Isso impediria o FDD de ser acionável; registrá-los como abertos preserva honestidade sem inventar decisões.

## Risks / Trade-offs

- **[Afirmação sem fonte ou fonte imprecisa]** → Manter IDs estáveis, registrar evidência durante a redação e rejeitar linha do Tracker sem timestamp/falante ou caminho existente.
- **[Contratos propostos parecerem decisões fechadas]** → Rotular recomendações e questões abertas no RFC/FDD e não registrá-las como ADRs aceitos.
- **[Duplicação entre PRD, RFC, ADRs e FDD]** → Aplicar as fronteiras documentais e preferir links/resumos de uma frase.
- **[Inconsistência entre nomes de endpoint, headers, payload e erros]** → Criar um glossário/contrato canônico no FDD e executar busca cruzada antes da conclusão.
- **[Cobertura nominal, mas superficial, da checklist]** → Validar cada critério do `ENUNCIADO.md`, incluindo contagens mínimas, exemplos e atributos exigidos.
- **[ADRs contados como blocos apesar de conterem unidades distintas]** → Identificar e rastrear separadamente cada decisão, alternativa e consequência, distinguindo evidência direta de inferência derivada.
- **[Caminhos de código desatualizados ou inexistentes]** → Validar cada caminho com `test -e`/`rg --files`; o código permanece somente leitura.
- **[Ordering confundido com garantia forte]** → Registrar explicitamente que depende do single-worker e não é global ([09:12]–[09:14]).
- **[Secrets aparecerem em exemplos ou logs]** → Usar valores fictícios/redigidos nos documentos e exigir extensão dos caminhos de redação do Pino na futura implementação.

## Migration Plan

Esta mudança não migra banco nem publica software. A produção documental seguirá etapas reversíveis:

1. Extrair decisões, requisitos, exclusões e dúvidas da transcrição e mapear o código atual.
2. Criar de cinco a oito ADRs apenas para decisões fechadas.
3. Redigir o RFC e submetê-lo conceitualmente aos participantes listados como revisores.
4. Redigir o FDD com contratos propostos, integração real, testes e questões abertas.
5. Consolidar o PRD no nível de produto.
6. Montar o Tracker, escrever o README com o processo real e executar a revisão cruzada.

Como rollback documental, cada arquivo pode ser revisado sem impacto no runtime. O FDD deverá, porém, explicitar que a futura implementação necessita migrações versionadas, implantação independente do worker, critérios de habilitação e rollback; o formato definitivo dessa operação permanecerá aberto até decisão do time. A revisão de segurança por Sofia deverá reservar ao menos dois dias úteis antes do deploy futuro ([09:46]–[09:49]).

## Open Questions

- A janela de 20 candidatos, o claim individual com `SKIP LOCKED` e o lease de 60 segundos oferecem o trade-off operacional correto?
- O gating por (`webhookId`, `orderId`), a sequência monotônica e a liberação após DLQ preservam o trade-off correto entre ordenação e bloqueio indefinido sem bloquear outro destino?
- A política representa cinco tentativas totais ou uma tentativa inicial seguida dos cinco intervalos de retry?
- Quais respostas HTTP e erros de rede são sucesso, falha permanente ou falha retentável?
- Como secrets atuais e anteriores serão criptografadas/protegidas em repouso e quem poderá lê-las?
- Qual é a string canônica assinada, a codificação de `X-Signature` e a semântica de `X-Timestamp`?
- Quais são os caminhos e métodos finais do CRUD, da rotação e das consultas, e onde `customer_id` aparece?
- Cancelar pendências ao desativar/remover e tornar replay repetido idempotente refletem a semântica de produto desejada?
- Qual política final de retenção vale para outbox, deliveries, DLQ e auditoria? É evolução fora da primeira fase e não bloqueia o início do código.
- Quais métricas, SLOs, alertas e estratégia de tracing serão aprovados?
- Como ocorrerão deploy, habilitação gradual, rollback e recuperação operacional do worker?
