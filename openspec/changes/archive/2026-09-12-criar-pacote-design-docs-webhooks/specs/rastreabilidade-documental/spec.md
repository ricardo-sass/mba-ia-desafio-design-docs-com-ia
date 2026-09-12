## ADDED Requirements

### Requirement: Tracker no formato obrigatório
O pacote MUST conter `docs/TRACKER.md` como tabela Markdown com as colunas `ID`, `Documento`, `Tipo`, `Conteúdo (resumo)`, `Fonte` e `Localização`, usando um ID único para cada item.

#### Scenario: Estrutura da tabela
- **WHEN** o Tracker for analisado por um revisor ou script
- **THEN** o cabeçalho SHALL conter exatamente as seis colunas exigidas e cada linha de conteúdo SHALL possuir um ID não repetido

### Requirement: Fontes primárias válidas
Cada linha do Tracker MUST usar `TRANSCRICAO` com timestamp e falante válidos ou `CODIGO` com caminho real; `ENUNCIADO.md` poderá sustentar a estrutura da entrega, mas não substituir a evidência de requisitos e decisões da feature.

#### Scenario: Evidência da transcrição
- **WHEN** uma linha declarar `Fonte = TRANSCRICAO`
- **THEN** sua localização SHALL usar o formato `[hh:mm] Nome` e corresponder a uma fala existente em `TRANSCRICAO.md`

#### Scenario: Evidência do código
- **WHEN** uma linha declarar `Fonte = CODIGO`
- **THEN** sua localização SHALL apontar para um caminho existente e o conteúdo resumido SHALL ser verificável nesse arquivo

#### Scenario: Afirmação sem origem conclusiva
- **WHEN** uma informação não puder ser sustentada por fonte primária
- **THEN** ela MUST ser removida, qualificada como recomendação/questão aberta ou receber evidência válida antes da conclusão

### Requirement: Cobertura quantitativa
O Tracker MUST cobrir ao menos 80% dos itens identificáveis do pacote, MUST ter ao menos 70% das linhas com fonte `TRANSCRICAO` e MUST incluir ao menos cinco linhas com fonte `CODIGO` e caminhos reais distintos ou relevantes.

#### Scenario: Auditoria de cobertura
- **WHEN** requisitos, decisões, restrições, exclusões e trade-offs identificáveis forem comparados às linhas do Tracker
- **THEN** as três metas quantitativas SHALL ser calculadas e satisfeitas antes da entrega
- **THEN** todo item mantido no denominador sem linha principal SHALL ser enumerado com documento, localização, conteúdo e tratamento
- **THEN** cada ADR SHALL contar separadamente sua decisão principal, cada alternativa e cada consequência individual
- **THEN** itens derivados MUST indicar que a fonte comprova a premissa, sem apresentar inferência como evidência direta
- **THEN** uma linha que agregue requisitos ou exclusões com fontes diferentes MUST ser desagregada em unidades rastreáveis
- **THEN** o percentual SHALL ser descrito como cobertura do inventário declarado, não como prova autônoma de completude semântica

### Requirement: Rastreabilidade cruzada estável
Os IDs usados no Tracker MUST corresponder inequivocamente ao documento e ao item resumido, e links/caminhos para os documentos e ADRs MUST permanecer válidos.

#### Scenario: Navegação de um item
- **WHEN** um revisor selecionar um ID do Tracker
- **THEN** conseguirá localizar o item correspondente no documento declarado e confirmar sua origem sem depender de inferência não registrada
