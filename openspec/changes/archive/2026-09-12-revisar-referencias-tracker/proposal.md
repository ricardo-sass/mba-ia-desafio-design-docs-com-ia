## Why

O Tracker contém referências existentes que sustentam apenas parte das afirmações: PRD-RNF-05 aponta HTTPS e HMAC por endpoint somente para [09:23] Sofia, embora HMAC e secrets sejam confirmados em [09:22] Sofia; PRD-AC-04 também depende de [09:24] Larissa, [09:43] Diego e [09:44] Diego e Sofia. A revisão deve tornar verificável a origem de cada unidade documental e impedir que a contagem de linhas seja confundida com sustentação integral das afirmações.

## What Changes

- Revisar todas as linhas principais e todo o inventário complementar de `docs/TRACKER.md` contra o conteúdo integral dos itens em PRD, RFC, FDD e ADRs e suas fontes primárias.
- Desagregar afirmações compostas com evidências distintas, mantendo a navegação dos IDs originais para suas unidades e evitando dupla contagem.
- Corrigir timestamps, autoria e alcance de cada referência; identificar a fala por trecho ou linha quando timestamp e falante se repetirem.
- Conferir o conteúdo dos caminhos `CODIGO`, diferenciando comportamento atual, integração futura, inferência, proposta e lacuna.
- Ajustar os documentos de origem apenas onde a correção de evidência, classificação ou identificação exigir consistência; preservar decisões e questões abertas da feature.
- Recalcular cobertura e proporções das fontes a partir do inventário revisado; registrar evidências da revisão integral e atualizar o relato no README com o trabalho efetivamente executado.

## Capabilities

### New Capabilities

- `auditoria-semantica-tracker`: revisão verificável de cada afirmação, decomposição rastreável, qualificação de evidências e reconciliação da cobertura documental.

### Modified Capabilities

Nenhuma especificação está consolidada em `openspec/specs/`. Esta capacidade complementa a rastreabilidade definida na mudança `criar-pacote-design-docs-webhooks`, ainda não arquivada, sem duplicar ou substituir seus requisitos.

## Impact

- **Objetivo:** nenhuma unidade apresentada como confirmada permanecer com referência parcial, incorreta ou sem sustentação; toda inferência mantida terá premissa e alcance explícitos.
- **Arquivos:** `docs/TRACKER.md`; ajustes correlatos em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/` e `README.md`; artefatos desta mudança e registro documental da auditoria.
- **Dependências:** versão de trabalho dos documentos, `ENUNCIADO.md`, `TRANSCRICAO.md` e inspeção dos caminhos citados, como `src/shared/logger/index.ts`, `src/config/database.ts` e `src/middlewares/error.middleware.ts`. A existência de um arquivo não basta como prova do comportamento atribuído a ele.
- **Não objetivos:** implementar webhooks, alterar aplicação/testes/configurações/transcrição, reabrir decisões da reunião, arquivar a mudança anterior ou criar infraestrutura/dependências.
- **Riscos:** perder vínculos ao decompor IDs, inflar cobertura, atribuir perguntas a decisões ou transformar propostas em requisitos; a matriz de revisão e a reconciliação das contagens devem detectar esses casos.
- **Questões abertas:** o total final de unidades e correções será determinado pela revisão integral. A ambiguidade de tentativas ([09:17] e [09:48] Larissa), os contratos propostos e demais lacunas de produto permanecem explícitos, sem decisões novas nesta revisão.
