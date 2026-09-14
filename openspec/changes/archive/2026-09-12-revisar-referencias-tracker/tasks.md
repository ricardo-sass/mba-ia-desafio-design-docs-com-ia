## 1. Estado inicial e inventário

- [x] 1.1 Registrar o estado inicial do workspace e preservar uma referência do conteúdo dos arquivos que serão revisados, distinguindo alterações preexistentes das desta mudança.
- [x] 1.2 Conferir as contagens iniciais do Tracker e criar `auditoria.md` nesta mudança com todas as linhas principais e todos os itens complementares, preparando campos de localização, afirmações, diagnóstico, evidência, tratamento e IDs finais.
- [x] 1.3 Indexar as falas da transcrição por timestamp/falante, identificar pares repetidos e registrar como as localizações serão desambiguadas; relacionar os caminhos CODIGO a inspecionar.

## 2. Revisão integral dos documentos e das fontes

- [x] 2.1 Revisar integralmente ADR-001 e ADR-002 contra as fontes, cobrindo decisões, alternativas, consequências, snapshot, atomicidade, polling, processo separado, client próprio e single-worker; registrar cada tratamento na auditoria.
- [x] 2.2 Revisar integralmente ADR-003 e ADR-004, cobrindo retry, cardinalidade, DLQ/replay, HMAC, HTTPS, secrets e rotação; separar evidência direta de inferência e registrar cada tratamento.
- [x] 2.3 Revisar integralmente ADR-005 e ADR-006, cobrindo deduplicação, consequências derivadas e reuso dos padrões; conferir o conteúdo de todos os caminhos citados e registrar cada tratamento.
- [x] 2.4 Revisar integralmente o RFC, incluindo objetivos, decisões compostas, exclusões, riscos e questões abertas; confrontar cada afirmação com fontes e Tracker e acrescentar unidades omitidas ao inventário.
- [x] 2.5 Revisar no FDD objetivos, modelos, fluxos, propostas e contratos HTTP, cobrindo filtros, snapshot, falhas, limites, autorização e headers; registrar evidências e unidades adicionais.
- [x] 2.6 Revisar no FDD integração com código, erros, observabilidade, testes, migração/deploy/rollback, riscos, critérios e questões abertas; conferir cada caminho CODIGO e distinguir extensão futura de comportamento existente.
- [x] 2.7 Revisar integralmente o PRD, incluindo objetivos, escopo, requisitos, decisões, riscos, critérios e conteúdo complementar; registrar a sustentação de todas as partes de PRD-RNF-05 e PRD-AC-04 e dos demais itens compostos.
- [x] 2.8 Consolidar a auditoria de todas as unidades anteriores e novas, com tratamento definido para fontes parciais/incorretas, inferências e afirmações sem evidência, sem pendências de exame.

## 3. Correção documental e navegação

- [x] 3.1 Corrigir nos ADRs e RFC apenas referências, classificações e trechos cujo alcance precise ser harmonizado, preservando decisões confirmadas e questões abertas.
- [x] 3.2 Corrigir no FDD e PRD as referências e classificações identificadas na auditoria, incluindo a distinção entre requisitos confirmados, propostas e premissas de código.
- [x] 3.3 Substituir no Tracker as linhas compostas por unidades verificáveis com sufixos estáveis e criar o mapa de decomposição para os IDs originais e suas localizações nos documentos, sem contar agrupadores como unidades adicionais.
- [x] 3.4 Corrigir todas as localizações TRANSCRICAO e CODIGO, desambiguando falas repetidas e explicitando nas unidades derivadas/propostas o alcance exato da fonte.
- [x] 3.5 Conferir novamente os casos de HTTPS/HMAC, cinco headers, limite de 64 KB, timeout de 10 s, snapshot, worker, CRUD/autorização, filtros, retry/DLQ/replay, deduplicação e exclusões contra os textos finais e as fontes primárias.

## 4. Cobertura e relato

- [x] 4.1 Reconciliar cada ID anterior com seus IDs finais ou justificativa de remoção; atualizar o inventário complementar, eliminando contagem duplicada de unidades promovidas ou decompostas.
- [x] 4.2 Recalcular contagens por documento, numerador, denominador e proporções de fontes, comprovando as metas do enunciado e distinguindo cobertura estrutural de revisão semântica.
- [x] 4.3 Completar `auditoria.md` com evidências e resultados efetivos da revisão de todas as unidades; atualizar o README com a iteração realizada e link para o registro da auditoria.

## 5. Validação final

- [x] 5.1 Verificar as seis colunas, unicidade dos IDs, mapa de decomposição, vínculos aos documentos, timestamps/falantes/trechos, caminhos reais e aritmética; registrar comandos e resultados na auditoria.
- [x] 5.2 Confrontar manualmente cada unidade final com sua evidência e o documento de origem, confirmando ausência de afirmações apresentadas como decisão/requisito confirmado com fonte parcial, incorreta ou insuficiente.
- [x] 5.3 Conferir os critérios aplicáveis de `ENUNCIADO.md` e executar `openspec validate revisar-referencias-tracker --strict` e `openspec validate criar-pacote-design-docs-webhooks --strict`, corrigindo problemas desta revisão antes de concluir.
- [x] 5.4 Comparar o resultado ao estado inicial, confirmar que aplicação, testes, configurações e transcrição foram preservados, revisar o diff documental e registrar as limitações e o resultado final sem atribuir validação semântica apenas aos checks estruturais.
