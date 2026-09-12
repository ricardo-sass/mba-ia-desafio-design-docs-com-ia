## ADDED Requirements

### Requirement: PRD orientado ao produto
O pacote MUST conter `docs/PRD.md` em pt-BR com todas as seções obrigatórias do `ENUNCIADO.md`, no mínimo oito requisitos funcionais rastreáveis, ao menos um objetivo com métrica e meta quantitativa, exclusões confirmadas e riscos avaliados por probabilidade, impacto e mitigação.

#### Scenario: Cobertura mínima do PRD
- **WHEN** um revisor confrontar `docs/PRD.md` com a checklist do enunciado
- **THEN** todas as seções estarão presentes, haverá pelo menos oito requisitos funcionais, uma meta quantitativa, duas exclusões e dois riscos qualificados

#### Scenario: Separação do nível de produto
- **WHEN** o PRD mencionar decisões de arquitetura ou implementação
- **THEN** ele SHALL resumir somente o trade-off relevante ao produto e referenciar o documento técnico apropriado, sem duplicar seu detalhamento

### Requirement: RFC arquitetural submetível à revisão
O pacote MUST conter `docs/RFC.md` conciso, com metadados, participantes da reunião como revisores, resumo executivo, contexto, visão arquitetural, ao menos duas alternativas realmente descartadas, ao menos duas questões não decididas, impacto, riscos e links para ao menos dois ADRs.

#### Scenario: Alternativas e questões abertas no RFC
- **WHEN** um revisor inspecionar as seções de alternativas e questões abertas
- **THEN** cada alternativa SHALL apresentar a evidência e o trade-off de descarte, e cada questão SHALL permanecer explicitamente sem decisão

#### Scenario: Fronteira entre RFC e FDD
- **WHEN** o RFC descrever outbox, worker, retry, HMAC ou APIs
- **THEN** ele SHALL permanecer no nível de proposta e delegar contratos, algoritmos e fluxos detalhados ao FDD
- **THEN** parâmetros como tamanho da janela, duração de lease e sintaxe de locking MUST NOT ser repetidos no RFC

### Requirement: ADRs para decisões isoladas
O pacote MUST conter entre cinco e oito arquivos `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`, cada um com Status, Contexto, Decisão, Alternativas Consideradas e Consequências positivas e negativas, cobrindo ao menos cinco das seis decisões principais exigidas pelo enunciado.

#### Scenario: Validação estrutural dos ADRs
- **WHEN** os arquivos em `docs/adrs/` forem enumerados e suas seções verificadas
- **THEN** a contagem SHALL estar entre cinco e oito, a nomenclatura SHALL seguir o padrão e todos os arquivos SHALL conter as cinco seções mínimas

#### Scenario: Integração com o código atual
- **WHEN** o conjunto de ADRs for revisado
- **THEN** ao menos um ADR MUST referenciar explicitamente arquivos, módulos, classes ou padrões existentes e explicar a consequência dessa integração

### Requirement: FDD acionável e fiel às decisões
O pacote MUST conter `docs/FDD.md` com os fluxos de criação da outbox, processamento, retry e DLQ; contratos propostos; matriz `WEBHOOK_*`; resiliência; métricas, logs e tracing; testes; compatibilidade; riscos; critérios técnicos e integração com ao menos quatro caminhos reais do código.

#### Scenario: Cobertura dos contratos HTTP
- **WHEN** um desenvolvedor consultar a seção de contratos públicos
- **THEN** encontrará pelo menos quatro endpoints com request, response, headers, status codes e semântica, distinguindo decisões confirmadas de contratos ainda propostos

#### Scenario: Tradução de erros Zod do domínio
- **WHEN** os schemas de webhook rejeitarem `url` ou `statuses`
- **THEN** o FDD SHALL explicar onde o `ZodError` será convertido em `WEBHOOK_INVALID_URL` ou `WEBHOOK_INVALID_STATUS_FILTER`
- **THEN** o comportamento padrão `VALIDATION_ERROR` das rotas existentes SHALL permanecer inalterado

#### Scenario: Atomicidade da outbox
- **WHEN** o FDD descrever uma mudança válida de status com webhook ativo e filtro compatível
- **THEN** ele SHALL exigir a persistência do snapshot na outbox na mesma transação Prisma de pedido, histórico e estoque
- **THEN** ele SHALL exigir rollback integral se o enqueue falhar

#### Scenario: Filtro sem destinatário elegível
- **WHEN** nenhum webhook ativo do cliente estiver inscrito no novo status
- **THEN** o FDD SHALL especificar que nenhuma linha de entrega será inserida na outbox

#### Scenario: Entrega segura
- **WHEN** o worker enviar um evento `order.status_changed`
- **THEN** o FDD SHALL exigir HTTPS, timeout de 10 segundos, corpo de no máximo 64 KB sem truncamento, HMAC-SHA256 e os headers `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`

#### Scenario: Retry e DLQ
- **WHEN** uma entrega sofrer falha classificada como retentável
- **THEN** o FDD SHALL registrar os cinco marcos temporais 1 min, 5 min, 30 min, 2 h e 12 h
- **THEN** o FDD MUST manter explícita a ambiguidade entre cinco tentativas totais e uma tentativa inicial mais cinco retries até confirmação dos revisores
- **THEN** após o esgotamento SHALL especificar persistência em DLQ separada e replay manual auditado por usuário `ADMIN`

#### Scenario: Garantia at-least-once
- **WHEN** uma tentativa for reenviada ou recuperada após falha
- **THEN** o mesmo UUID de evento SHALL permanecer disponível em `X-Event-Id` para deduplicação pelo consumidor
- **THEN** o documento MUST declarar que duplicatas são possíveis e que não há exactly-once

#### Scenario: Limitação de ordenação
- **WHEN** o FDD tratar da ordem das entregas
- **THEN** ele MUST negar que o single-worker sozinho garanta ordem durante retry e MUST negar garantia global ou sob múltiplos workers

#### Scenario: Ordenação durante retry
- **WHEN** um evento anterior de um pedido estiver aguardando seu próximo marco de retry
- **THEN** o FDD MUST reconhecer que ordenar apenas por `created_at` permite overtaking
- **THEN** o FDD SHALL propor sequência monotônica e gating apenas entre eventos do mesmo destino e `order_id`
- **THEN** timestamps iguais SHALL ser desempatados pela sequência da transição, sem bloquear outro endpoint saudável
- **THEN** o FDD MUST explicitar que replay posterior da DLQ ainda pode chegar fora de ordem

#### Scenario: Claim compatível com o lease
- **WHEN** o worker consultar até 20 candidatos e executar HTTP com concorrência 1 e timeout de 10 segundos
- **THEN** o FDD SHALL tratar 20 somente como janela de consulta sem reserva
- **THEN** o worker SHALL reivindicar um evento imediatamente antes de cada HTTP e o lease de 60 segundos SHALL se aplicar somente ao item em curso

#### Scenario: Conclusão atômica de tentativa
- **WHEN** uma chamada terminar com sucesso, retry ou falha definitiva
- **THEN** `WebhookDelivery`, estado da outbox e eventual DLQ SHALL ser confirmados na mesma transação
- **THEN** um token de claim obsoleto MUST NOT gravar histórico nem estado após recuperação do lease
- **THEN** falha da transação de conclusão SHALL deixar o item recuperável sem histórico parcial

#### Scenario: Meta de latência verificável
- **WHEN** o pacote usar a meta inferior a 10 segundos como aceite
- **THEN** o FDD SHALL declarar volume, backlog inicial, saúde do worker/banco, latência do destino e ponto inicial/final da medição
- **THEN** o FDD MUST reconhecer que concorrência 1 pode violar a meta quando uma chamada lenta antecede outra

#### Scenario: Desativação ou remoção com pendências
- **WHEN** um endpoint for desativado ou removido enquanto possuir eventos pendentes
- **THEN** o FDD SHALL propor cancelamento transacional das pendências, interrupção de novos enqueues/retries e preservação do histórico
- **THEN** reativar o endpoint MUST NOT ressuscitar eventos cancelados

#### Scenario: Replay repetido
- **WHEN** a mesma entrada de DLQ receber solicitações de replay repetidas ou concorrentes
- **THEN** o FDD SHALL propor uma única outbox de replay e SHALL retornar sua identidade sem duplicar trabalho ou auditoria

### Requirement: Segurança e autorização documentadas
Os documentos técnicos MUST especificar secret única por endpoint, geração pelo sistema, retorno controlado na criação, rotação com coexistência da secret anterior por 24 horas, validação de URL HTTPS e replay da DLQ restrito a `ADMIN` com auditoria.

#### Scenario: Rotação da secret
- **WHEN** uma configuração solicitar rotação
- **THEN** a documentação SHALL estabelecer uma nova secret e validade paralela da anterior por 24 horas
- **THEN** armazenamento, criptografia e formato final da assinatura SHALL permanecer sinalizados como questões abertas até revisão de segurança

#### Scenario: Rotação sucessiva durante o grace period
- **WHEN** uma nova rotação for solicitada antes de expirarem as 24 horas da secret anterior
- **THEN** o FDD SHALL propor uma regra que não substitua a secret anterior nem encurte sua validade prometida

#### Scenario: Baseline e extensão da assinatura
- **WHEN** o FDD apresentar a mensagem canônica do HMAC
- **THEN** ele SHALL separar `HMAC(rawBody)` confirmado na reunião da extensão proposta `<timestamp>.<rawBody>`
- **THEN** a extensão MUST ser marcada como alteração que exige aprovação e versionamento/comunicação aos consumidores

#### Scenario: Pendências separadas por horizonte
- **WHEN** o FDD listar lacunas e decisões pendentes
- **THEN** ele SHALL separar bloqueios do comportamento correspondente, definições necessárias antes do deploy e evoluções futuras
- **THEN** retenção/limpeza definitiva MUST permanecer fora do escopo da primeira fase e MUST NOT bloquear o início do código

#### Scenario: Critério derivado de proposta
- **WHEN** um critério de aceite depender de um default não aprovado
- **THEN** o FDD MUST marcá-lo explicitamente como condicional à aprovação da proposta correspondente

#### Scenario: Autorização de replay
- **WHEN** um ator sem role `ADMIN` tentar reprocessar uma entrada da DLQ
- **THEN** o contrato SHALL exigir rejeição pelo padrão de autorização existente e nenhum replay

### Requirement: Consistência e delimitação do estado futuro
Todos os documentos MUST distinguir o OMS atual da solução futura, usar termos e contratos consistentes entre si e não promover itens descartados, futuros ou abertos a requisitos confirmados.

#### Scenario: Verificação de exclusões
- **WHEN** inbound webhooks, exactly-once, Redis, múltiplos workers, email, dashboard, rate limiting de saída ou limpeza da outbox forem encontrados
- **THEN** eles SHALL aparecer somente como alternativa, limitação, risco, questão aberta ou fora de escopo, conforme a transcrição

#### Scenario: Verificação de caminhos
- **WHEN** um documento citar um arquivo do repositório
- **THEN** o caminho MUST existir no momento da revisão e sua descrição SHALL corresponder ao conteúdo observado
