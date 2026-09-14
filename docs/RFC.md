# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Equipe do OMS — compilação documental assistida por IA |
| Status | Proposto para revisão |
| Data | 2026-09-05 |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| Escopo | Webhooks outbound para mudanças de status de pedidos |

## Resumo executivo (TL;DR)

Propomos registrar, na mesma transação MySQL que altera um pedido, um snapshot de evento `order.status_changed` em uma transactional outbox. Um worker Node.js separado, inicialmente único, consultará pendências a cada 2 segundos e entregará o evento via HTTPS com HMAC-SHA256. O modelo será at-least-once, com `X-Event-Id` estável para deduplicação, política de retry limitada por cinco marcos temporais e DLQ com replay administrativo auditado. A quantidade total de envios dessa política ainda precisa ser confirmada.

A abordagem evita chamadas externas na transação crítica do pedido e não adiciona Redis ou outra infraestrutura. O CRUD autenticado permitirá configurar endpoints e filtros de status; detalhes de contratos, dados e algoritmos ficam no [FDD](./FDD.md). Esta RFC solicita revisão da arquitetura e, principalmente, resolução das questões abertas antes da implementação.

## Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — querem deixar de consultar `GET /orders` repetidamente e receber notificações quando seus pedidos mudarem de status. Para eles, entrega abaixo de 10 segundos atende à expectativa de “tempo real” ([09:00] Marcos (TRANSCRICAO.md:L18); [09:02] Marcos (TRANSCRICAO.md:L22)). O fluxo é apenas OMS → cliente; inbound webhooks não pertencem ao problema ([09:02] Marcos (TRANSCRICAO.md:L26)).

Hoje o OMS não possui eventos, fila, outbox, DLQ ou notificação externa. Em `src/modules/orders/order.service.ts`, `OrderService.changeStatus` executa em uma transação Prisma a validação da transição, efeitos no estoque, atualização do pedido e inserção em `order_status_history`. Colocar HTTP nesse caminho aumentaria a duração da transação e faria a disponibilidade do cliente interferir na mudança do pedido ([09:04] Bruno (TRANSCRICAO.md:L32); [09:04] Bruno (TRANSCRICAO.md:L36); [09:06] Diego (TRANSCRICAO.md:L48)).

O desenho precisa preservar a atomicidade existente, oferecer recuperação de falhas e seguir os padrões já usados por Express, Prisma, Zod, `AppError`, JWT/RBAC e Pino.

## Objetivos e limites

- <a id="rfc-obj-01"></a>[RFC-OBJ-01] Notificar mudanças de status por um fluxo assíncrono cuja arquitetura seja compatível com a expectativa de menos de 10 segundos em condições normais.
- <a id="rfc-obj-02"></a>[RFC-OBJ-02] Impedir o estado “pedido alterado sem evento registrado”.
- <a id="rfc-obj-03"></a>[RFC-OBJ-03] Permitir autenticação do remetente, verificação de integridade, retry limitado, diagnóstico e recuperação administrativa.
- <a id="rfc-lim-01"></a>[RFC-LIM-01] Não oferecer exactly-once nem ausência de duplicatas.
- <a id="rfc-lim-02"></a>[RFC-LIM-02] Não introduzir Redis ou outra infraestrutura de fila nesta fase.
- <a id="rfc-lim-03"></a>[RFC-LIM-03] Não prometer ordering global: single-worker não impede overtaking durante backoff; o FDD propõe sequência e gating isolados por destino/pedido, e replay após DLQ ainda pode chegar fora de ordem.
- <a id="rfc-lim-04"></a>[RFC-LIM-04] Não incluir inbound webhooks; o fluxo permanece somente OMS → cliente.
- <a id="rfc-lim-05"></a>[RFC-LIM-05] Não incluir email de alerta/fallback nesta fase.
- <a id="rfc-lim-06"></a>[RFC-LIM-06] Não incluir dashboard visual; a primeira fase oferece apenas APIs.
- <a id="rfc-lim-07"></a>[RFC-LIM-07] Não incluir rate limiting de saída; observar o comportamento antes de decidir essa evolução.

## Proposta técnica

### <a id="rfc-dec-01"></a>[RFC-DEC-01] Registro atômico no MySQL

Ao confirmar uma transição válida, o OMS localizará configurações do cliente interessadas no novo status e persistirá os snapshots correspondentes na outbox dentro da transação existente de `changeStatus`. Usar o campo ativo como condição adicional é proposta do FDD. Falha no enqueue causará rollback da transação completa ([09:06] Diego (TRANSCRICAO.md:L48); [09:33] Marcos (TRANSCRICAO.md:L194); [09:34] Bruno (TRANSCRICAO.md:L198); [09:40] Bruno (TRANSCRICAO.md:L238); [09:41] Diego (TRANSCRICAO.md:L244)).

O evento receberá UUID e guardará o JSON já renderizado para preservar o estado no instante da mudança ([09:51] Larissa (TRANSCRICAO.md:L304); [09:52] Larissa (TRANSCRICAO.md:L310)). Modelos, índices, estados e parâmetros operacionais de claim/lease ficam exclusivamente no FDD.

### <a id="rfc-dec-02"></a>[RFC-DEC-02] Processamento em worker separado

Um processo Node.js separado da API, com `PrismaClient` próprio e o mesmo banco, consultará pendências a cada 2 segundos ([09:09] Diego (TRANSCRICAO.md:L60); [09:11] Diego (TRANSCRICAO.md:L70); [09:12] Diego (TRANSCRICAO.md:L80); [09:30] Bruno (TRANSCRICAO.md:L178)). A primeira fase terá um único worker. Isso simplifica a entrega inicial e limita vazão, mas não basta para ordenar eventos durante retry: o FDD propõe sequência monotônica e isolamento por destino/pedido; DLQ libera a fila e replay posterior pode chegar fora de ordem.

O polling é compatível com o cenário nominal, mas não prova sozinho a meta inferior a 10 segundos: um endpoint lento ou backlog pode atrasar entregas seguintes no worker sequencial. Carga, saúde e ponto de medição do aceite ficam definidos no FDD.

Cada tentativa terá timeout de 10 segundos. A política usa os marcos 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas; depois de esgotada, o evento irá para uma DLQ separada, passível de replay manual por usuário `ADMIN`, com auditoria ([09:17] Diego (TRANSCRICAO.md:L104); [09:18] Diego (TRANSCRICAO.md:L110); [09:18] Diego (TRANSCRICAO.md:L114); [09:36] Sofia (TRANSCRICAO.md:L210); [09:42] Diego (TRANSCRICAO.md:L250)). A reunião é ambígua entre cinco tentativas totais e uma tentativa inicial mais cinco retries, e a classificação de respostas retentáveis também precisa ser aprovada.

### <a id="rfc-dec-03"></a>[RFC-DEC-03] Entrega autenticada e idempotência do consumidor

O destino deve ser HTTPS. O OMS assinará com HMAC-SHA256 o corpo enviado, usando secret única por endpoint. A rotação criará uma secret e manterá a anterior válida por 24 horas ([09:22] Sofia (TRANSCRICAO.md:L134); [09:21] Sofia (TRANSCRICAO.md:L130); [09:23] Sofia (TRANSCRICAO.md:L138)). Para não encurtar essa validade mantendo apenas secret atual/anterior, FDD-PROP-04 propõe rejeitar nova rotação durante o grace period. A entrega incluirá `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`, terá payload máximo de 64 KB sem truncamento e não incluirá os itens do pedido ([09:24] Larissa (TRANSCRICAO.md:L144); [09:43] Diego (TRANSCRICAO.md:L256); [09:44] Diego (TRANSCRICAO.md:L262); [09:44] Sofia (TRANSCRICAO.md:L264)).

A garantia é at-least-once. O consumidor deve deduplicar pelo UUID estável de `X-Event-Id`; retries e replay podem produzir duplicatas ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:26] Marcos (TRANSCRICAO.md:L156)).

### <a id="rfc-dec-04"></a>[RFC-DEC-04] Gestão no padrão do OMS

Um novo domínio de webhooks sob `src/modules/` seguirá controller, service, repository, routes e schemas. O CRUD autenticado administrará URL, estado ativo e filtros de status; o OMS gerará a secret na criação. Como defaults sujeitos a aprovação, desativar/remover cancela pendências e remoção é lógica (FDD-PROP-06); replay da mesma entrada de DLQ é idempotente (FDD-PROP-07). Também haverá rotação, consulta das 100 entregas mais recentes e replay restrito a `ADMIN` ([09:27] Bruno (TRANSCRICAO.md:L162); [09:31] Marcos (TRANSCRICAO.md:L184); [09:33] Bruno (TRANSCRICAO.md:L192); [09:33] Marcos (TRANSCRICAO.md:L194); [09:21] Sofia (TRANSCRICAO.md:L130); [09:34] Marcos (TRANSCRICAO.md:L202); [09:36] Sofia (TRANSCRICAO.md:L210)).

O módulo reutilizará Zod, `AppError`, error middleware, Pino, Prisma e `requireRole`, com códigos do domínio prefixados por `WEBHOOK_`. Como o `validate` atual converte toda falha Zod em `VALIDATION_ERROR`, FDD-PROP-08 propõe um mapper opcional e retrocompatível para URL/filtro do webhook. A reunião verbalizou `GET /webhooks/:id/deliveries` ([09:34] Marcos (TRANSCRICAO.md:L202)) e `POST /admin/webhooks/dead-letter/:id/replay` ([09:35] Diego (TRANSCRICAO.md:L206)). Prefixo, paths do CRUD e posição de `customer_id` apresentados no FDD são propostas para revisão.

## Alternativas consideradas

### <a id="rfc-alt-01"></a>[RFC-ALT-01] Chamada HTTP síncrona em `changeStatus`

Foi descartada porque um cliente lento prolongaria uma transação que já atualiza pedido, histórico e estoque; um cliente fora do ar ainda criaria o dilema incorreto de reverter a mudança ou perder a notificação ([09:04] Bruno (TRANSCRICAO.md:L32)). Como análise derivada, o trade-off da solução assíncrona é consistência eventual da entrega e operação de outbox/worker.

### <a id="rfc-alt-02"></a>[RFC-ALT-02] Redis Streams ou infraestrutura de fila

Foi descartada porque o MySQL existente atende ao volume inicial e o time considerou subir um Redis Cluster um caso de overengineering ([09:07] Diego (TRANSCRICAO.md:L52)). Como análise derivada, o trade-off do MySQL é polling e manutenção das tabelas de outbox/DLQ.

### <a id="rfc-alt-03"></a>[RFC-ALT-03] Trigger no banco

Foi descartada porque uma trigger MySQL executa SQL, mas não notifica adequadamente o processo externo; seriam necessários mecanismos improvisados ([09:09] Diego (TRANSCRICAO.md:L64)). O polling introduz consultas periódicas e é compatível com a expectativa nominal sem nova infraestrutura, sujeito às condições de carga e saúde detalhadas no FDD.

### <a id="rfc-alt-04"></a>[RFC-ALT-04] Exactly-once

Foi descartada pela necessidade de coordenação com cada consumidor e pela complexidade adicional ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L154)). At-least-once simplifica o produtor, mas torna a deduplicação uma obrigação do cliente.

## Questões em aberto

As lacunas abaixo resultam da leitura integral: as fontes confirmam as capacidades de base, mas não especificam esses detalhes. A organização de gates por comportamento/deploy é proposta documental.

- <a id="rfc-oq-01"></a>[RFC-OQ-01] Qual estratégia de claim, recuperação e ordenação por destino/pedido será aprovada? Algoritmos e parâmetros propostos ficam no FDD.
- <a id="rfc-oq-02"></a>[RFC-OQ-02] Quais respostas HTTP, timeouts e erros de rede são sucesso, falha permanente ou falha retentável?
- <a id="rfc-oq-03"></a>[RFC-OQ-03] Como as secrets atual e anterior serão protegidas em repouso e quem poderá recuperá-las?
- <a id="rfc-oq-04"></a>[RFC-OQ-04] Qual string será assinada, qual a codificação de `X-Signature` e como `X-Timestamp` participará da verificação?
- <a id="rfc-oq-05"></a>[RFC-OQ-05] Quais paths finais serão adotados para CRUD, rotação e deliveries, e `customer_id` ficará no body ou no path? A reunião deixou esse ponto sem fechamento ([09:32] Larissa (TRANSCRICAO.md:L190); [09:33] Bruno (TRANSCRICAO.md:L192)).
- <a id="rfc-oq-06"></a>[RFC-OQ-06] Qual será a retenção de outbox, deliveries, DLQ e auditoria? Apenas o arquivamento de entregues após “30 dias ou assim” foi citado e explicitamente retirado desta fase ([09:08] Diego (TRANSCRICAO.md:L56)).
- <a id="rfc-oq-07"></a>[RFC-OQ-07] Quais SLOs, métricas, alertas, tracing e estratégia de deploy/rollback serão aprovados?
- <a id="rfc-oq-08"></a>[RFC-OQ-08] A política possui cinco tentativas totais ou uma tentativa inicial mais cinco retries associados aos cinco intervalos?

Retenção definitiva, tracing distribuído e escala horizontal continuam abertos como evoluções, mas não bloqueiam o início do código da primeira fase. Limites de response body, alertas e runbooks devem ser fechados antes do deploy.

## Impacto e riscos

Efeitos e mitigações são análise derivada das decisões e propostas; não são medições de produção.

| ID | Impacto ou risco | Efeito | Mitigação proposta |
| --- | --- | --- | --- |
| <a id="rfc-risk-01"></a>RFC-RISK-01 | Nova escrita na transação de pedido | A mudança de status passa a depender também do enqueue | Índices adequados, teste de rollback e monitoramento de duração da transação |
| <a id="rfc-risk-02"></a>RFC-RISK-02 | Indisponibilidade do worker | Crescimento de pendências e atraso | Processo independente, backlog observável e procedimento de recuperação |
| <a id="rfc-risk-03"></a>RFC-RISK-03 | Duplicatas | Consumidor pode aplicar o evento duas vezes | Contrato explícito e UUID estável em `X-Event-Id` |
| <a id="rfc-risk-04"></a>RFC-RISK-04 | Vazamento de secret | Falsificação de requisições | Secret por endpoint, rotação, TLS, redação de logs e revisão por Sofia |
| <a id="rfc-risk-05"></a>RFC-RISK-05 | Single-worker | Limite de vazão e ponto único | Medir backlog/latência; particionamento permanece evolução futura |
| <a id="rfc-risk-06"></a>RFC-RISK-06 | Contratos ainda abertos | Implementações divergentes | Fechar decisões do comportamento afetado; revisão do código de segurança antes do deploy |

<a id="inv-rfc-02-a"></a><a id="inv-rfc-02-b"></a>A estimativa registrada é de três sprints, incluindo ao menos dois dias úteis para revisão de segurança antes do deploy ([09:47] Larissa (TRANSCRICAO.md:L276); [09:46] Sofia (TRANSCRICAO.md:L274)).

## Decisões relacionadas

- [ADR-001 — Outbox transacional no MySQL](./adrs/ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker separado com polling](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e DLQ](./adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 e rotação de secret](./adrs/ADR-004-hmac-sha256-e-rotacao-de-secret.md)
- [ADR-005 — Entrega at-least-once](./adrs/ADR-005-entrega-at-least-once.md)
- [ADR-006 — Reuso dos padrões do OMS](./adrs/ADR-006-reuso-dos-padroes-do-oms.md)

## Solicitação de revisão

Larissa, Marcos, Bruno, Diego e Sofia devem confirmar se esta síntese preserva as decisões da reunião. Antes de concluir cada comportamento, o time precisa aprovar ou ajustar claim/lease, gating de ordenação, classificação de falhas, proteção/canonicalização e rotação sucessiva das secrets, tradução dos erros Zod, cancelamento em desativação/remoção, replay idempotente e contrato final de APIs. Itens operacionais e futuros seguem os horizontes da seção 18 do FDD. Sofia deverá revisar o código de segurança antes do deploy, como combinado em [09:46] Sofia (TRANSCRICAO.md:L274).
