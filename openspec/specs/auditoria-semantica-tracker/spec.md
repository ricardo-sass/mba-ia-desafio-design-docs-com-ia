# auditoria-semantica-tracker Specification

## Purpose
Definir os critérios de auditoria semântica do Tracker, incluindo evidências primárias, decomposição de afirmações, qualificação de inferências e reconciliação reproduzível da cobertura documental.

## Requirements

### Requirement: Revisão integral das afirmações documentais

A revisão MUST confrontar todas as linhas principais e todos os itens complementares do Tracker com o conteúdo integral correspondente em PRD, RFC, FDD e ADRs. A leitura dos documentos MUST acrescentar ao inventário unidades identificáveis omitidas, sem limitar a verificação ao resumo ou aos exemplos do feedback.

#### Scenario: Revisão de um item composto
- **WHEN** o resumo do Tracker omitir uma condição, restrição ou comportamento do item no documento original
- **THEN** a revisão SHALL verificar também essa afirmação e registrar sua evidência e tratamento

#### Scenario: Unidade ainda não inventariada
- **WHEN** a leitura dos documentos identificar uma unidade relevante sem representação no inventário
- **THEN** a revisão SHALL acrescentá-la ao inventário, decidir sua linha principal ou tratamento complementar e incluí-la no denominador correspondente

#### Scenario: Encerramento da revisão integral
- **WHEN** a revisão for declarada concluída
- **THEN** `auditoria.md` nesta mudança SHALL registrar para cada item anterior e novo o documento/localização, afirmações verificadas, diagnóstico, evidência, tratamento e IDs finais, sem itens pendentes de exame

### Requirement: Evidência direta e localização inequívoca

Toda unidade apresentada como confirmada MUST ter fonte primária que sustente integralmente seu conteúdo. Localizações TRANSCRICAO MUST preservar `[hh:mm] Nome`; quando esse par identificar várias falas, SHALL incluir trecho identificador ou linha que distinga a evidência. Perguntas, concordâncias e decisões MUST ser interpretadas conforme seu conteúdo e contexto.

#### Scenario: HTTPS e HMAC por endpoint
- **WHEN** a revisão verificar as unidades originadas em PRD-RNF-05
- **THEN** a exigência HTTPS SHALL apontar à fala de TLS em [09:23] Sofia e HMAC-SHA256 com secret por endpoint SHALL apontar a [09:22] Sofia ou às falas específicas de [09:20] e [09:21] Sofia
- **THEN** [09:23] Sofia isoladamente SHALL ser rejeitada como sustentação integral do requisito composto

#### Scenario: Payload e cinco headers
- **WHEN** a revisão verificar as unidades originadas em PRD-AC-04
- **THEN** SHALL existir evidência para HTTPS em [09:23] Sofia, limite em [09:24] Larissa, payload em [09:43] Diego, quatro headers em [09:44] Diego e X-Webhook-Id em [09:44] Sofia

#### Scenario: Pergunta sobre timeout
- **WHEN** uma unidade afirmar timeout HTTP de 10 segundos
- **THEN** a localização SHALL identificar [09:42] Diego como a fala que define o valor, sem usar isoladamente a pergunta ou concordância de Sofia como definição do número

#### Scenario: Evidência parcial ou inexistente
- **WHEN** a fonte não sustentar integralmente uma unidade apresentada como confirmada
- **THEN** a revisão SHALL corrigir/complementar a evidência, decompor o item, qualificá-lo ou remover a afirmação infundada antes de considerar a unidade resolvida

### Requirement: Decomposição rastreável sem duplicação

Itens compostos por afirmações independentes com fontes distintas MUST ser decompostos em unidades rastreáveis com IDs únicos. O ID anterior MUST continuar navegável como agrupador ou item preservado. O Tracker SHALL explicar o vínculo entre agrupador, unidades e localização no documento; agrupadores e partes MUST NOT ser contados simultaneamente como unidades principais.

#### Scenario: Decomposição de um requisito
- **WHEN** PRD-RNF-05 for dividido em unidades com sufixos
- **THEN** o leitor SHALL conseguir partir de PRD-RNF-05 para suas unidades e vice-versa, localizando o requisito original e a fonte específica de cada parte
- **THEN** a contagem SHALL substituir o item composto pelas unidades finais, tanto no numerador quanto no denominador

#### Scenario: Mais de uma evidência para uma única análise
- **WHEN** uma unidade indivisível depender de comparar falas, como a ambiguidade da cardinalidade de retry
- **THEN** a localização SHALL indicar as falas e seus papéis sem multiplicar a mesma unidade por número de referências

### Requirement: Qualificação de inferências e fontes de código

Toda fonte CODIGO MUST ser verificada quanto à existência do caminho e ao conteúdo que sustenta a afirmação. Propostas, integrações futuras, consequências derivadas e lacunas MUST indicar no tipo/resumo e no documento de origem quais fatos são comprovados e quais conclusões ou extensões permanecem propostas. Nenhum caminho futuro MUST ser usado como prova de implementação existente.

#### Scenario: Redação de secrets no logger
- **WHEN** a unidade usar `src/shared/logger/index.ts` para justificar proteção de secrets de webhook
- **THEN** SHALL distinguir a redação de campos atualmente implementada da extensão proposta para secrets de webhook, sem afirmar que os caminhos de redação futuros já existem

#### Scenario: Validação de chave-mestra
- **WHEN** a unidade usar `src/config/env.ts` para justificar uma chave-mestra de secrets de webhook
- **THEN** SHALL identificar a validação Zod existente como premissa e a inclusão da chave como proposta, sem atribuir ao arquivo configuração que ele não contém

#### Scenario: Questões e decisões preservadas
- **WHEN** a revisão tratar atomicidade, filtros, at-least-once, deduplicação, retry/DLQ, HMAC/TLS, rotação, autorização, limites ou observabilidade
- **THEN** SHALL manter as decisões sustentadas e qualificar separadamente propostas e lacunas, incluindo a cardinalidade do retry, sem usar a correção editorial para decidir comportamento novo

### Requirement: Reconciliação reproduzível da cobertura

A auditoria MUST recalcular o inventário e os percentuais após as correções e decomposições. Numerador, denominador, contagens por documento e proporções de fontes MUST ser reproduzíveis pelas unidades enumeradas. Unidades complementares MUST ter localização e tratamento explícitos; sua promoção à tabela principal MUST eliminar sua contagem complementar. As metas MUST permanecer em pelo menos 80% de cobertura, 70% de linhas TRANSCRICAO e cinco linhas CODIGO com caminhos reais e conteúdo pertinente.

#### Scenario: Reconstrução após decomposição
- **WHEN** o inventário final tiver número de unidades diferente do inicial
- **THEN** a auditoria SHALL recalcular todas as contagens afetadas sem reutilizar os percentuais anteriores e sem contar simultaneamente agrupadores, partes ou complementares promovidos

#### Scenario: Meta quantitativa atendida com fonte incorreta
- **WHEN** os percentuais mínimos forem atingidos mas houver unidade confirmada com evidência parcial ou incorreta
- **THEN** a revisão SHALL permanecer incompleta até tratar a unidade; o percentual SHALL ser descrito como cobertura do inventário, sem alegação de que prova sustentação semântica

#### Scenario: Remoção de unidade
- **WHEN** uma unidade anterior for removida do inventário
- **THEN** a auditoria SHALL registrar a justificativa e o tratamento no documento de origem, sem reduzir o denominador apenas para alcançar a meta

### Requirement: Entrega documental verificável e preservação do workspace

A entrega MUST manter as seis colunas obrigatórias do Tracker, IDs únicos, vínculos válidos e consistência com os documentos de origem. O registro da auditoria MUST separar verificações mecânicas de revisão semântica manual e informar os resultados efetivamente obtidos. Alterações MUST permanecer documentais e preservar mudanças preexistentes do usuário, sem alterar aplicação, testes, configurações ou transcrição.

#### Scenario: Validação final
- **WHEN** a revisão for finalizada
- **THEN** SHALL haver evidência da verificação de estrutura, IDs, mapa de decomposição, fontes, links, aritmética, critérios aplicáveis do enunciado e revisão semântica de todas as unidades
- **THEN** SHALL ser executado `openspec validate revisar-referencias-tracker --strict`, e o README SHALL relatar somente a revisão e as validações efetivamente realizadas

#### Scenario: Alterações anteriores no repositório
- **WHEN** o workspace já contiver arquivos modificados no início da aplicação
- **THEN** a revisão SHALL comparar o resultado ao estado inicial capturado e preservar as alterações anteriores, sem usar um diff global contra HEAD como prova de autoria desta mudança
