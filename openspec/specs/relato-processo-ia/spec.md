# relato-processo-ia Specification

## Purpose
Definir o relato verificável do uso de IA na produção e revisão do pacote documental, com prompts reproduzíveis, ajustes realizados e navegação da entrega.

## Requirements

### Requirement: README centrado no processo real
O `README.md` MUST substituir o conteúdo anterior por uma descrição em pt-BR do processo efetivamente usado para produzir e revisar o pacote, incluindo Sobre o desafio, Ferramentas de IA utilizadas, Workflow adotado, Prompts customizados, Iterações e ajustes e Como navegar a entrega.

#### Scenario: Cobertura das seções
- **WHEN** o README for comparado à estrutura obrigatória do `ENUNCIADO.md`
- **THEN** todas as seis seções SHALL existir e descrever esta entrega, não apenas repetir o enunciado

### Requirement: Ferramentas e workflow verificáveis
O README MUST listar ao menos uma ferramenta de IA realmente utilizada e explicar seu papel, além de apresentar a ordem de produção e a forma de revisão adotadas.

#### Scenario: Relato fiel das ferramentas
- **WHEN** uma ferramenta ou etapa for mencionada
- **THEN** ela SHALL corresponder a uma ação realizada no repositório e não a uma capacidade hipotética

### Requirement: Prompts customizados reproduzíveis
O README MUST conter ao menos dois prompts relevantes em blocos de código, com contexto suficiente para que outro leitor entenda o objetivo de extração, geração ou revisão.

#### Scenario: Inspeção dos prompts
- **WHEN** os blocos de código da seção de prompts forem enumerados
- **THEN** ao menos dois SHALL representar prompts distintos e específicos para as fontes e critérios deste desafio

### Requirement: Iterações e ajustes honestos
O README MUST registrar o número de iterações principais e ao menos dois ajustes concretos realizados após identificar saída superficial, inconsistente ou sem rastreabilidade.

#### Scenario: Evidência de revisão crítica
- **WHEN** um revisor ler a seção de iterações
- **THEN** encontrará pelo menos dois problemas concretos, a correção aplicada e o efeito observado no pacote

### Requirement: Navegação completa da entrega
O README MUST listar os caminhos de todos os documentos produzidos e recomendar uma ordem de leitura coerente com seus papéis.

#### Scenario: Acesso aos artefatos
- **WHEN** um leitor seguir a seção de navegação
- **THEN** todos os caminhos SHALL existir e conduzir a PRD, RFC, FDD, Tracker e ADRs sem links quebrados

