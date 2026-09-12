# PRD — Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e contexto da feature

O OMS permitirá que clientes B2B recebam notificações outbound quando o status de seus pedidos mudar. Hoje, Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` periodicamente para detectar alterações, o que torna a integração lenta e cara ([09:00] Marcos (TRANSCRICAO.md:L18)). Para esses clientes, uma notificação recebida em menos de 10 segundos atende à expectativa de “tempo real” ([09:02] Marcos (TRANSCRICAO.md:L22)).

A primeira fase oferece configuração por API, filtros de status, entrega segura e histórico operacional. O escopo é exclusivamente OMS → cliente ([09:02] Marcos (TRANSCRICAO.md:L26)). A arquitetura e os contratos estão, respectivamente, no [RFC](./RFC.md), nos [ADRs](./adrs/) e no [FDD](./FDD.md).

## 2. Problema e motivação

### Problema

- Clientes precisam consultar pedidos repetidamente mesmo quando nada mudou.
- A detecção por polling atrasa a reação dos sistemas B2B e aumenta o custo de integração.
- O OMS não oferece hoje mecanismo de notificação externa.
- A Atlas informou risco de migrar para um concorrente se a capacidade não for entregue até o fim do trimestre citado na reunião ([09:00] Marcos (TRANSCRICAO.md:L18)).

### Oportunidade

Como oportunidade derivada da demanda de notificação, uma notificação acionada pela mudança de status reduz a dependência do polling e permite que sistemas de logística, faturamento e atendimento reajam ao ciclo do pedido. O mecanismo também cria uma base operacional para observar entregas, recuperar falhas e evoluir integrações futuras sem acoplar a disponibilidade do cliente ao processamento do pedido.

## 3. Público-alvo e cenários de uso

| Público | Necessidade | Cenário principal |
| --- | --- | --- |
| Integradores dos clientes B2B | Reagir à mudança sem consultar continuamente o OMS | Receber `order.status_changed`, validar assinatura e deduplicar pelo evento |
| Usuários autenticados do OMS | Administrar destinos e interesse por status | Cadastrar, listar, editar, remover e rotacionar configuração |
| Administradores do OMS | Recuperar falhas permanentes com controle | Inspecionar evidência e reprocessar uma entrada da DLQ |
| Suporte e operação | Diagnosticar entregas | Consultar as 100 tentativas mais recentes, resposta e latência |
| Engenharia e Segurança | Operar e revisar a solução | Observar backlog/falhas e validar HMAC, secrets, TLS e auditoria |

### Cenários

Exemplos ilustrativos derivados dos requisitos abaixo; a escolha de status pela Nova Cargo não é um pedido registrado da empresa.

1. A Nova Cargo configura um endpoint HTTPS somente para `SHIPPED` e `DELIVERED`; mudanças para outros status não geram entregas para esse endpoint.
2. Quando um pedido se torna `SHIPPED`, o cliente recebe um snapshot enxuto e valida que a chamada veio do OMS.
3. Se o endpoint estiver indisponível, o OMS retenta em janelas crescentes e preserva a falha na DLQ ao esgotá-las.
4. Um `ADMIN` corrige a causa externa e solicita replay manual; a ação fica auditada.
5. Um cliente rotaciona sua secret e tem 24 horas de coexistência para atualizar seu consumidor.

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| <a id="prd-obj-01"></a>PRD-OBJ-01 | Reduzir dependência de polling para mudanças de status | Latência entre mudança confirmada e recebimento da primeira entrega em condições normais | Menor que 10 segundos ([09:02] Marcos (TRANSCRICAO.md:L22)) |
| <a id="prd-obj-02"></a>PRD-OBJ-02 | Atender os demandantes iniciais | Clientes B2B contemplados pelo desenho e validação da primeira fase | Atlas Comercial, MaxDistribuição e Nova Cargo ([09:00] Marcos (TRANSCRICAO.md:L18)) |
| <a id="prd-obj-03"></a>PRD-OBJ-03 | Evitar perda silenciosa | Mudanças confirmadas com endpoint elegível que deixam evento persistido ou evidência de falha | Garantia at-least-once; duplicatas são admitidas ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:25] Diego (TRANSCRICAO.md:L154)) |
| <a id="prd-obj-04"></a>PRD-OBJ-04 | Dar capacidade de recuperação | Falhas esgotadas disponíveis para replay administrativo | Cinco marcos de backoff antes da DLQ; quantidade total de envios pendente ([09:17] Diego (TRANSCRICAO.md:L104); [09:17] Larissa (TRANSCRICAO.md:L108); [09:18] Diego (TRANSCRICAO.md:L110); [09:18] Diego (TRANSCRICAO.md:L114); [09:48] Larissa (TRANSCRICAO.md:L282)) |

Métricas operacionais finais — percentis, janela de medição, SLO e alertas — não foram decididas. FDD-PROP-09 fornece um cenário de aceite reproduzível, explicitamente proposto, e também documenta que concorrência 1 não sustenta a meta sob backlog ou endpoint anterior lento.

## 5. Escopo

### 5.1 Incluído

- Webhooks outbound para mudanças de status de pedido.
- CRUD autenticado de endpoints, ativação e filtros de status.
- Geração e rotação de secret por endpoint.
- Entrega assinada, identificável e retentável.
- Histórico das 100 entregas mais recentes.
- DLQ persistida e replay manual restrito a `ADMIN`, com auditoria.
- Documentação da semântica at-least-once e responsabilidade de deduplicação.

### 5.2 Fora de escopo

- <a id="prd-oos-01"></a>[PRD-OOS-01] Webhooks recebidos pelo OMS (inbound); o fluxo é apenas de saída ([09:02] Marcos (TRANSCRICAO.md:L26)).
- <a id="prd-oos-02"></a>[PRD-OOS-02] Exactly-once; a coordenação necessária foi descartada em favor de at-least-once ([09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:25] Diego (TRANSCRICAO.md:L154)).
- <a id="prd-oos-03"></a>[PRD-OOS-03] Redis ou infraestrutura adicional de fila ([09:07] Diego (TRANSCRICAO.md:L52)).
- <a id="prd-oos-04"></a>[PRD-OOS-04] Múltiplos workers, particionamento e ordering global ([09:12] Diego (TRANSCRICAO.md:L80); [09:13] Larissa (TRANSCRICAO.md:L86); [09:13] Diego (TRANSCRICAO.md:L84)).
- <a id="prd-oos-05"></a>[PRD-OOS-05] Email de alerta/fallback ([09:37] Larissa (TRANSCRICAO.md:L220)).
- <a id="prd-oos-06"></a>[PRD-OOS-06] Dashboard visual para clientes ([09:40] Larissa (TRANSCRICAO.md:L234)).
- <a id="prd-oos-07"></a>[PRD-OOS-07] Rate limiting de saída nesta fase; será observado e decidido depois ([09:39] Larissa (TRANSCRICAO.md:L230)).
- <a id="prd-oos-08"></a>[PRD-OOS-08] Arquivamento/limpeza de eventos entregues ([09:08] Diego (TRANSCRICAO.md:L56)).

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| <a id="prd-rf-01"></a>PRD-RF-01 | Usuário autenticado deve poder cadastrar um endpoint HTTPS para um cliente. | [09:23] Sofia (TRANSCRICAO.md:L138); [09:31] Marcos (TRANSCRICAO.md:L184); [09:32] Larissa (TRANSCRICAO.md:L190); [09:33] Bruno (TRANSCRICAO.md:L192); [09:37] Sofia (TRANSCRICAO.md:L216) |
| <a id="prd-rf-02"></a>PRD-RF-02 | O OMS deve gerar uma secret exclusiva por endpoint e devolvê-la na criação. | [09:21] Sofia (TRANSCRICAO.md:L126); [09:31] Marcos (TRANSCRICAO.md:L184) |
| <a id="prd-rf-03"></a>PRD-RF-03 | Usuário autenticado deve poder listar, editar e remover configurações de webhook. | Operações em [09:33] Bruno (TRANSCRICAO.md:L192); autorização em [09:37] Sofia (TRANSCRICAO.md:L216) |
| <a id="prd-rf-04"></a>PRD-RF-04 | Cada endpoint deve permitir selecionar quais status de pedido deseja receber. | [09:33] Marcos (TRANSCRICAO.md:L194) |
| <a id="prd-rf-05"></a>PRD-RF-05 | Se nenhum endpoint do cliente estiver inscrito no novo status, o OMS não deve criar uma entrega. Usar o campo ativo como condição adicional de elegibilidade é regra proposta no FDD. | [09:33] Marcos (TRANSCRICAO.md:L194); [09:34] Bruno (TRANSCRICAO.md:L198) |
| <a id="prd-rf-06"></a>PRD-RF-06 | Uma mudança elegível deve produzir `order.status_changed` com snapshot do momento da transição, sem itens do pedido. | [09:43] Diego (TRANSCRICAO.md:L256); [09:51] Larissa (TRANSCRICAO.md:L304); [09:52] Larissa (TRANSCRICAO.md:L310) |
| <a id="prd-rf-07"></a>PRD-RF-07 | Cada tentativa deve enviar os headers `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`. | [09:44] Diego (TRANSCRICAO.md:L262); [09:44] Sofia (TRANSCRICAO.md:L264) |
| <a id="prd-rf-08"></a>PRD-RF-08 | O sistema deve aplicar os marcos 1 min, 5 min, 30 min, 2 h e 12 h, mantendo em aberto se representam cinco tentativas totais ou uma tentativa inicial mais cinco retries. | [09:17] Diego (TRANSCRICAO.md:L104); [09:17] Larissa (TRANSCRICAO.md:L108); [09:48] Larissa (TRANSCRICAO.md:L282) |
| <a id="prd-rf-09"></a>PRD-RF-09 | Após esgotar retries, o sistema deve preservar payload, causa e timestamp em uma DLQ separada. | [09:18] Diego (TRANSCRICAO.md:L110) |
| <a id="prd-rf-10"></a>PRD-RF-10 | Um usuário `ADMIN` deve poder solicitar replay da DLQ, com registro de quem executou a ação. | [09:18] Diego (TRANSCRICAO.md:L114); [09:36] Sofia (TRANSCRICAO.md:L210) |
| <a id="prd-rf-11"></a>PRD-RF-11 | Como regra de autenticação proposta por reuso do JWT existente, usuário autenticado deve poder consultar as 100 entregas mais recentes de um webhook, incluindo sucesso/falha, payload, resposta e latência. | [09:34] Marcos (TRANSCRICAO.md:L202) |
| <a id="prd-rf-12"></a>PRD-RF-12 | A rotação deve estar disponível pela API; exigir autenticação nessa operação é extensão proposta do padrão JWT. Na rotação, a anterior deve coexistir por 24 horas. Para não encurtar essa promessa, a proposta técnica rejeita nova rotação enquanto o grace period estiver ativo. | Decisão em [09:21] Sofia (TRANSCRICAO.md:L130); regra sucessiva proposta no FDD |
| <a id="prd-rf-13"></a>PRD-RF-13 | O consumidor deve receber um UUID estável em `X-Event-Id` para deduplicar tentativas do mesmo evento. | [09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:25] Diego (TRANSCRICAO.md:L154) |

Os paths finais, ownership por cliente e posição de `customer_id` ainda precisam de aprovação; o FDD apresenta contratos propostos sem tratá-los como decisões fechadas.

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| <a id="prd-rnf-01"></a>PRD-RNF-01 | A entrega deve chegar em menos de 10 segundos; o recorte de primeira entrega após commit e suas condições de medição são propostos em FDD-PROP-09. | [09:02] Marcos (TRANSCRICAO.md:L22); polling aceito em [09:10] Larissa (TRANSCRICAO.md:L68) |
| <a id="prd-rnf-02"></a>PRD-RNF-02 | A disponibilidade/latência do cliente não deve bloquear a request de mudança de status. | [09:04] Bruno (TRANSCRICAO.md:L32); [09:04] Bruno (TRANSCRICAO.md:L36); [09:06] Diego (TRANSCRICAO.md:L48) |
| <a id="prd-rnf-03"></a>PRD-RNF-03 | Pedido, histórico, estoque e registro do evento devem compartilhar a mesma transação; falha no evento causa rollback. | [09:06] Diego (TRANSCRICAO.md:L48); [09:40] Bruno (TRANSCRICAO.md:L238); [09:41] Diego (TRANSCRICAO.md:L244) |
| <a id="prd-rnf-04"></a>PRD-RNF-04 | A entrega é at-least-once e pode conter duplicatas; exactly-once não é prometido. | [09:24] Diego (TRANSCRICAO.md:L146); [09:25] Diego (TRANSCRICAO.md:L150); [09:25] Diego (TRANSCRICAO.md:L154) |
| <a id="prd-rnf-05"></a>PRD-RNF-05 | O destino deve usar HTTPS e a integridade/autenticidade deve ser protegida por HMAC-SHA256 com secret por endpoint. | [09:22] Sofia (TRANSCRICAO.md:L134); [09:23] Sofia (TRANSCRICAO.md:L138) |
| <a id="prd-rnf-06"></a>PRD-RNF-06 | O payload não pode exceder 64 KB; excedentes geram erro e nunca são truncados. | [09:24] Larissa (TRANSCRICAO.md:L144) |
| <a id="prd-rnf-07"></a>PRD-RNF-07 | Cada chamada externa deve ter timeout de 10 segundos. | [09:42] Diego (TRANSCRICAO.md:L250) |
| <a id="prd-rnf-08"></a>PRD-RNF-08 | A primeira fase usa single-worker, sem garantia global. Retry pode quebrar ordem por pedido sem gating; a proposta técnica bloqueia eventos posteriores até entrega/DLQ/cancelamento e documenta que replay ainda pode chegar fora de ordem. | Limitação em [09:12] Diego (TRANSCRICAO.md:L80); [09:13] Larissa (TRANSCRICAO.md:L86); [09:13] Diego (TRANSCRICAO.md:L84); mecanismo proposto no FDD |
| <a id="prd-rnf-09"></a>PRD-RNF-09 | Logs devem reutilizar Pino; ampliar a redação para não expor secrets é recomendação de integração; códigos do domínio devem usar prefixo `WEBHOOK_`. | [09:29] Larissa (TRANSCRICAO.md:L172); [09:29] Bruno (TRANSCRICAO.md:L174); [09:30] Larissa (TRANSCRICAO.md:L180) |

## 8. Decisões e trade-offs principais

Benefícios e custos são análises derivadas das decisões referenciadas; mecanismos do FDD continuam propostas.

| ID | Decisão | Benefício | Custo/limitação | Registro |
| --- | --- | --- | --- | --- |
| <a id="prd-dec-01"></a>PRD-DEC-01 | Outbox no MySQL | Atomicidade sem nova infraestrutura | Polling e crescimento de tabela | [ADR-001](./adrs/ADR-001-outbox-transacional-no-mysql.md) |
| <a id="prd-dec-02"></a>PRD-DEC-02 | Worker separado, single-worker | Isola a API e simplifica a fase inicial | Vazão limitada; retry exige gating proposto e replay pode sair de ordem | [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |
| <a id="prd-dec-03"></a>PRD-DEC-03 | Backoff limitado + DLQ | Recupera indisponibilidade sem retry infinito | Entrega pode atrasar horas; cardinalidade ainda aberta | [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| <a id="prd-dec-04"></a>PRD-DEC-04 | HMAC por endpoint | Autenticidade e menor raio de vazamento | Custódia e rotação de secrets | [ADR-004](./adrs/ADR-004-hmac-sha256-e-rotacao-de-secret.md) |
| <a id="prd-dec-05"></a>PRD-DEC-05 | At-least-once | Evita desistência em falhas ambíguas | Cliente precisa deduplicar | [ADR-005](./adrs/ADR-005-entrega-at-least-once.md) |
| <a id="prd-dec-06"></a>PRD-DEC-06 | Reuso dos padrões do OMS | Menor divergência e curva de aprendizado | Amplia o acoplamento à stack atual | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-oms.md) |

## 9. Dependências

- Máquina de estados, transação e histórico de pedidos existentes.
- MySQL e Prisma como persistência compartilhada entre API e worker.
- JWT e roles `ADMIN`/`OPERATOR` para autenticação e autorização.
- Zod, `AppError`, error middleware e Pino como padrões internos.
- Endpoint HTTPS funcional e implementação de HMAC/deduplicação pelo cliente.
- Decisão prévia somente das questões que afetam o comportamento em implementação; partes independentes podem avançar conforme a seção 12.
- Planejamento de três sprints e ao menos dois dias úteis de revisão de segurança por Sofia antes do deploy ([09:47] Larissa (TRANSCRICAO.md:L276); [09:46] Sofia (TRANSCRICAO.md:L274)).

## 10. Riscos e mitigação

Probabilidade e impacto são estimativas qualitativas do desenho, não medições nem avaliações feitas na reunião. As mitigações são propostas a partir das premissas rastreadas.

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| <a id="prd-risk-01"></a>PRD-RISK-01 | Cliente não implementa deduplicação e processa duplicatas | Média | Alto | Destacar at-least-once e `X-Event-Id` no contrato e nos testes de integração do cliente |
| <a id="prd-risk-02"></a>PRD-RISK-02 | Endpoint indisponível por período prolongado | Alta no ciclo de vida | Médio | Política limitada nos cinco marcos, DLQ, histórico e replay; confirmar cardinalidade antes do código |
| <a id="prd-risk-03"></a>PRD-RISK-03 | Secret vaza e permite falsificação | Média | Crítico | Secret por endpoint, HTTPS, rotação, redação de logs e revisão de segurança |
| <a id="prd-risk-04"></a>PRD-RISK-04 | Worker único acumula backlog | Baixa no volume inicial | Alto | Observar atraso e backlog; particionamento é evolução futura |
| <a id="prd-risk-05"></a>PRD-RISK-05 | Contratos ainda abertos geram integrações incompatíveis | Média | Alto | Aprovar paths, assinatura, falhas e ownership antes de concluir os comportamentos afetados |
| <a id="prd-risk-06"></a>PRD-RISK-06 | Outbox cresce sem retenção | Média | Médio | Monitorar crescimento e decidir arquivamento em fase posterior |

## 11. Critérios de aceitação

- <a id="prd-ac-01"></a>[PRD-AC-01] Um usuário autenticado consegue criar, listar, editar e remover uma configuração válida.
- <a id="prd-ac-02"></a>[PRD-AC-02] A criação gera secret exclusiva; a rotação fornece uma nova e mantém a anterior por 24 horas. Se a regra sucessiva de FDD-PROP-04 for aprovada, nova rotação durante esse período retorna conflito.
- <a id="prd-ac-03"></a>[PRD-AC-03] Uma mudança elegível gera snapshot atômico; sem endpoint/filtro correspondente, não gera entrega.
- <a id="prd-ac-04"></a>[PRD-AC-04] O cliente recebe o evento com payload e cinco headers acordados, via HTTPS, dentro do limite de 64 KB.
- <a id="prd-ac-05"></a>[PRD-AC-05] Falha retentável segue os cinco marcos conforme a cardinalidade que os revisores aprovarem e, depois de esgotada, aparece na DLQ.
- <a id="prd-ac-06"></a>[PRD-AC-06] Apenas `ADMIN` pode reprocessar DLQ e a ação identifica o executor.
- <a id="prd-ac-07"></a>[PRD-AC-07] A consulta apresenta no máximo as 100 entregas mais recentes com os campos acordados.
- <a id="prd-ac-08"></a>[PRD-AC-08] Tentativas repetidas mantêm o `event_id`; o contrato alerta sobre duplicatas.
- <a id="prd-ac-09"></a>[PRD-AC-09] No cenário nominal de carga/saúde que Produto e Plataforma aprovarem a partir de FDD-PROP-09, a primeira entrega ocorre em menos de 10 segundos; backlog e endpoint anterior lento ficam explicitamente fora dessa comprovação.
- <a id="prd-ac-10"></a>[PRD-AC-10] Nenhum item de fora de escopo aparece como capacidade entregue na primeira fase.

## 12. Estratégia de testes e validação

Plano de validação proposto a partir dos requisitos; participantes, cenários adicionais e gates incrementais não constituem novos compromissos da reunião. A exigência confirmada é reservar ao menos dois dias úteis para Sofia revisar o código de segurança antes do deploy.

- **Produto:** validar os cenários com representantes de Atlas, MaxDistribuição e Nova Cargo e confirmar se filtros, payload e histórico atendem à integração.
- **Contrato:** revisar exemplos de API, corpo outbound, headers, HMAC e semântica de at-least-once com clientes e Segurança.
- **Funcional:** cobrir CRUD, filtro, snapshot, histórico, rotação, retry, DLQ e replay autorizado/não autorizado.
- **Confiabilidade:** simular endpoint lento, timeout, indisponibilidade prolongada, duplicata, crash do worker e falha de persistência da outbox.
- **Desempenho:** medir commit → primeiro byte sob carga, backlog e latência de destino declarados; executar também cenário com endpoint lento para demonstrar o limite do worker sequencial.
- **Segurança:** Sofia revisará geração/custódia/rotação de secret, canonicalização do HMAC, TLS, redação de logs e autorização por ao menos dois dias úteis antes do deploy.
- **Aceite incremental:** a implementação pode começar por partes independentes após revisão do RFC; cada comportamento só é concluído após suas decisões de FDD 18.1, os itens de 18.2 fecham antes do deploy e as evoluções de 18.3 não bloqueiam a primeira fase.
