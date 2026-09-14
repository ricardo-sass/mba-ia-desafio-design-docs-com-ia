## Context

A mudança anterior produziu PRD, RFC, FDD, seis ADRs e Tracker, mas permanece em `openspec/changes/criar-pacote-design-docs-webhooks/`. O Tracker declara 240 linhas principais e 27 itens complementares, cobertura de 89,9% e 211 referências à transcrição. Esses números são o ponto de partida declarado, a conferir no início da aplicação; não constituem certificação semântica.

Há afirmações compostas ligadas a apenas uma fala: PRD-RNF-05 reúne HTTPS e HMAC por endpoint; PRD-AC-04 reúne HTTPS, payload, limite e headers. Há também resumos que omitem partes do item original e fontes de código que comprovam somente o padrão a estender. Produto, Engenharia e Segurança precisam conseguir reproduzir cada vínculo sem inferir decisões não registradas.

A arquitetura futura continua sendo API → transação MySQL com pedido/histórico/estoque/outbox → worker separado → endpoint HTTPS externo. A atomicidade é discutida em [09:40] Bruno, o processo separado em [09:11] Diego e HTTPS em [09:23] Sofia. Esta revisão documenta a origem dessas afirmações; não altera essa arquitetura nem cria seus componentes.

## Goals / Non-Goals

**Objetivos:**

- Examinar integralmente o inventário e os documentos de origem, incluindo unidades sem linha principal e afirmações ausentes dos resumos.
- Deixar cada unidade confirmada sustentada diretamente e cada inferência/proposta explicitamente delimitada.
- Preservar IDs de navegação, as seis colunas do Tracker e uma contagem reproduzível sem duplicar agrupadores e partes.
- Produzir um registro de auditoria que permita conferir o destino de todos os itens anteriores e novos.

**Não objetivos:**

- Implementar a feature, alterar fontes primárias ou decidir cardinalidade de retry, criptografia em repouso, assinatura com timestamp, claim/locking, gating, métricas ou contratos ainda propostos.
- Reescrever a arquitetura, criar ADR de produto por decisão editorial, instalar ferramentas ou executar testes que dependam de banco.
- Arquivar/reabrir a mudança anterior ou modificar seu histórico de tarefas para representar esta revisão.

## Decisions

### 1. Auditar o item integral e registrar o resultado de cada revisão

**Decisão documental:** criar `auditoria.md` nesta mudança durante a aplicação. Registrar para cada ID anterior: documento e seção/âncora, afirmações verificadas, diagnóstico, evidência exata, tratamento e IDs finais. O mesmo registro inclui os itens complementares e unidades novas descobertas na leitura. Os diagnósticos iniciais são `direta`, `parcial`, `incorreta`, `derivada/proposta` e `sem evidência`; toda pendência recebe um tratamento final verificável.

Ler ADRs, RFC, FDD e PRD integralmente, confrontando cada unidade com o Tracker. Não limitar a revisão a linhas que contêm conjunções nem a buscas pelos dois IDs do feedback. Decisões, alternativas e consequências dos ADRs continuam individualizadas. Observabilidade, riscos, migração, deploy, rollback e recomendações também entram na revisão do inventário.

**Motivação:** PRD-AC-04 inclui o formato do payload no documento original, embora o resumo do Tracker só diga “HTTPS, headers e limite de payload”. [09:43] Diego é necessário para verificar essa parte.

**Alternativa considerada:** corrigir apenas as células reportadas. É menor, mas deixa o mesmo defeito em outros itens. O registro integral custa mais leitura e torna a conclusão auditável.

### 2. Decompor por afirmação verificável e preservar o ID de origem como agrupador

**Decisão documental:** manter os IDs atuais nos documentos como agrupadores quando um item for decomposto. Na tabela principal, substituir a linha composta por unidades com sufixos estáveis, como `PRD-RNF-05-A` e `PRD-RNF-05-B`. Documentar na auditoria e em um mapa de decomposição no Tracker a relação entre agrupador, unidades e localização no documento. Não contar o agrupador como uma linha adicional. Unidades sem decomposição mantêm seu ID.

Exemplo de decomposição proposta:

| Unidade | Conteúdo | Evidência primária |
| --- | --- | --- |
| PRD-RNF-05-A | Destino HTTPS obrigatório | [09:23] Sofia, fala iniciada por “TLS obrigatório.” |
| PRD-RNF-05-B | HMAC-SHA256 sobre o corpo com secret por endpoint | [09:22] Sofia, consolidação da decisão |
| PRD-AC-04-A | Entrega por HTTPS | [09:23] Sofia, fala sobre TLS |
| PRD-AC-04-B | Payload com os campos acordados, sem itens | [09:43] Diego |
| PRD-AC-04-C | Limite de 64 KB | [09:24] Larissa |
| PRD-AC-04-D | Content-Type, X-Event-Id, X-Signature e X-Timestamp | [09:44] Diego |
| PRD-AC-04-E | X-Webhook-Id identifica o cadastro de destino | [09:44] Sofia |

Agrupar detalhes que uma mesma fala sustenta integralmente é permitido. Uma unidade indivisível pode citar mais de uma fala quando depende da comparação entre elas, como a ambiguidade da cardinalidade do retry; a localização explicita o papel de cada evidência. Não criar uma linha por referência repetida para aumentar a cobertura.

**Alternativas consideradas:** apenas acrescentar timestamps preservaria o formato, mas não cumpriria a desagregação exigida pela especificação anterior; renumerar todos os IDs quebraria vínculos desnecessariamente. Sufixos e mapa acrescentam manutenção, mas preservam a navegação.

### 3. Desambiguar as falas e preservar sua força probatória

**Decisão documental:** manter `[hh:mm] Nome` em cada localização. Quando esse par ocorrer mais de uma vez, acrescentar um trecho identificador curto ou linha da transcrição. Citações múltiplas terão autores explícitos e escopo claro, sem intervalos genéricos que incluam perguntas, sugestões rejeitadas ou participantes errados.

Casos obrigatórios de conferência, além do feedback:

| Caso | Evidência a verificar |
| --- | --- |
| Snapshot e conteúdo do evento | [09:43] Diego para campos/ausência de itens; [09:52] Larissa ou Diego para snapshot na inserção |
| Worker | [09:09] Diego ou [09:10] Larissa para polling; [09:11] Diego para separação; [09:12] Diego para single-worker; [09:30] Bruno para client próprio |
| Timeout | [09:42] Diego define 10 segundos; Sofia pergunta e depois concorda |
| CRUD e autenticação | [09:31] Marcos; [09:32] Larissa corrige customer_id; [09:33] Bruno lista operações; [09:37] Sofia confirma roles autenticadas |
| Elegibilidade e atomicidade | [09:33] Marcos e [09:34] Bruno para filtro; [09:40] Bruno para transação e rollback |
| Retry, DLQ e replay | [09:15] Diego; [09:17] Diego e Larissa; [09:18] Diego para tabela/replay; [09:36] Sofia para ADMIN e auditoria; [09:48] Larissa para confrontar a cardinalidade |
| Deduplicação | [09:24] Diego para duplicatas; [09:25] Diego para UUID em X-Event-Id; [09:26] Marcos para documentar ao consumidor |
| Exclusões | Confrontar cada exclusão com sua própria decisão; [09:40] Larissa sobre painel não sustenta todas as exclusões |

**Alternativa considerada:** aceitar qualquer par timestamp/falante existente. Isso falha, por exemplo, nas duas falas de [09:23] Sofia e nas duas de [09:24] Diego. Desambiguar aumenta a precisão sem alterar a transcrição.

### 4. Separar evidência direta, premissa e extensão proposta

**Decisão documental:** corrigir `Tipo` e resumo para explicitar o alcance da fonte, harmonizando o documento de origem quando necessário. Recomendações devem identificar o fato que as motiva e o comportamento adicional proposto. Se não houver evidência direta nem premissa defensável, remover a afirmação infundada ou reformulá-la como pergunta aberta; registrar o tratamento, sem inventar uma fala.

Na auditoria de `CODIGO`, ler o conteúdo de cada caminho, não apenas testar sua existência. Exemplos já inspecionados:

- `src/shared/logger/index.ts` contém Pino e redação de campos como password/token, mas não inclui secrets de webhook. A extensão é proposta.
- `src/config/env.ts` contém validação Zod do ambiente, mas não uma chave-mestra de secrets de webhook. FDD-INT-17 precisa delimitar esse alcance.
- `src/config/database.ts` oferece uma fábrica de PrismaClient; a exigência de instância própria do worker vem de [09:30] Bruno.
- `src/middlewares/error.middleware.ts` sustenta o envelope de erros; autenticação e logger exigem seus caminhos próprios.

Uma unidade técnica derivada pode usar a fonte que comprova sua premissa, desde que isso esteja explícito. Unidades confirmadas independentes com fontes de tipos distintos serão separadas para preservar `Fonte = TRANSCRICAO` ou `Fonte = CODIGO`.

**Alternativa considerada:** usar “proposto” como ressalva geral para qualquer conteúdo. Isso não explica a relação causal nem resolve afirmações equivocadas. A classificação será específica por unidade.

### 5. Reconciliar cobertura estrutural e revisão semântica separadamente

**Decisão documental:** reconstruir o inventário depois da decomposição. Contar unidades principais finais uma vez; somar somente unidades complementares distintas ao denominador. Uma unidade promovida à tabela principal sai do conjunto complementar. Cada agrupador decomposto é substituído por suas partes também no denominador. Remoções são justificadas na auditoria, nunca usadas apenas para atingir um percentual.

Publicar números por documento, total, proporção de cada fonte e itens complementares enumerados. Manter as metas do enunciado: cobertura mínima de 80%, ao menos 70% de linhas TRANSCRICAO e ao menos cinco linhas CODIGO. As proporções não substituem a revisão: o registro deve demonstrar tratamento de 100% dos itens inventariados e ausência de referências parciais/incorretas ainda apresentadas como confirmação.

**Alternativa considerada:** atualizar apenas o numerador ou reaproveitar os 89,9%. Isso torna a métrica inválida após a decomposição. Os valores finais permanecem desconhecidos até a reconciliação.

## Risks / Trade-offs

- **[Alterações preexistentes no workspace]** → Registrar estado e conteúdo inicial durante a aplicação; preservar mudanças do usuário e não usar reset/checkout para rollback.
- **[Revisão estrutural tomada por semântica]** → Registrar leitura e tratamento por item; separar verificações mecânicas da conclusão manual.
- **[Inferência promovida a decisão]** → Conferir o texto da fonte e o rótulo no documento de origem, especialmente riscos, consequências, segurança e gates de implementação/deploy.
- **[Contagem inflada por decomposição]** → Reconciliar agrupadores, partes e complementares antes de calcular percentuais.
- **[Referências por linha envelhecem]** → Preservar timestamp/falante e usar trecho identificador ou símbolo de código como apoio.
- **[Revisão aumenta o tamanho do Tracker]** → Manter unidades concisas e usar o registro da auditoria para justificativas detalhadas.

## Migration Plan

1. Capturar o estado inicial e inventário; criar a matriz de auditoria.
2. Revisar ADRs, RFC, FDD e PRD contra transcrição/código, registrando os tratamentos sem alterar fontes primárias.
3. Corrigir Tracker e apenas os trechos correlatos dos documentos de origem; reconciliar IDs e contagens.
4. Registrar resultados e comandos em `auditoria.md`; atualizar o README com a iteração efetivamente realizada.
5. Verificar as seis colunas, unicidade de IDs, resolução de vínculos, existência e conteúdo das fontes, inventário complementar e aritmética. Confrontar manualmente as afirmações finais, com atenção aos casos da seção 3.
6. Executar `openspec validate revisar-referencias-tracker --strict` e a validação da mudança documental anterior para detectar incompatibilidades. Conferir o diff e que aplicação, testes, configurações e transcrição preservam seu conteúdo inicial.

Não há migração de banco, deploy ou revisão de código de segurança nesta mudança. A exigência futura de revisão por Sofia ([09:46] Sofia) permanece documentada. Validações podem usar scripts efêmeros sem dependências novas; não criar testes de aplicação para alterações editoriais. Rollback é a reversão seletiva dos trechos desta revisão, preservando o estado anterior do usuário.

## Open Questions

- Quantas unidades adicionais e referências parciais serão identificadas na revisão integral? A auditoria responderá sem fixar contagens antecipadas.
- Alguma afirmação mantida nos documentos não possui premissa primária defensável? Cada caso terá remoção, reformulação ou qualificação explícita antes do encerramento.
- Questões da feature já registradas continuam abertas; esta mudança não as transforma em decisões nem depende de resolvê-las para corrigir a rastreabilidade.
