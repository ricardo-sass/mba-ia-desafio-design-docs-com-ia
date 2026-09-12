## 1. Matriz de fontes e vocabulário

- [x] 1.1 Extrair de `TRANSCRICAO.md` uma matriz de requisitos, decisões, alternativas, exclusões e questões abertas, registrando para cada item o timestamp e o falante exatos.
- [x] 1.2 Inspecionar os pontos de integração em `src/`, `prisma/` e `tests/`, registrar o comportamento observado e validar a existência de cada caminho que poderá ser citado.
- [x] 1.3 Definir IDs estáveis, termos canônicos e uma lista de contratos confirmados versus propostos para uso consistente em PRD, RFC, FDD, ADRs e Tracker.

## 2. Registros de decisões arquiteturais

- [x] 2.1 Criar `docs/adrs/ADR-001-outbox-transacional-no-mysql.md` no formato MADR, fundamentando atomicidade, snapshot e descarte de HTTP síncrono/Redis.
- [x] 2.2 Criar `docs/adrs/ADR-002-worker-separado-com-polling.md` no formato MADR, registrando processo separado, polling de 2 segundos, single-worker e limitação de ordenação.
- [x] 2.3 Criar `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` no formato MADR, registrando as cinco janelas, DLQ separada e replay manual administrativo.
- [x] 2.4 Criar `docs/adrs/ADR-004-hmac-sha256-e-rotacao-de-secret.md` no formato MADR, registrando secret por endpoint, TLS, rotação e grace period de 24 horas sem decidir lacunas de armazenamento/assinatura.
- [x] 2.5 Criar `docs/adrs/ADR-005-entrega-at-least-once.md` no formato MADR, registrando duplicatas, UUID em `X-Event-Id` e descarte de exactly-once.
- [x] 2.6 Criar `docs/adrs/ADR-006-reuso-dos-padroes-do-oms.md` no formato MADR, relacionando explicitamente módulo, Zod, `AppError`, Pino, middleware, Prisma e caminhos reais do código.
- [x] 2.7 Validar contagem, nomes, seções MADR, alternativas, consequências positivas/negativas, fontes e cobertura das seis decisões no conjunto de ADRs.

## 3. RFC para revisão arquitetural

- [x] 3.1 Redigir `docs/RFC.md` com metadados, participantes da reunião como revisores, TL;DR, contexto e visão geral da arquitetura futura, mantendo-o conciso e sem detalhe próprio do FDD.
- [x] 3.2 Documentar no RFC ao menos duas alternativas efetivamente descartadas, ao menos duas questões abertas, impactos e riscos, com timestamps e trade-offs verificáveis.
- [x] 3.3 Adicionar links relativos válidos para ao menos dois ADRs e revisar o RFC contra os ADRs e a transcrição, removendo divergências ou decisões não confirmadas.

## 4. FDD acionável

- [x] 4.1 Redigir em `docs/FDD.md` contexto técnico, objetivos, escopo/exclusões e a arquitetura proposta, distinguindo explicitamente estado atual, decisão confirmada, recomendação e questão aberta.
- [x] 4.2 Detalhar os fluxos de enqueue transacional com snapshot/filtros, claim e processamento pelo worker, política de retry com contagem ambígua, DLQ e replay auditado; separar decisões confirmadas de defaults técnicos propostos.
- [x] 4.3 Propor e identificar como sujeitos à revisão ao menos quatro contratos HTTP completos, com requests, responses, headers, status codes e semântica consistentes, preservando `GET /webhooks/:id/deliveries` e `POST /admin/webhooks/dead-letter/:id/replay` como paths verbalizados.
- [x] 4.4 Especificar o payload `order.status_changed`, limite de 64 KB sem truncamento, timeout de 10 segundos, HTTPS, headers de entrega, HMAC-SHA256, rotação e garantia at-least-once.
- [x] 4.5 Criar a matriz de erros com códigos `WEBHOOK_*` e cobrir autorização JWT, `ADMIN` no replay, validação, conflito, falha de entrega e limites conforme os padrões existentes.
- [x] 4.6 Escrever a seção “Integração com o sistema existente” usando ao menos quatro caminhos reais e descrevendo a fronteira transacional, máquina de estados, composição, rotas, autenticação, erros, logs e persistência relevantes.
- [x] 4.7 Documentar métricas, logs e tracing; testes unitários, integração e ponta a ponta; dependências, compatibilidade, migração, deploy/rollback, riscos, mitigação e critérios técnicos, qualificando itens não decididos.
- [x] 4.8 Revisar o FDD contra a transcrição, código, ADRs e checklist do enunciado; validar contagem de endpoints, prefixo de erros, caminhos citados e ausência de contradições.

## 5. PRD no nível de produto

- [x] 5.1 Redigir `docs/PRD.md` com resumo, problema, público, cenários, objetivos, dependências e ao menos uma métrica com meta quantitativa rastreável.
- [x] 5.2 Documentar escopo, ao menos oito requisitos funcionais, requisitos não funcionais, decisões/trade-offs, critérios de aceitação e estratégia de testes/validação sem duplicar o FDD.
- [x] 5.3 Registrar explicitamente ao menos duas exclusões confirmadas e ao menos dois riscos com probabilidade, impacto e mitigação.
- [x] 5.4 Revisar o PRD contra a transcrição, RFC, FDD e checklist do enunciado, preservando a fronteira de produto e removendo detalhes técnicos indevidos.

## 6. Rastreabilidade e relato do processo

- [x] 6.1 Criar `docs/TRACKER.md` com as seis colunas obrigatórias e uma linha de ID único para cada requisito, decisão, restrição, exclusão e trade-off identificável nos documentos.
- [x] 6.2 Validar cada localização `TRANSCRICAO` contra timestamp/falante e cada localização `CODIGO` contra caminho e conteúdo reais, corrigindo ou removendo afirmações sem evidência.
- [x] 6.3 Calcular e registrar a auditoria do Tracker, comprovando cobertura mínima de 80%, ao menos 70% das linhas com fonte `TRANSCRICAO` e ao menos cinco linhas com fonte `CODIGO`.
- [x] 6.4 Substituir `README.md` pelo relato em pt-BR do processo realmente executado, incluindo as seis seções obrigatórias, ferramentas e workflow.
- [x] 6.5 Incluir no README ao menos dois prompts customizados em blocos de código e ao menos duas iterações/ajustes concretos, sem atribuir ferramentas ou etapas que não tenham sido usadas.
- [x] 6.6 Adicionar ao README os links para todos os entregáveis e uma ordem sugerida de leitura; verificar que cada link resolve para um arquivo existente.

## 7. Revisão final não destrutiva

- [x] 7.1 Auditar todos os critérios de aceite de `ENUNCIADO.md`, incluindo seções e contagens mínimas de PRD, RFC, FDD, ADRs, Tracker e README.
- [x] 7.2 Buscar inconsistências de endpoints, headers, payloads, códigos `WEBHOOK_*`, números, estados e termos; alinhar as ocorrências a um contrato canônico ou marcá-las como abertas.
- [x] 7.3 Verificar todos os timestamps/falantes, caminhos de código e links Markdown, garantindo que nenhuma fonte ou arquivo citado seja inexistente.
- [x] 7.4 Confirmar com `git diff -- src prisma tests TRANSCRICAO.md` e inspeção das configurações que a mudança alterou somente os documentos autorizados.
- [x] 7.5 Executar `openspec validate criar-pacote-design-docs-webhooks --strict` e as demais validações documentais não destrutivas disponíveis; corrigir todos os erros antes de marcar a proposta como concluída.

## 8. Revisão crítica posterior

- [x] 8.1 Harmonizar PRD, RFC, FDD, ADR-003, Tracker, README e artefatos OpenSpec para manter explícita a ambiguidade entre cinco tentativas totais e tentativa inicial mais cinco retries.
- [x] 8.2 Corrigir a autoria das evidências no Tracker, reconhecer os dois paths verbalizados e substituir a alegação de 100% por uma auditoria conservadora de todos os itens identificáveis.
- [x] 8.3 Tornar o FDD mais acionável com propostas concretas e premissas explícitas para classificação HTTP, claim/recuperação após crash, assinatura e rotação de secrets.
- [x] 8.4 Revalidar fontes, consistência, cobertura, links, escopo documental e OpenSpec em modo estrito.

## 9. Revisão de ordenação, leases e acionabilidade

- [x] 9.1 Explicitar o risco de overtaking durante retry e propor gating por `order_id`, incluindo a limitação após DLQ/replay.
- [x] 9.2 Alinhar claim e lease: usar 20 apenas como janela de candidatos e reservar um evento imediatamente antes da chamada HTTP.
- [x] 9.3 Enumerar integralmente o inventário sem linha própria do Tracker e corrigir referências que sustentam apenas parte do conteúdo.
- [x] 9.4 Definir como propostas o efeito de desativar/excluir sobre pendências e a idempotência de replay repetido.
- [x] 9.5 Revalidar consistência, contagens, fontes, links, escopo documental e OpenSpec em modo estrito.

## 10. Revisão de bloqueios, validação, rotação e cobertura dos ADRs

- [x] 10.1 Separar decisões indispensáveis antes do código, definições necessárias antes do deploy e evoluções futuras, mantendo retenção/limpeza fora do escopo da primeira fase.
- [x] 10.2 Especificar como a validação Zod produzirá `WEBHOOK_INVALID_URL` e `WEBHOOK_INVALID_STATUS_FILTER` sem alterar o comportamento padrão das rotas existentes.
- [x] 10.3 Definir a semântica de rotações sucessivas dentro das 24 horas sem encurtar a validade prometida à secret anterior.
- [x] 10.4 Decompor alternativas e consequências de cada ADR em unidades rastreáveis, explicitar o método do inventário e recalcular integralmente a cobertura do Tracker.
- [x] 10.5 Harmonizar os artefatos, atualizar o relato da iteração e revalidar IDs, fontes, links, escopo documental, contagens e OpenSpec em modo estrito.

## 11. Revisão semântica, fronteiras e consistência transacional

- [x] 11.1 Desagregar exclusões compostas no RFC/Tracker e explicitar os limites probatórios da auditoria manual de cobertura.
- [x] 11.2 Alinhar PRD/FDD sobre implementação incremental e remover parâmetros operacionais detalhados do RFC.
- [x] 11.3 Tornar condicionais os critérios derivados de propostas e separar HMAC sobre o corpo confirmado da extensão `timestamp.rawBody`.
- [x] 11.4 Definir cenário verificável de latência e documentar a limitação do worker sequencial sob backlog ou endpoint lento.
- [x] 11.5 Isolar ordenação por destino e pedido, com sequência monotônica como desempate de transições simultâneas.
- [x] 11.6 Especificar transações atômicas de finalização para sucesso, retry e DLQ, incluindo token de claim e recuperação após interrupção.
- [x] 11.7 Atualizar Tracker, README e artefatos OpenSpec, recalculando inventário e fontes após as novas unidades.
- [x] 11.8 Revalidar consistência semântica, IDs, contagens, fontes, links, escopo documental e OpenSpec em modo estrito.
