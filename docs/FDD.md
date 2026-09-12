# FDD — Sistema de Webhooks de Notificação de Pedidos

## 1. Contexto e motivação técnica

O OMS atual é uma API Node.js/TypeScript com Express, Prisma e MySQL. Não existem componentes de webhook, eventos, outbox, worker ou DLQ. A mudança de status ocorre em `OrderService.changeStatus`, dentro de `prisma.$transaction`, junto ao histórico e aos efeitos de estoque (`src/modules/orders/order.service.ts`).

Uma chamada HTTP nesse fluxo faria a transação depender da latência e disponibilidade do cliente ([09:04] Bruno (TRANSCRICAO.md:L32); [09:04] Bruno (TRANSCRICAO.md:L36); [09:06] Diego (TRANSCRICAO.md:L48)). Por isso, a futura implementação deverá registrar o fato de forma atômica e entregá-lo assincronamente. Este FDD detalha como construir a proposta aprovada nos ADRs; onde a reunião não decidiu o contrato, o texto usa a marca **PROPOSTA — REQUER APROVAÇÃO**.

## 2. Objetivos técnicos

- <a id="fdd-obj-01"></a>[FDD-OBJ-01] Persistir o snapshot do evento na mesma transação da mudança de status, sem janela de perda.
- <a id="fdd-obj-02"></a>[FDD-OBJ-02] Entregar eventos em processo separado, por polling de 2 segundos, sem bloquear a API.
- <a id="fdd-obj-03"></a>[FDD-OBJ-03] Implementar retry limitado, DLQ persistida e replay administrativo auditado.
- <a id="fdd-obj-04"></a>[FDD-OBJ-04] Assinar cada entrega com HMAC-SHA256 e secret exclusiva do endpoint, sempre via HTTPS.
- <a id="fdd-obj-05"></a>[FDD-OBJ-05] Tornar duplicatas identificáveis por um `event_id` UUID estável, conforme a garantia at-least-once.
- <a id="fdd-obj-06"></a>[FDD-OBJ-06] Reutilizar a arquitetura modular, Prisma, Zod, JWT/RBAC, `AppError` e Pino do OMS.
- <a id="fdd-obj-07"></a>[FDD-OBJ-07] Expor configuração e histórico suficientes para integração e diagnóstico pelos clientes.

## 3. Escopo e exclusões

### 3.1 Incluído

O uso do campo ativo para habilitar enqueue e os controles de exposição da secret são extensões propostas; filtros de status e geração/rotação da secret são confirmados.

- CRUD autenticado de endpoints outbound, ativação e filtros por status.
- Geração de secret pelo OMS, retorno controlado na criação e rotação com coexistência por 24 horas.
- Evento `order.status_changed` com snapshot enxuto, sem itens.
- Transactional outbox, worker separado single-worker, tentativas e histórico das 100 entregas mais recentes.
- Política limitada com marcos em 1 min, 5 min, 30 min, 2 h e 12 h; cardinalidade total pendente, DLQ e replay por `ADMIN`.
- Headers `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- Timeout de 10 segundos, limite de 64 KB sem truncamento e códigos de domínio `WEBHOOK_*`.

<a id="inv-fdd-02-i"></a>

<a id="inv-fdd-02-f"></a>

<a id="inv-fdd-02-a"></a>

<a id="inv-fdd-02-d"></a>

<a id="inv-fdd-02-h"></a>

<a id="inv-fdd-02-c"></a>

<a id="inv-fdd-02-g"></a>

<a id="inv-fdd-02-b"></a>

<a id="inv-fdd-02-j"></a>

<a id="inv-fdd-02-e"></a>

### 3.2 Excluído ou futuro

- HTTP síncrono dentro de `changeStatus`, Redis ou outra fila externa.
- Inbound webhooks e garantia exactly-once.
- Múltiplos workers, particionamento e garantia de ordering global.
- Email de alerta/fallback, dashboard visual e rate limiting de saída.
- Arquivamento/limpeza de outbox entregue; política definitiva de retenção.

As fontes individuais de cada exclusão estão no mapa INV-FDD-02 do Tracker; HTTP síncrono foi rejeitado por Bruno, inbound por Marcos, Redis/exactly-once/escala por Diego e email/dashboard/rate limiting por Larissa.

## 4. Decisões, recomendações e questões abertas

| Classificação | Item | Estado |
| --- | --- | --- |
| Confirmado | MySQL outbox na transação de pedido, com snapshot JSON | [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Confirmado | Worker separado, single-worker, polling a cada 2 s | [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |
| Confirmado parcialmente | Retry limitado com cinco marcos, DLQ separada e replay manual `ADMIN`; quantidade total de envios em aberto | [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Confirmado | HMAC-SHA256, secret por endpoint e rotação com 24 h | [ADR-004](./adrs/ADR-004-hmac-sha256-e-rotacao-de-secret.md) |
| Confirmado | At-least-once e UUID em `X-Event-Id` | [ADR-005](./adrs/ADR-005-entrega-at-least-once.md) |
| Confirmado | Reuso dos padrões do OMS | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-oms.md) |
| Proposta | Um registro entregável de outbox por endpoint inscrito | Derivada de secret/filtro/retry por endpoint; requer revisão |
| Proposta | Batch, claim/locking e recuperação após crash | Default FDD-PROP-02 requer aprovação |
| Proposta | Classificação de resposta retentável/permanente | Default FDD-PROP-01 requer aprovação |
| Proposta | Persistência da secret e contrato canônico da assinatura | Defaults FDD-PROP-03/04 requerem revisão de Segurança |
| Proposta | Gating de ordenação por destino/pedido durante retry | Default FDD-PROP-05 requer aprovação |
| Proposta | Cancelamento de pendências ao desativar/remover e replay idempotente | Defaults FDD-PROP-06/07 requerem aprovação |
| Proposta | Tradução de falhas Zod para erros específicos de webhook | Default FDD-PROP-08 requer aprovação |
| Proposta | Cenário mensurável da meta inferior a 10 s | Default FDD-PROP-09 requer aprovação de Produto/Plataforma |
| Proposta | Finalização atômica de tentativa com token de claim | Default FDD-PROP-10 requer aprovação |
| Aberto | Paths finais e posição de `customer_id` | Contratos abaixo são propostas |

### 4.1 Defaults técnicos propostos para aprovação

Estes defaults não vieram prontos da reunião. Eles são recomendações explícitas para permitir que os revisores concluam o desenho; não adquirem status de decisão até aprovação no RFC/ADR correspondente.

#### <a id="fdd-prop-01"></a>[FDD-PROP-01] Classificação de resultados HTTP

**Premissas de origem:** a reunião exige timeout de 10 segundos ([09:42] Diego (TRANSCRICAO.md:L250)), retry com teto e DLQ para indisponibilidade ([09:15] Diego (TRANSCRICAO.md:L92); [09:18] Diego (TRANSCRICAO.md:L110)), mas não classifica respostas.

Proposta:

- `200–299`: sucesso.
- Timeout, falha de DNS/conexão/TLS, `408`, `425`, `429` e `500–599`: falha retentável.
- `300–399` e demais `400–499`: falha permanente e envio direto à DLQ, preservando a tentativa no histórico.
- Não seguir redirects automaticamente; uma nova URL deve passar novamente pela validação HTTPS do cadastro.

O objetivo é retentar falhas presumivelmente transitórias e evitar repetir erros de contrato. Os grupos de status e o tratamento de redirects são premissas técnicas propostas, não fatos da transcrição.

#### <a id="fdd-prop-02"></a>[FDD-PROP-02] Claim e recuperação após crash

**Premissas de origem:** a reunião menciona “batch pequeno”, índice por estado/`created_at` ([09:08] Diego (TRANSCRICAO.md:L56)), single-worker ([09:12] Diego (TRANSCRICAO.md:L80)) e timeout de 10 segundos ([09:42] Diego (TRANSCRICAO.md:L250)).

Proposta:

- Consultar uma janela de até 20 IDs candidatos, sem reservá-los e sem alterar seus estados. Essa janela materializa apenas o “batch pequeno”.
- Imediatamente antes de cada chamada HTTP, abrir uma transação curta, revalidar elegibilidade e gating de FDD-PROP-05, selecionar **um** evento com `FOR UPDATE SKIP LOCKED`, marcá-lo `PROCESSING`, preencher `lockedAt`/`lockedBy` e então liberar a transação.
- Manter concorrência HTTP 1 e não reivindicar o próximo evento até persistir o resultado ou reagendamento do atual. Um candidato que ficou inelegível entre a consulta e o claim é ignorado e poderá reaparecer em polling posterior.
- Considerar expirado após 60 segundos somente o claim individual em curso. No bootstrap e a cada polling, retornar `PROCESSING` expirado para `PENDING`, invalidar `claimToken` e preservar `event_id`, payload e tentativa já iniciada.
- A recuperação não registra sucesso presumido; eventual reenvio é compatível com at-least-once.

A janela 20 e o lease de 60 segundos são valores de partida. Como há no máximo um item reservado e o timeout HTTP é de 10 segundos, o lease não expira enquanto outros 19 itens aguardam localmente; se uma tentativa ultrapassar o prazo por crash ou falha de controle, apenas o item em curso será recuperado. Os valores devem ser ajustados por métricas, sem habilitar múltiplos workers nesta fase.

#### <a id="fdd-prop-03"></a>[FDD-PROP-03] Contrato confirmado e extensão proposta da assinatura

**Contrato confirmado na reunião:** HMAC-SHA256 é calculado sobre os bytes do corpo efetivamente enviado, usando secret exclusiva do endpoint; a assinatura segue em `X-Signature`. `X-Timestamp` também é enviado, mas a reunião não decidiu incluí-lo no HMAC ([09:22] Sofia (TRANSCRICAO.md:L134); [09:20] Sofia (TRANSCRICAO.md:L120); [09:44] Diego (TRANSCRICAO.md:L262)).

O baseline confirmado é HMAC sobre o corpo. Os detalhes de serialização, codificação e representação temporal abaixo são propostas de implementação:

- Serializar uma vez o JSON e tratar seus bytes UTF-8 como `rawBody`; persistir, assinar e enviar exatamente esses bytes.
- A mensagem do HMAC é somente `rawBody`.
- A codificação de `X-Signature` ainda precisa ser aprovada; hexadecimal minúsculo com `v1=<hex>` é o default proposto.
- **Proposta de formato:** `X-Timestamp` contém Unix time em segundos, em base 10, mas não participa do cálculo confirmado.

**ALTERAÇÃO PROPOSTA — REQUER APROVAÇÃO DE SEGURANÇA E DOS CONSUMIDORES:** substituir a mensagem confirmada por `<X-Timestamp>.<rawBody>`. Essa extensão vincula timestamp e corpo e permite ao consumidor aplicar uma janela anti-replay, mas muda o cálculo que toda integração precisa reproduzir. Ela só pode substituir o baseline `HMAC(rawBody)` após aprovação e comunicação/versionamento do contrato; até lá, a implementação deve assinar somente o corpo.

Como recomendação de segurança em qualquer variante, o consumidor compara o digest em tempo constante. Formato, codificação e eventual janela temporal dependem de aprovação de Sofia.

#### <a id="fdd-prop-04"></a>[FDD-PROP-04] Armazenamento e rotação de secrets

**Premissas de origem:** secret única por endpoint, geração pelo OMS ([09:31] Marcos (TRANSCRICAO.md:L184)), coexistência da anterior por 24 horas e reuso da stack atual ([09:21] Sofia (TRANSCRICAO.md:L130); [09:30] Larissa (TRANSCRICAO.md:L180)).

Proposta:

- Gerar 32 bytes criptograficamente aleatórios e representar a secret ao cliente em base64url com prefixo `whsec_`.
- Retornar o valor somente na criação e na rotação.
- Criptografar em repouso com AES-256-GCM do módulo nativo `node:crypto`; manter a chave-mestra fora do banco em `WEBHOOK_SECRET_ENCRYPTION_KEY`, validada pelo padrão de `src/config/env.ts`.
- Persistir ciphertext, IV e authentication tag da secret atual. Durante a rotação, mover esse conjunto para os campos anteriores e gravar `previousSecretExpiresAt = now + 24 h`.
- Antes de rotacionar, bloquear a linha do endpoint e verificar `previousSecretExpiresAt`. Se a secret anterior ainda estiver válida, rejeitar a nova rotação com `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS` e informar a expiração em `details`; não substituir nem encurtar o grace period em curso.
- Quando `previousSecretExpiresAt <= now`, limpar o material anterior expirado na mesma transação e então permitir a próxima rotação. A verificação sob lock serializa solicitações concorrentes.
- Durante as 24 horas, assinar com ambas as secrets e publicar dois valores `v1=<hex>` no mesmo `X-Signature`; depois, remover o material anterior e emitir apenas a assinatura atual.

Essa proposta torna “as duas secrets válidas” observável para consumidores ainda não migrados sem expor as secrets no header. Rejeitar uma segunda rotação durante o grace period mantém o modelo de apenas duas secrets sem violar as 24 horas prometidas à anterior. A gestão da chave-mestra, o formato de múltiplas assinaturas e a regra de rotação sucessiva exigem aprovação formal de Segurança.

#### <a id="fdd-prop-05"></a>[FDD-PROP-05] Gating de ordenação por destino/pedido

**Premissas de origem:** a reunião associa a ordenação por pedido ao single-worker e reconhece que não há ordering global ([09:12] Diego (TRANSCRICAO.md:L80); [09:13] Larissa (TRANSCRICAO.md:L86); [09:13] Diego (TRANSCRICAO.md:L84)), mas não trata retry, múltiplos destinos do mesmo pedido nem timestamps iguais. Ordenar somente eventos já vencidos por `createdAt` permite overtaking durante backoff; usar apenas `orderId` no gating faz um endpoint indisponível bloquear outro saudável.

Proposta:

- Cada transição de status recebe um `orderSequence` inteiro monotônico. Dentro da transação de `changeStatus`, bloquear a linha do pedido, incrementar `Order.webhookSequence` e copiar o novo valor para todas as outboxes daquela transição. Assim, timestamps iguais não decidem a ordem.
- Um evento só pode ser reivindicado se não existir evento com `orderSequence` menor para o mesmo par (`webhookId`, `orderId`) em estado não terminal (`PENDING` ou `PROCESSING`), mesmo que o `nextAttemptAt` do predecessor ainda esteja no futuro.
- A consulta aplica esse critério com `NOT EXISTS` sobre o mesmo destino/pedido e o worker o revalida no claim individual de FDD-PROP-02. Para desempate global entre pares independentes, usa `(createdAt, id)`; esse desempate não define a sequência do pedido.
- O evento em backoff bloqueia apenas eventos posteriores enviados ao **mesmo endpoint para o mesmo pedido**. Outro endpoint inscrito e outros pedidos continuam elegíveis.
- `DELIVERED`, `FAILED` já materializado em DLQ e `CANCELLED` liberam o próximo evento do pedido. Essa escolha evita bloqueio indefinido, mas um replay posterior da DLQ pode chegar depois de eventos mais novos.

O gating preserva a ordem de cada pedido vista por cada destino até entrega, DLQ ou cancelamento; não oferece ordering global, nem recompõe a ordem histórica de um replay. O custo aceito pela proposta é head-of-line blocking somente dentro do par destino/pedido enquanto o predecessor aguarda retry.

#### <a id="fdd-prop-06"></a>[FDD-PROP-06] Desativação, remoção e eventos pendentes

**Premissas de origem:** a reunião confirma editar/remover configurações ([09:33] Bruno (TRANSCRICAO.md:L192)) e filtrar por status na inserção; o campo ativo foi aceito no cadastro, e usá-lo como condição de elegibilidade é proposta de integração. O filtro é confirmado por [09:33] Marcos (TRANSCRICAO.md:L194) e [09:34] Bruno (TRANSCRICAO.md:L198); o campo ativo aparece em [09:21] Bruno (TRANSCRICAO.md:L128), confirmado por [09:21] Sofia (TRANSCRICAO.md:L130). O destino das pendências já persistidas não foi definido.

Proposta:

- `PATCH active=false` interrompe novos enqueues e, na mesma transação, marca como `CANCELLED` todas as pendências daquele endpoint, com `cancelReason=ENDPOINT_DISABLED`.
- `DELETE` é remoção lógica por `deletedAt`: interrompe novos enqueues, preserva histórico/auditoria/FKs e cancela as pendências com `cancelReason=ENDPOINT_DELETED` na mesma transação.
- O worker revalida `active=true` e `deletedAt IS NULL` no claim e imediatamente antes do HTTP. Se a configuração deixar de ser elegível nesse intervalo, cancela a pendência sem fazer a chamada.
- Uma chamada já enviada não pode ser recolhida. Se sua resposta chegar após a desativação/remoção, a tentativa fica no histórico, mas uma falha não agenda retry e o evento termina `CANCELLED`.
- Reativar um endpoint não ressuscita eventos cancelados; somente mudanças futuras geram novas outboxes. Eventos cancelados não entram na DLQ nem podem ser reprocessados.

Essa semântica privilegia a intenção operacional de parar envios e a preservação de auditoria. Ela é uma decisão de produto proposta e precisa de aprovação de Marcos e Bruno.

#### <a id="fdd-prop-07"></a>[FDD-PROP-07] Idempotência de replay repetido

**Premissas de origem:** a reunião exige replay manual de DLQ por `ADMIN` e identificação do executor ([09:18] Diego (TRANSCRICAO.md:L114); [09:36] Sofia (TRANSCRICAO.md:L210)), sem definir chamadas repetidas.

Proposta:

- A primeira solicitação cria, em transação, uma nova outbox `PENDING` que preserva `eventId`, snapshot e destino; grava `replayedAt`, `replayedByUserId` e `replayOutboxId` na entrada da DLQ.
- Uma restrição única sobre a relação com a entrada da DLQ impede duas outboxes de replay para a mesma solicitação original, inclusive sob concorrência.
- A primeira chamada retorna `202 Accepted`; chamadas posteriores retornam `200 OK` com o mesmo `replayOutboxId` e o estado atual, sem criar nova pendência nem nova auditoria de execução.
- Se o evento reprocessado falhar até gerar uma **nova** entrada de DLQ, essa nova entrada constitui outra unidade replayável e pode receber sua própria solicitação idempotente.

Isso torna retry de cliente seguro e elimina a necessidade de `409` para repetição legítima. A preservação do `eventId` mantém a deduplicação at-least-once; não promete uma nova entrega para cada clique.

#### <a id="fdd-prop-08"></a>[FDD-PROP-08] Tradução de validação Zod para erros do domínio

**Premissas de origem:** a reunião pede códigos `WEBHOOK_*` e reuso de Zod/`AppError` ([09:29] Larissa (TRANSCRICAO.md:L172); [09:30] Larissa (TRANSCRICAO.md:L180)). Hoje `src/middlewares/validate.middleware.ts` captura qualquer `ZodError` e cria `ValidationError`, que o `src/middlewares/error.middleware.ts` serializa como `VALIDATION_ERROR`; portanto, somente definir mensagens nos schemas não produz os códigos específicos da matriz.

Proposta:

- Estender `validate(schemas)` de forma retrocompatível para `validate(schemas, options?)`, onde `options.mapZodError(error): AppError` é opcional. Sem mapper, todas as rotas existentes continuam produzindo `VALIDATION_ERROR` com a lista atual de `details`.
- No futuro módulo de webhooks, criar `WebhookInvalidUrlError` e `WebhookInvalidStatusFilterError` como subclasses de `BadRequestError`, com códigos `WEBHOOK_INVALID_URL` e `WEBHOOK_INVALID_STATUS_FILTER`.
- Criar `mapWebhookZodError`: se houver issue cujo primeiro segmento do path seja `url`, retornar `WebhookInvalidUrlError`; senão, se houver issue em `statuses`, retornar `WebhookInvalidStatusFilterError`; demais issues continuam como `ValidationError`. Em múltiplas falhas, essa precedência é determinística e o erro mantém em `details` todas as issues, não apenas a escolhida.
- As rotas de criação/edição chamam `validate(..., { mapZodError: mapWebhookZodError })`. O service lança as mesmas classes quando uma invariante de URL/filtro for detectada depois do parse. O `errorMiddleware` não muda: ele já serializa qualquer `AppError` usando `errorCode`.

Assim, o comportamento global permanece compatível e a tradução acontece na fronteira de validação do domínio, antes do controller. `WEBHOOK_SECRET_REQUIRED` continua reservado a uma operação cuja semântica ainda precisa ser aprovada; não deve ser emitido por um schema arbitrário.

#### <a id="fdd-prop-09"></a>[FDD-PROP-09] Cenário verificável da meta de latência

**Premissas de origem:** Produto espera entrega abaixo de 10 segundos em condições normais ([09:02] Marcos (TRANSCRICAO.md:L22)), e o polling de 2 segundos foi aceito como compatível ([09:09] Diego (TRANSCRICAO.md:L60); [09:10] Larissa (TRANSCRICAO.md:L68)). A reunião não definiu carga, percentil nem saúde do destino. Com concorrência HTTP 1, uma tentativa anterior que consuma o timeout de 10 segundos pode fazer o evento seguinte violar a meta.

Proposta de cenário de aceite, sujeita a Produto/Plataforma:

- Executar 100 transições elegíveis a taxa constante máxima de 1 evento por segundo, com uma configuração de webhook destinatária por transição.
- Iniciar com outbox sem backlog/retry vencido, worker e banco saudáveis e endpoint de teste respondendo `2xx` em até 500 ms.
- Medir, por `eventId`, do commit da transação do pedido até o recebimento do primeiro byte pelo servidor de teste; todas as 100 amostras devem ficar abaixo de 10 segundos.
- Publicar junto ao resultado a idade máxima da outbox e latências de banco/endpoint, para que “condições normais” seja reproduzível.

Esse teste valida o cenário nominal, não constitui SLO sob saturação. Um teste separado deve colocar uma chamada de 10 segundos antes de um destino saudável e comprovar/registrar que o worker sequencial pode ultrapassar a meta. Se Produto exigir menos de 10 segundos também com backlog, fan-out ou destino anterior lento, concorrência 1 deverá ser revista; polling de 2 segundos, sozinho, não demonstra a meta.

#### <a id="fdd-prop-10"></a>[FDD-PROP-10] Finalização atômica de cada tentativa

**Premissas de origem:** a reunião exige persistência/histórico e recuperação por retry/DLQ, enquanto o código atual demonstra uso de transações Prisma para manter efeitos relacionados consistentes (`src/modules/orders/order.service.ts`). O HTTP precisa permanecer fora da transação, mas as gravações que concluem seu resultado não podem ficar parcialmente aplicadas.

Proposta:

- No claim individual, gerar `claimToken` UUID, incrementar/fixar `attemptNumber` e persistir `PROCESSING`, `lockedAt`, `lockedBy` e o token na mesma transação.
- Depois do HTTP, abrir uma nova transação e bloquear a outbox. Finalizar somente se ela ainda estiver `PROCESSING` e seu `claimToken` coincidir; um resultado tardio de lease já recuperado não pode alterar estado nem criar histórico.
- **Sucesso:** inserir `WebhookDelivery` e atualizar a outbox para `DELIVERED`, limpando campos de lock, na mesma transação.
- **Falha retentável:** inserir `WebhookDelivery` e atualizar a outbox para `PENDING`, com tentativa/`nextAttemptAt` e locks limpos, na mesma transação.
- **Falha permanente ou esgotada:** inserir `WebhookDelivery`, inserir `WebhookDeadLetter` protegida por unicidade sobre a outbox/tentativa terminal e atualizar a outbox para `FAILED`, limpando locks, na mesma transação.
- **Endpoint desativado/removido durante o HTTP:** inserir `WebhookDelivery` e, se a tentativa não teve sucesso, atualizar a outbox para `CANCELLED` sem retry, na mesma transação, conforme FDD-PROP-06. Uma resposta de sucesso ainda conclui como `DELIVERED`.
- Se qualquer transação de conclusão falhar, nenhuma de suas gravações é confirmada: o item permanece `PROCESSING` e a recuperação do lease o devolve a `PENDING`. O possível reenvio mantém `eventId` e é uma duplicata admissível por at-least-once.
- A recuperação de lease não inventa `WebhookDelivery`, pois não conhece o resultado externo. Ela troca `PROCESSING` por `PENDING` somente se `lockedAt` expirou e invalida o `claimToken`; qualquer finalizador antigo registra apenas log de resultado descartado.

Esse protocolo torna histórico, estado e DLQ consistentes sem manter transação aberta durante I/O externo. `claimToken` também evita que um worker considerado morto finalize depois que a tentativa já foi reivindicada novamente.

## 5. Arquitetura proposta

```text
PATCH /api/v1/orders/:id/status
        |
        v
OrderService.changeStatus
        |
        v
transação Prisma / MySQL
  +-- valida transição e estoque
  +-- atualiza Order
  +-- cria OrderStatusHistory
  +-- consulta webhooks ativos do customer filtrados por to_status
  +-- grava snapshots na WebhookOutbox
        |
        +-- falha: rollback total
        +-- commit: resposta normal da API

processo worker (PrismaClient próprio)
  +-- polling a cada 2 s
  +-- janela de até 20 candidatos; gating pelo predecessor de sequência do mesmo destino/pedido
  +-- claim individual com lease imediatamente antes do HTTP
  +-- assina corpo e POST HTTPS por endpoint
  +-- transação de conclusão: WebhookDelivery + estado
        +-- sucesso -> DELIVERED
        +-- falha retentável -> próximo retry
        +-- esgotado/permanente -> FAILED + WebhookDeadLetter

POST /api/v1/admin/webhooks/dead-letter/:id/replay
  +-- authenticate + requireRole('ADMIN')
  +-- cria nova pendência preservando event_id
  +-- registra auditoria do usuário
```

### 5.1 Componentes futuros

Responsabilidades propostas por analogia com controller/service/repository existentes; a reunião confirma a estrutura modular, a função com `tx` e o entry point separado.

- `WebhookController`: traduz requests/responses e encaminha erros com `next`.
- `WebhookService`: regras de cadastro, rotação, listagem e replay.
- `WebhookRepository`: persistência de configurações, entregas e DLQ.
- `publishWebhookEvent(tx, order, fromStatus, toStatus)`: consulta inscrições e grava snapshots usando o `Prisma.TransactionClient` recebido de `changeStatus` ([09:41] Bruno (TRANSCRICAO.md:L242); [09:41] Diego (TRANSCRICAO.md:L244)).
- `WebhookProcessor`: claim, assinatura, HTTP, classificação da tentativa e transições de estado.
- Novo entry point do worker, análogo a `src/server.ts`: bootstrap, `PrismaClient` próprio, loop de 2 s e shutdown do processo.

## 6. Modelo de dados lógico

Os nomes abaixo são **PROPOSTA — REQUER APROVAÇÃO**; representam informação necessária, não um schema Prisma fechado.

| ID | Entidade | Campos conceituais | Invariantes/índices |
| --- | --- | --- | --- |
| <a id="fdd-data-01"></a>FDD-DATA-01 | `WebhookEndpoint` | `id` UUID, `customerId`, `url`, `active`, `deletedAt`, filtros de status, ciphertext/IV/tag das secrets atual/anterior, `previousSecretExpiresAt`, timestamps | Secret exclusiva; URL HTTPS; busca por cliente/ativo/filtro; remoção lógica conforme FDD-PROP-06; formato físico sujeito a FDD-PROP-04 |
| <a id="fdd-data-02"></a>FDD-DATA-02 | `WebhookOutbox` | `id`/`eventId` UUID, `webhookId`, `orderId`, `orderSequence`, payload JSON, estado, `cancelReason`, `attemptNumber`, `nextAttemptAt`, `lockedAt`, `lockedBy`, `claimToken`, timestamps | Snapshot imutável; índices de elegibilidade e `(webhookId, orderId, orderSequence)`; claim, gating e finalização sujeitos a FDD-PROP-02/05/10 |
| <a id="fdd-data-03"></a>FDD-DATA-03 | `WebhookDelivery` | UUID, `outboxId`, `eventId`, `webhookId`, tentativa, sucesso/falha, payload, status/resposta remota, latência e timestamp | Unicidade `(outboxId, tentativa)`; suporta as 100 entregas mais recentes por endpoint ([09:34] Marcos (TRANSCRICAO.md:L202)); inserção atômica conforme FDD-PROP-10 |
| <a id="fdd-data-04"></a>FDD-DATA-04 | `WebhookDeadLetter` | UUID, `outboxId`, referência do evento/endpoint, payload, motivo, contagem, timestamp, `replayedAt`, `replayedByUserId`, `replayOutboxId` | Unicidade sobre a tentativa terminal da outbox; tabela separada para diagnóstico/replay ([09:18] Diego (TRANSCRICAO.md:L110)); vínculo de replay conforme FDD-PROP-07/10 |
| <a id="fdd-data-05"></a>FDD-DATA-05 | `WebhookReplayAudit` | UUID, dead-letter, evento recolocado, `userId`, timestamp | Identifica quem executou o replay ([09:36] Sofia (TRANSCRICAO.md:L210)) |

FDD-PROP-05 requer ainda uma coluna aditiva `Order.webhookSequence` inteira, inicializada em zero e incrementada sob lock em cada transição de status. Ela é o contador de origem copiado para `WebhookOutbox.orderSequence`; não representa uma nova entidade nem muda o contrato público de pedidos.

Estados mínimos propostos para outbox: `PENDING`, `PROCESSING`, `DELIVERED`, `FAILED` e `CANCELLED`. Os quatro primeiros se alinham à enumeração verbal de [09:08] Diego (TRANSCRICAO.md:L56); `CANCELLED` é acrescentado por FDD-PROP-06. Um retry pode retornar o item a `PENDING` com horário futuro; após esgotamento, o registro correspondente é persistido na DLQ. FDD-PROP-02 fornece claim/lease e FDD-PROP-05 define o gating por destino/pedido, ambos sujeitos a aprovação.

## 7. Fluxos detalhados

### 7.1 <a id="fdd-flow-01"></a>[FDD-FLOW-01] Criação do evento na outbox

1. `OrderService.changeStatus` inicia a transação Prisma já existente.
2. Carrega o pedido e valida `from_status -> to_status` por `src/modules/orders/order.status.ts`.
3. Bloqueia a linha do pedido, incrementa `Order.webhookSequence` e usa o novo `orderSequence` para esta transição, conforme FDD-PROP-05.
4. Aplica débito ou reposição de estoque quando a máquina de estados exigir.
5. Atualiza `Order.status` e cria `OrderStatusHistory`.
6. **Regra proposta para o campo ativo:** consulta endpoints ativos do `customer_id` cujo filtro contenha `to_status`.
7. Se nenhum endpoint corresponder, não cria outbox ([09:33] Marcos (TRANSCRICAO.md:L194); [09:34] Bruno (TRANSCRICAO.md:L198)); o incremento da sequência permanece associado à transição, e lacunas não alteram ordenação.
8. Para cada endpoint elegível, gera UUID, copia o mesmo `orderSequence` e serializa o snapshot no mesmo instante. **Recomendação:** uma linha por endpoint, pois assinatura, tentativas e `X-Webhook-Id` são específicos do destino.
9. **Integração proposta do limite:** mede o corpo serializado em bytes. Se exceder 64 KB, lança erro de domínio e faz rollback; não trunca ([09:24] Larissa (TRANSCRICAO.md:L144)).
10. Insere a outbox usando o `tx` existente. Qualquer falha aborta pedido, histórico, estoque, incremento da sequência e todos os eventos ([09:40] Bruno (TRANSCRICAO.md:L238); [09:41] Diego (TRANSCRICAO.md:L244)).
11. Após commit, devolve a resposta normal da mudança de status; nenhum HTTP externo ocorre na request.

### 7.2 <a id="fdd-flow-02"></a>[FDD-FLOW-02] Polling e processamento

1. O entry point separado do worker cria seu próprio `PrismaClient`, conecta e inicia o loop.
2. A cada 2 segundos, consulta uma janela de até 20 pendências cujo horário venceu. A ordem global de varredura usa `(createdAt, id)`, enquanto a elegibilidade exclui predecessor não terminal de menor `orderSequence` para o mesmo (`webhookId`, `orderId`), conforme FDD-PROP-05.
3. Para cada candidato, imediatamente antes do HTTP, o worker revalida endpoint e gating e reivindica exatamente um registro em transação curta com `FOR UPDATE SKIP LOCKED`, `claimToken` e lease de 60 segundos, conforme FDD-PROP-02/10. A janela não é marcada `PROCESSING` em lote.
4. Para cada item, usa exatamente o snapshot persistido, sem recarregar o pedido.
5. Valida HTTPS e tamanho, produz `X-Timestamp`, calcula a assinatura conforme o contrato criptográfico aprovado e envia com timeout de 10 segundos.
6. Finaliza pelo `claimToken` em uma única transação FDD-PROP-10: insere `WebhookDelivery` e atualiza a outbox para `DELIVERED`, `PENDING` ou `FAILED`; no último caso, inclui também a DLQ.
7. Em falha, classificação e agendamento seguem o fluxo 7.3, mas suas gravações são confirmadas juntas pela transação anterior.
8. **Extensão proposta de shutdown:** em `SIGINT`/`SIGTERM`, para novos claims, aguarda a tentativa atual por no máximo o timeout de 10 segundos e desconecta o Prisma, seguindo o padrão de `src/server.ts`. Se o processo morrer, o lease devolve o item a `PENDING` e invalida o token; nenhum histórico parcial é criado.

### 7.3 <a id="fdd-flow-03"></a>[FDD-FLOW-03] Retry e DLQ

1. Classifica o resultado em sucesso, falha retentável ou falha permanente pelo default FDD-PROP-01, sujeito à aprovação.
2. Usa, na ordem, os marcos +1 min, +5 min, +30 min, +2 h e +12 h. A cardinalidade permanece ambígua: cinco envios totais não comportam todos os cinco intervalos após a primeira falha, enquanto uma tentativa inicial mais cinco retries contraria “total 5 tentativas”.
3. Mantém o mesmo `event_id`, payload e `webhook_id` em todas as tentativas.
4. Registra a tentativa e sua transição de estado na mesma transação, conforme FDD-PROP-10.
5. Depois de esgotar a cardinalidade aprovada — ou imediatamente após falha permanente, conforme FDD-PROP-01 — a mesma transação insere `WebhookDelivery`, cria `WebhookDeadLetter` e marca a outbox `FAILED`.

Nota de revisão: [09:17] Diego (TRANSCRICAO.md:L104) fornece cinco intervalos, enquanto [09:17] Larissa (TRANSCRICAO.md:L108) e [09:48] Larissa (TRANSCRICAO.md:L282) registram cinco tentativas totais. Este FDD não escolhe uma interpretação; os revisores devem resolver a divergência antes do código.

### 7.4 <a id="fdd-flow-04"></a>[FDD-FLOW-04] Replay manual

1. A rota aplica `authenticate` e `requireRole('ADMIN')`.
2. Valida o UUID e localiza a entrada da DLQ.
3. Recoloca o evento como pendente, preservando `event_id`, snapshot e destino para manter a deduplicação.
4. Registra `userId`, dead-letter, nova pendência e timestamp de forma auditável.
5. Retorna aceitação; o worker fará a entrega de forma assíncrona.
6. Conforme FDD-PROP-07, a primeira chamada retorna `202`; repetição para a mesma entrada retorna `200` e a mesma outbox de replay, sem duplicar trabalho ou auditoria.

### 7.5 <a id="fdd-flow-05"></a>[FDD-FLOW-05] Rotação da secret

1. Usuário autenticado solicita rotação para um endpoint ao qual tenha acesso conforme a política futura de ownership.
2. O OMS gera uma nova secret e a retorna nessa operação.
3. A secret anterior recebe expiração 24 horas à frente e permanece válida em paralelo. Pelo default FDD-PROP-04, o worker emite assinaturas calculadas pelas duas secrets durante essa janela.
4. Depois do período, a anterior deixa de ser válida ([09:21] Sofia (TRANSCRICAO.md:L130)).
5. **Proteção proposta em FDD-PROP-04:** logs, histórico e respostas posteriores nunca exibem o valor das secrets.

O armazenamento AES-256-GCM, a chave de ambiente e as assinaturas múltiplas estão definidos em FDD-PROP-04 como proposta completa e precisam de aprovação de Segurança.

## 8. Contratos públicos

### 8.1 Convenções

Todos os endpoints de gestão usam JWT Bearer e o prefixo atual `/api/v1`. O envelope de erro segue `{ "error": { "code", "message", "details?" } }`. O replay exige `ADMIN`; na reunião, o CRUD de configuração ficou liberado para qualquer role autenticada; estender essa regra a deliveries e rotação é proposta de integração ([09:32] Larissa (TRANSCRICAO.md:L190); [09:37] Sofia (TRANSCRICAO.md:L216); [09:36] Sofia (TRANSCRICAO.md:L210)).

Os paths de CRUD e rotação em 8.2–8.6 são **PROPOSTA — REQUER APROVAÇÃO**, especialmente a escolha de `customerId` no path. Dois paths relativos foram verbalizados: `GET /webhooks/:id/deliveries` em [09:34] Marcos (TRANSCRICAO.md:L202) e `POST /admin/webhooks/dead-letter/:id/replay` em [09:18] Diego (TRANSCRICAO.md:L114). O prefixo `/api/v1` usado em 8.7–8.8 decorre de `src/app.ts` e `src/routes/index.ts`, não da transcrição.

### 8.2 <a id="fdd-api-01"></a>[FDD-API-01] Criar webhook

`POST /api/v1/customers/:customerId/webhooks`

```http
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "url": "https://cliente.example/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

Resposta proposta — `201 Created`:

```json
{
  "id": "816d4a66-ec43-4f8a-98a0-5ca906f25fb3",
  "customerId": "ff1d2d73-f916-42ca-a064-c2ee72b56da6",
  "url": "https://cliente.example/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_EXEMPLO_EXIBIDO_APENAS_NESTA_RESPOSTA"
}
```

Status: `201` criado; `400` URL/filtro inválido; `401` sem autenticação; `404` cliente inexistente; `409` conflito a definir. A secret é gerada pelo OMS e devolvida na criação ([09:31] Marcos (TRANSCRICAO.md:L184)); exibição única é recomendação de segurança, não decisão registrada.

### 8.3 <a id="fdd-api-02"></a>[FDD-API-02] Listar webhooks do cliente

`GET /api/v1/customers/:customerId/webhooks`

```http
Authorization: Bearer <jwt>
```

Resposta proposta — `200 OK`:

```json
{
  "data": [
    {
      "id": "816d4a66-ec43-4f8a-98a0-5ca906f25fb3",
      "customerId": "ff1d2d73-f916-42ca-a064-c2ee72b56da6",
      "url": "https://cliente.example/webhooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ]
}
```

Status: `200`; `400` parâmetro inválido; `401`; `404` cliente inexistente. A resposta nunca inclui secret.

### 8.4 <a id="fdd-api-03"></a>[FDD-API-03] Editar webhook

`PATCH /api/v1/customers/:customerId/webhooks/:webhookId`

```http
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "url": "https://novo.example/webhooks/orders",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Resposta proposta — `200 OK`:

```json
{
  "id": "816d4a66-ec43-4f8a-98a0-5ca906f25fb3",
  "customerId": "ff1d2d73-f916-42ca-a064-c2ee72b56da6",
  "url": "https://novo.example/webhooks/orders",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

Status: `200`; `400` URL/filtro inválido; `401`; `404` recurso não encontrado. Alterar configuração não reescreve snapshots já persistidos. Se `active` mudar para `false`, aplicam-se cancelamento e revalidação de FDD-PROP-06.

### 8.5 <a id="fdd-api-04"></a>[FDD-API-04] Remover webhook

`DELETE /api/v1/customers/:customerId/webhooks/:webhookId`

```http
Authorization: Bearer <jwt>
```

Resposta proposta — `204 No Content`, sem body. Status adicionais: `400`, `401`, `404`. Conforme FDD-PROP-06, a remoção é lógica, interrompe novos enqueues e cancela pendências na mesma transação, preservando histórico e auditoria.

### 8.6 <a id="fdd-api-05"></a>[FDD-API-05] Rotacionar secret

`POST /api/v1/customers/:customerId/webhooks/:webhookId/rotate-secret`

```http
Authorization: Bearer <jwt>
Content-Type: application/json

{}
```

Resposta proposta — `200 OK`:

```json
{
  "webhookId": "816d4a66-ec43-4f8a-98a0-5ca906f25fb3",
  "secret": "whsec_NOVA_SECRET_EXEMPLO",
  "previousSecretValidForHours": 24
}
```

Status: `200`; `400`; `401`; `404`; `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS` quando a secret anterior ainda estiver no grace period. A resposta `409` inclui `previousSecretExpiresAt` em `details`; solicitações concorrentes são serializadas pelo lock do endpoint, conforme FDD-PROP-04. O formato da secret, a exposição única e essa política precisam de aprovação de Segurança.

### 8.7 <a id="fdd-api-06"></a>[FDD-API-06] Consultar entregas recentes

`GET /api/v1/webhooks/:webhookId/deliveries`

```http
Authorization: Bearer <jwt>
```

Resposta proposta — `200 OK`:

```json
{
  "data": [
    {
      "eventId": "78d7a3cc-4a87-481c-9cd4-cf3ec3428072",
      "success": false,
      "attempt": 2,
      "payload": { "event_type": "order.status_changed" },
      "response": { "statusCode": 503, "body": "temporarily unavailable" },
      "latencyMs": 318,
      "attemptedAt": "2026-09-05T12:00:00.000Z"
    }
  ],
  "limit": 100
}
```

Status: `200`; `400`; `401`; `404`. Retorna no máximo as 100 tentativas mais recentes com sucesso/falha, payload, resposta e latência ([09:34] Marcos (TRANSCRICAO.md:L202)). Limites de armazenamento do corpo da resposta permanecem abertos.

### 8.8 <a id="fdd-api-07"></a>[FDD-API-07] Reprocessar DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay`

```http
Authorization: Bearer <jwt-admin>
Content-Type: application/json

{}
```

Resposta proposta — `202 Accepted`:

```json
{
  "deadLetterId": "3925df20-c2d9-4ab8-8af6-e25d254dd295",
  "eventId": "78d7a3cc-4a87-481c-9cd4-cf3ec3428072",
  "replayOutboxId": "47d7875c-0c68-444f-910c-a680a26bdc6e",
  "status": "PENDING"
}
```

Status: `202` no primeiro replay; `200` em repetição idempotente; `400`; `401`; `403` se não `ADMIN`; `404`. Ambas as respostas de sucesso retornam o mesmo `replayOutboxId`; apenas a primeira cria pendência e registra o usuário executor ([09:36] Sofia (TRANSCRICAO.md:L210)), conforme FDD-PROP-07.

### 8.9 Contrato outbound `order.status_changed`

Corpo persistido e enviado:

```json
{
  "event_id": "78d7a3cc-4a87-481c-9cd4-cf3ec3428072",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-05T11:59:58.120Z",
  "order_id": "198c11a9-eb55-480f-bd60-e9952d1c0b76",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "ff1d2d73-f916-42ca-a064-c2ee72b56da6",
  "total_cents": 15990
}
```

Campos são os listados em [09:43] Diego (TRANSCRICAO.md:L256); itens não são enviados. O cliente pode consultar `GET /orders/:id` para detalhes ([09:43] Diego (TRANSCRICAO.md:L256)).

Headers (nomes confirmados; Unix time, `v1=<hex>` e assinaturas múltiplas são formatos propostos em FDD-PROP-03/04):

```http
Content-Type: application/json
X-Event-Id: 78d7a3cc-4a87-481c-9cd4-cf3ec3428072
X-Webhook-Id: 816d4a66-ec43-4f8a-98a0-5ca906f25fb3
X-Timestamp: 1788955198
X-Signature: v1=<hex-hmac-current>,v1=<hex-hmac-previous-during-grace-period>
```

O contrato confirmado assina somente `rawBody`. FDD-PROP-03 oferece `<X-Timestamp>.<rawBody>` como **alteração proposta**, não como interpretação automática da reunião; adotá-la exige aprovação e versionamento/comunicação aos consumidores. FDD-PROP-01 classifica a resposta e também requer aprovação. O consumidor deve tratar `event_id` como chave de deduplicação. O corpo é medido já serializado e permanece idêntico ao usado no HMAC.

## 9. Matriz de erros previstos

Os códigos abaixo são **propostas de contrato** no prefixo decidido `WEBHOOK_*` ([09:28] Bruno (TRANSCRICAO.md:L170); [09:29] Larissa (TRANSCRICAO.md:L172)). `UNAUTHORIZED`, `FORBIDDEN` e `VALIDATION_ERROR` continuam sendo códigos transversais existentes.

| ID | Código | HTTP/contexto | Quando ocorre | Origem do comportamento |
| --- | --- | --- | --- | --- |
| <a id="fdd-err-01"></a>FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | Configuração não encontrada no escopo informado | Exemplo verbal [09:28] Bruno (TRANSCRICAO.md:L170); padrão `NotFoundError` |
| <a id="fdd-err-02"></a>FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | URL ausente, inválida ou não HTTPS | [09:23] Sofia (TRANSCRICAO.md:L138); exemplo [09:28] Bruno (TRANSCRICAO.md:L170) |
| <a id="fdd-err-03"></a>FDD-ERR-03 | `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige material de secret não o possui | Exemplo [09:28] Bruno (TRANSCRICAO.md:L170); fluxo exato em aberto |
| <a id="fdd-err-04"></a>FDD-ERR-04 | `WEBHOOK_INVALID_STATUS_FILTER` | 400 | Filtro vazio ou com status fora de `OrderStatus` | Filtros [09:33] Marcos (TRANSCRICAO.md:L194); [09:34] Bruno (TRANSCRICAO.md:L198); validação proposta |
| <a id="fdd-err-05"></a>FDD-ERR-05 | `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Corpo serializado excede 64 KB | [09:24] Larissa (TRANSCRICAO.md:L144) |
| <a id="fdd-err-06"></a>FDD-ERR-06 | `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | Histórico solicitado não existe/no escopo | Contrato proposto de deliveries |
| <a id="fdd-err-07"></a>FDD-ERR-07 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Entrada da DLQ não existe | Replay [09:18] Diego (TRANSCRICAO.md:L114) |
| <a id="fdd-err-08"></a>FDD-ERR-08 | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | Nova rotação solicitada antes de expirar a secret anterior | Grace period em [09:21] Sofia (TRANSCRICAO.md:L130); bloqueio sucessivo proposto em FDD-PROP-04 |
| <a id="fdd-err-09"></a>FDD-ERR-09 | `WEBHOOK_DELIVERY_FAILED` | worker | Tentativa externa falha e precisa ser registrada/classificada | Retry [09:15] Diego (TRANSCRICAO.md:L92); [09:18] Diego (TRANSCRICAO.md:L110) |

Erros `AppError` devem manter o envelope atual. FDD-PROP-08 define a tradução necessária para que issues Zod de `url` e `statuses` se tornem FDD-ERR-02/04; sem o mapper do domínio, o comportamento permanece `VALIDATION_ERROR`. Falhas não tratadas serão logadas e retornarão `INTERNAL_SERVER_ERROR` pelo middleware existente; secrets, payload sensível e autorização não devem aparecer no log.

## 10. Resiliência

Síntese dos fluxos 7.1–7.5: agenda persistida, shutdown, concorrência, gating, cancelamento e idempotência de replay são extensões propostas com as premissas indicadas nesses fluxos.

### 10.1 Timeouts e isolamento

- Timeout HTTP fixo de 10 segundos ([09:42] Diego (TRANSCRICAO.md:L250)).
- Nenhuma chamada externa na transação do pedido.
- Processo do worker independente da API, com shutdown explícito.
- Uma falha de endpoint não deve bloquear permanentemente os demais; a política de concorrência continua aberta.

### 10.2 Retry e backoff

- Agenda persistida, não baseada somente em timers em memória.
- Cinco marcos temporais na ordem decidida; a quantidade total de envios continua bloqueante.
- Mesmo `event_id` e mesmo snapshot em todas as tentativas.
- Classificação HTTP/rede segue o default FDD-PROP-01 quando aprovado.

### 10.3 DLQ e fallback

- DLQ persistida separadamente, com payload, causa e timestamp.
- Replay somente por `ADMIN`, auditado, assíncrono e idempotente por entrada de DLQ conforme FDD-PROP-07.
- Não há email, dashboard nem fallback alternativo nesta fase.

### 10.4 Ordenação e duplicatas

- O single-worker e a ordenação por `createdAt` não bastam para preservar a ordem quando um evento aguarda retry: sem gating, o posterior pode ultrapassá-lo.
- FDD-PROP-05 usa `orderSequence`, não `createdAt`, para ordenar transições do mesmo pedido, inclusive quando timestamps coincidem.
- O gating bloqueia somente o evento posterior do mesmo (`webhookId`, `orderId`); endpoint diferente e outros pedidos continuam. `DELIVERED`, DLQ e `CANCELLED` liberam esse par destino/pedido.
- Não há ordering global. Replay posterior da DLQ pode chegar depois de eventos mais novos do mesmo pedido e essa limitação integra o contrato ([09:13] Larissa (TRANSCRICAO.md:L86)).
- At-least-once admite duplicatas; o consumidor deduplica por `X-Event-Id`.
- Múltiplos workers exigirão novo desenho de particionamento/locking.

## 11. Observabilidade

### 11.1 Logs

**Confirmado:** reutilizar Pino ([09:29] Larissa (TRANSCRICAO.md:L172); [09:29] Bruno (TRANSCRICAO.md:L174); [09:30] Larissa (TRANSCRICAO.md:L180)). **Extensão proposta:** cada log do worker deve ser estruturado e correlacionável por `eventId`, `webhookId`, `orderId`, tentativa e estado. Eventos mínimos recomendados: claim, início/fim da tentativa, timeout, retry agendado, envio à DLQ, replay e shutdown.

O logger atual redige tokens e senhas (`src/shared/logger/index.ts`); a implementação futura deve acrescentar todos os nomes usados para secret/assinatura. Não registrar a secret nem headers de autorização. Definir se payload e response body exigem redação/truncamento é questão de segurança e retenção.

### 11.2 Métricas

Os nomes são **recomendações**, pois a reunião não fechou métricas/SLOs:

- `webhook_outbox_pending` e idade do evento pendente mais antigo;
- total de deliveries por resultado/status remoto;
- latência de entrega e latência fim a fim desde o timestamp do evento;
- retries agendados e esgotados;
- tamanho da DLQ e total de replays;
- duração e falhas do enqueue dentro de `changeStatus`.

A meta de produto inferior a 10 segundos é a meta de latência confirmada ([09:02] Marcos (TRANSCRICAO.md:L22)); percentil, janela e SLO operacional precisam de aprovação.

### 11.3 Tracing

Premissas: `src/middlewares/request-logger.middleware.ts` cria/propaga `requestId` na API; `package.json` não declara biblioteca dedicada de tracing. Isso não prova ausência de todo SLO organizacional.

Como extensão proposta da correlação HTTP existente, o FDD recomenda rastreabilidade de API → outbox → worker → tentativa por IDs correlacionados. Não existe dependência de tracing no `package.json`; adotar biblioteca ou backend novo requer decisão própria. Até lá, logs estruturados devem carregar `requestId` quando originados na API e sempre `eventId` no processamento. Spans distribuídos são recomendados, mas não decididos.

## 12. Integração com o sistema existente

| ID | Caminho real | Integração futura |
| --- | --- | --- |
| <a id="fdd-int-01"></a>FDD-INT-01 | `src/modules/orders/order.service.ts` | Chamar `publishWebhookEvent(tx, ...)` dentro do callback de `$transaction`, depois de validar/aplicar a mudança e antes do commit; falha faz rollback integral. |
| <a id="fdd-int-02"></a>FDD-INT-02 | `src/modules/orders/order.status.ts` | Usar `OrderStatus` e a transição efetivamente aprovada para compor `from_status`/`to_status`; não duplicar a máquina de estados no módulo de webhooks. |
| <a id="fdd-int-03"></a>FDD-INT-03 | `src/modules/orders/order.schemas.ts` | Manter o contrato de mudança de status existente; o evento decorre do resultado interno, não de um novo campo fornecido pelo cliente. |
| <a id="fdd-int-04"></a>FDD-INT-04 | `src/app.ts` | Compor repository/service/controller do módulo futuro com o `PrismaClient` da API, seguindo a composição manual atual. |
| <a id="fdd-int-05"></a>FDD-INT-05 | `src/routes/index.ts` | Montar rotas propostas sob `/api/v1`, após aprovação dos paths e de `customer_id`. |
| <a id="fdd-int-06"></a>FDD-INT-06 | `src/middlewares/auth.middleware.ts` | Aplicar `authenticate` ao CRUD e deliveries; aplicar `requireRole('ADMIN')` ao replay. |
| <a id="fdd-int-07"></a>FDD-INT-07 | `src/middlewares/validate.middleware.ts` | Preservar o default `VALIDATION_ERROR` e acrescentar o mapper opcional de FDD-PROP-08 para URL/filtro nas rotas de webhook. |
| <a id="fdd-int-08"></a>FDD-INT-08 | `src/shared/errors/app-error.ts` | Criar erros do domínio compatíveis com `AppError`; o error middleware existente usará seus `errorCode` específicos. |
| <a id="fdd-int-09"></a>FDD-INT-09 | `src/middlewares/error.middleware.ts` | Preservar o envelope JSON e o tratamento centralizado de erros. |
| <a id="fdd-int-10"></a>FDD-INT-10 | `src/shared/logger/index.ts` | Reutilizar Pino e ampliar redação para secrets/assinaturas; correlacionar logs do worker. |
| <a id="fdd-int-11"></a>FDD-INT-11 | `src/middlewares/request-logger.middleware.ts` | Propagar `requestId` ao enqueue e à auditoria quando disponível. |
| <a id="fdd-int-12"></a>FDD-INT-12 | `src/config/database.ts` | Criar um `PrismaClient` próprio para o processo worker usando a fábrica atual como referência. |
| <a id="fdd-int-13"></a>FDD-INT-13 | `src/server.ts` | Reutilizar o padrão de bootstrap e shutdown em um entry point separado. |
| <a id="fdd-int-14"></a>FDD-INT-14 | `prisma/schema.prisma` | Adicionar futuramente modelos, relações, enums e índices por migração versionada, mantendo UUID como padrão. |
| <a id="fdd-int-15"></a>FDD-INT-15 | `tests/orders.test.ts` | Estender os cenários de transação e mudança de status com os efeitos da outbox. |
| <a id="fdd-int-16"></a>FDD-INT-16 | `tests/setup.ts` | Limpar as futuras tabelas em ordem compatível com FKs, somente em banco isolado selecionado pelo executor; o setup atual não impõe esse isolamento. |
| <a id="fdd-int-17"></a>FDD-INT-17 | `src/config/env.ts` | Estender `envSchema`/`loadEnv`, que hoje validam Zod sem chave de webhook, para validar a chave-mestra proposta; essa configuração ainda não existe. |

## 13. Dependências e compatibilidade

- <a id="fdd-dep-01"></a>Node.js >= 20, TypeScript 5.6, ESM, Express 4.21, Prisma 5.22, Zod 3, Pino 9 e Vitest/Supertest constam de `package.json`; o target ES2022 está em `tsconfig.json`, e MySQL 8.0 em `docker-compose.yml`.
- <a id="fdd-dep-02"></a>A API atual limita JSON inbound a 1 MB (`src/app.ts`); o limite outbound específico é 64 KB.
- <a id="fdd-dep-03"></a>O worker usa a mesma `DATABASE_URL`, mas não compartilha a instância em memória da API.
- <a id="fdd-dep-04"></a>Não há dependência HTTP ou tracing dedicada no projeto. A escolha de cliente HTTP e eventual tracing deve ser avaliada antes de alterar `package.json`; APIs nativas do Node 20 podem ser consideradas, sem decisão neste FDD.
- <a id="fdd-dep-05"></a>A feature não altera o ciclo de estados do pedido nem os contratos atuais de status; adiciona um efeito transacional interno e novas rotas.

## 14. Estratégia de testes e validação

Plano proposto de verificação dos requisitos e defaults. Os cenários que dependem de FDD-PROP só se tornam critérios obrigatórios após aprovação; não representam testes já implementados.

### 14.1 Unitários

- Serialização determinística do snapshot e cálculo de tamanho em bytes.
- Filtro por `to_status` e endpoint ativo.
- HMAC com vetores separados para o baseline `rawBody` e, somente se aprovada, a extensão `timestamp.rawBody`, inclusive rotação durante/depois de 24 h.
- Tabela de backoff e preservação de `event_id`/payload.
- Validação Zod de HTTPS, UUIDs, filtros e payloads.
- Mapper de Zod: `url` prevalece sobre `statuses`, ambos preservam todas as issues em `details`, e outros campos mantêm `VALIDATION_ERROR`.
- Classificação de respostas após a decisão pendente.

### 14.2 Integração com MySQL isolado

- Commit da mudança gera histórico e outbox; falha de outbox reverte pedido, estoque e histórico.
- Sem endpoint elegível não gera outbox.
- Claim não duplica processamento conforme o mecanismo aprovado; crash permite recuperação.
- Um lote de 20 candidatos mantém no máximo um claim em curso; nenhum item aguardando localmente expira como `PROCESSING`.
- Evento em backoff impede claim de sequência posterior somente no mesmo destino/pedido; outro endpoint saudável continua. Transições com o mesmo `createdAt` seguem `orderSequence`; após DLQ, o replay pode chegar fora de ordem.
- Sucesso, retry, cancelamento e DLQ confirmam `WebhookDelivery` e estado na mesma transação; falha da conclusão deixa o claim recuperável sem histórico parcial.
- Finalizador com `claimToken` expirado não grava delivery nem altera estado depois que outro claim assumiu o item.
- Injetar falha entre cada operação da conclusão e comprovar rollback integral; depois expirar o lease e verificar reenvio com o mesmo `eventId` e nova tentativa.
- Tentativas atualizam estado e deliveries; esgotamento persiste DLQ.
- Replay exige `ADMIN`, preserva o evento, grava uma auditoria e é idempotente sob repetição/concorrência.
- Desativar/remover cancela pendências atomicamente; reativar não as ressuscita e remoção preserva histórico.
- CRUD, rotação e consulta das 100 entregas respeitam autenticação/validação.
- Duas rotações concorrentes ou dentro de 24 horas produzem uma rotação e um `409`, sem substituir a secret anterior; após a expiração, nova rotação é permitida.
- URL e filtro inválidos retornam os códigos `WEBHOOK_*` previstos; demais falhas Zod continuam usando `VALIDATION_ERROR`.

`tests/setup.ts` apaga tabelas conhecidas antes de cada teste usando o Prisma configurado por `DATABASE_URL`; não verifica que o banco é isolado. O executor precisa garantir esse isolamento. As tabelas futuras devem ser limpas em ordem compatível com suas FKs. Nunca executar contra banco compartilhado ou de produção.

### 14.3 Ponta a ponta e segurança

- Servidor HTTP controlado recebe corpo e cinco headers; verificar timeout e duplicata.
- Executar o cenário FDD-PROP-09 de 100 transições a até 1 evento/s, endpoint `2xx` em até 500 ms e fila inicial vazia; medir commit → primeiro byte e exigir todas as amostras abaixo de 10 segundos.
- Colocar uma chamada de 10 segundos antes de destino saudável e registrar a violação esperada do worker sequencial, sem apresentar o cenário saturado como atendimento da meta.
- Exercitar os cinco marcos com relógio controlado, conforme a cardinalidade aprovada, e o envio à DLQ.
- Confirmar que logs e respostas não vazam secrets.
- A reunião reserva ao menos dois dias úteis para Sofia revisar o código de segurança, destacando HMAC e geração de secret; incluir armazenamento, rotação, canonicalização, comparação e TLS é detalhamento proposto do checklist ([09:46] Sofia (TRANSCRICAO.md:L274)).

## 15. Migração, deploy e rollback

Esta seção é **recomendação para revisão operacional**; a reunião não fechou a estratégia.

1. Aprovar questões de dados, locking, HTTP e segurança.
2. Criar migração aditiva dos modelos/índices, sem remover estruturas existentes.
3. Implantar API com CRUD e capacidade de enqueue somente após o schema estar disponível.
4. Implantar exatamente uma réplica do worker, inicialmente sem acumular concorrência horizontal.
5. Validar ponta a ponta e observabilidade antes de habilitar clientes.
6. Habilitar endpoints de forma controlada e observar atraso, retries e DLQ.

Rollback recomendado: impedir novos cadastros/enqueues conforme mecanismo a definir, parar o worker de forma graciosa e preservar outbox/DLQ para recuperação; não remover tabelas automaticamente. O time ainda precisa decidir mecanismo de habilitação, compatibilidade entre versões e autoridade para replay durante rollback.

## 16. Critérios de aceite técnicos

### 16.1 Baseline confirmado, extensões e gates documentais

- <a id="fdd-ac-01"></a>[FDD-AC-01] Uma transição com endpoint elegível faz commit do pedido, histórico, estoque e snapshot; falha no snapshot reverte tudo.
- <a id="fdd-ac-02"></a>[FDD-AC-02] Uma transição sem endpoint inscrito não gera outbox. Exigir também endpoint ativo é regra de elegibilidade proposta.
- <a id="fdd-ac-03"></a>[FDD-AC-03] O worker roda fora da API, com client Prisma próprio, polling de 2 s e single-worker documentado.
- <a id="fdd-ac-04"></a>[FDD-AC-04] Entregas usam HTTPS, timeout de 10 s, limite de 64 KB e todos os headers confirmados.
- <a id="fdd-ac-05"></a>[FDD-AC-05] O corpo contém o snapshot confirmado, sem itens, e permanece idêntico nos retries.
- <a id="fdd-ac-06"></a>[FDD-AC-06] Os marcos 1 min/5 min/30 min/2 h/12 h respeitam a cardinalidade aprovada; o esgotamento cria DLQ separada.
- <a id="fdd-ac-07"></a>[FDD-AC-07] O mesmo UUID aparece no body e em `X-Event-Id` em tentativa, retry e replay.
- <a id="fdd-ac-08"></a>[FDD-AC-08] O baseline usa HMAC-SHA256 da `rawBody` com secret única; rotação mantém a anterior por 24 h. A extensão com timestamp só integra o aceite após aprovação/versionamento.
- <a id="fdd-ac-09"></a>[FDD-AC-09] CRUD exige autenticação; autenticar deliveries é extensão proposta; replay exige `ADMIN` e auditoria.
- <a id="fdd-ac-10"></a>[FDD-AC-10] As 100 entregas mais recentes expõem resultado, payload, resposta e latência. A redação adicional para não expor secrets é controle proposto.
- <a id="fdd-ac-11"></a>[FDD-AC-11] Erros de domínio seguem `WEBHOOK_*` e o envelope centralizado.
- <a id="fdd-ac-12"></a>[FDD-AC-12] **Critério proposto de observabilidade:** métricas, logs e correlação permitem identificar backlog, tentativas, DLQ e latência fim a fim.
- <a id="fdd-ac-13"></a>[FDD-AC-13] **Organização proposta de gates:** as decisões de 18.1 estão aprovadas antes do comportamento afetado, as de 18.2 antes do deploy e as evoluções de 18.3 não bloqueiam a primeira fase.

### 16.2 Critérios condicionais aos defaults propostos

Os itens seguintes são gates de aceite **somente se** as propostas citadas forem aprovadas. Enquanto isso, são cenários para decisão/revisão e não requisitos confirmados da reunião.

- <a id="fdd-ac-14"></a>[FDD-AC-14] Se FDD-PROP-05 for aprovada, um retry bloqueia sequência posterior apenas no mesmo (`webhookId`, `orderId`); `orderSequence` desempata timestamps iguais, outros destinos continuam e replay fora de ordem permanece documentado.
- <a id="fdd-ac-15"></a>[FDD-AC-15] Se FDD-PROP-06 for aprovada, desativar/remover cancela pendências na mesma transação, evita novos enqueues/retries e preserva histórico; reativar não ressuscita eventos.
- <a id="fdd-ac-16"></a>[FDD-AC-16] Se FDD-PROP-07 for aprovada, repetir ou concorrer o replay da mesma entrada de DLQ retorna a mesma outbox sem criar outra pendência/auditoria.
- <a id="fdd-ac-17"></a>[FDD-AC-17] Se FDD-PROP-08 for aprovada, falhas Zod em `url` e `statuses` viram FDD-ERR-02/04; schemas existentes e demais campos preservam `VALIDATION_ERROR`.
- <a id="fdd-ac-18"></a>[FDD-AC-18] Se FDD-PROP-04 for aprovada, nova rotação durante o grace period retorna FDD-ERR-08 sem alterar secrets/expiração; depois das 24 horas, pode prosseguir sob lock.
- <a id="fdd-ac-19"></a>[FDD-AC-19] Se FDD-PROP-09 for aprovada, todas as 100 amostras do cenário nominal definido ficam abaixo de 10 segundos; o resultado declara carga, backlog e latência do destino.
- <a id="fdd-ac-20"></a>[FDD-AC-20] Se FDD-PROP-10 for aprovada, cada tentativa confirma delivery + estado + eventual DLQ em uma transação e rejeita finalização com token de claim obsoleto.

## 17. Riscos e mitigação

Análise derivada do desenho: probabilidades/impactos são estimativas qualitativas; controles são propostas, não evidência de funcionamento atual.

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| <a id="fdd-risk-01"></a>Enqueue aumenta duração/contenda da transação | Média | Alto | Consulta/indexação focada, teste de carga e métrica de duração |
| <a id="fdd-risk-02"></a>Worker indisponível acumula eventos | Média | Alto | Backlog/idade observáveis, processo independente e runbook de recuperação |
| <a id="fdd-risk-03"></a>Worker sequencial viola meta após destino lento/backlog | Alta nesse cenário | Alto | Delimitar FDD-PROP-09 e rever concorrência se Produto exigir a meta sob saturação |
| <a id="fdd-risk-04"></a>Claim incorreto duplica ou abandona itens | Média | Alto | Aprovar FDD-PROP-02 e testar crash antes do deploy |
| <a id="fdd-risk-05"></a>Conclusão parcial diverge delivery, outbox e DLQ | Média | Alto | Transação e `claimToken` de FDD-PROP-10, com testes de falha entre operações |
| <a id="fdd-risk-06"></a>Retry permite overtaking no mesmo pedido | Média | Alto | Aprovar gating de FDD-PROP-05 e testar backoff/DLQ/replay |
| <a id="fdd-risk-07"></a>Desativação ainda permite envio indesejado | Média | Alto | Cancelamento transacional e revalidação de FDD-PROP-06 |
| <a id="fdd-risk-08"></a>Cliente processa duplicatas | Alta por contrato | Médio/alto | `X-Event-Id` estável e documentação explícita de idempotência |
| <a id="fdd-risk-09"></a>Secret vaza em banco, log ou response | Média | Crítico | Aprovar AES-256-GCM/chave externa de FDD-PROP-04, redação e revisão de Sofia |
| <a id="fdd-risk-10"></a>Rotação sucessiva encurta o grace period | Média | Alto | Serializar a rotação e rejeitar nova solicitação durante 24 h conforme FDD-PROP-04 |
| <a id="fdd-risk-11"></a>Canonicalização divergente invalida HMAC | Média | Alto | Aprovar FDD-PROP-03 e usar vetores de teste byte a byte |
| <a id="fdd-risk-12"></a>Tabela cresce sem retenção | Média | Médio | Monitorar tamanho; fechar política em evolução posterior |
| <a id="fdd-risk-13"></a>Single-worker não suporta volume | Baixa inicialmente | Alto | Medir backlog/latência; particionamento é evolução planejada, não escopo atual |

## 18. Decisões pendentes por horizonte

**Organização proposta da revisão:** nem toda lacuna impede começar a implementação. Esta separação preserva como fora de escopo os itens adiados na reunião e evita misturar contrato necessário ao código com evolução operacional.

### 18.1 Indispensáveis antes de implementar o comportamento correspondente

1. Resolver se são cinco tentativas totais ou uma tentativa inicial mais cinco retries; essa ambiguidade impede implementar corretamente a máquina de tentativas.
2. Aprovar ou ajustar janela 20, concorrência 1, claim individual, `SKIP LOCKED` e lease de 60 segundos de FDD-PROP-02 antes de implementar o worker.
3. Aprovar ou ajustar a classificação HTTP/rede de FDD-PROP-01 antes de implementar retry/DLQ.
4. Aprovar ou ajustar AES-256-GCM, chave externa, duas assinaturas e bloqueio de rotação sucessiva de FDD-PROP-04 antes de persistir/rotacionar secrets.
5. Aprovar o formato de `X-Signature` e decidir se mantém `HMAC(rawBody)` confirmado ou adota a extensão `<X-Timestamp>.<rawBody>` de FDD-PROP-03 antes de publicar o contrato.
6. Aprovar paths, ownership e posição de `customer_id` antes de estabilizar os contratos de gestão.
7. Aprovar ou ajustar sequência monotônica e gating por (`webhookId`, `orderId`) de FDD-PROP-05 antes de implementar enqueue/seleção.
8. Aprovar ou ajustar cancelamento em delete/disable e replay idempotente de FDD-PROP-06/07 antes dessas operações.
9. Aprovar ou ajustar o mapper Zod de FDD-PROP-08 antes de prometer FDD-ERR-02/04 no contrato público.
10. Aprovar ou ajustar transações de conclusão e `claimToken` de FDD-PROP-10 antes de implementar a finalização do worker.

Esses bloqueios são locais: por exemplo, a migração aditiva e o esqueleto do módulo podem começar enquanto contratos independentes estão em revisão, mas nenhum comportamento afetado deve ser considerado concluído antes da decisão correspondente.

### 18.2 Necessárias antes do deploy, sem bloquear o início do código

- Definir limite/retenção e redação do corpo da resposta remota armazenada em `WebhookDelivery`.
- Aprovar métricas operacionais mínimas, limiares de alerta e critérios de habilitação/rollback do worker.
- Aprovar as condições mensuráveis de FDD-PROP-09 antes de usar a meta inferior a 10 segundos como aceite da primeira fase.
- Produzir o runbook de chave-mestra, recuperação do worker, DLQ e rollback, além de concluir a revisão de Segurança.

### 18.3 Evoluções futuras, fora do escopo da primeira fase

- Política de arquivamento/limpeza e retenção definitiva de outbox entregue, deliveries, DLQ e auditoria; monitorar crescimento é recomendação operacional para esta fase ([09:08] Diego (TRANSCRICAO.md:L56)).
- Backend/biblioteca de tracing distribuído, dashboard visual, rate limiting de saída, múltiplos workers e particionamento.

Esses itens permanecem documentados como dívida/evolução, mas **não** são pré-condição para iniciar nem concluir o código da primeira fase, salvo se uma revisão posterior alterar formalmente o escopo.
