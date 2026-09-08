<!--
Sync Impact Report
- Version change: 1.3.0 → 1.3.1
- Rationale (1.3.0 → 1.3.1, PATCH): Correção do caminho da documentação de endpoints — de `beach-center-documentations/` (plural) para `beach-center-documentation/` (singular), pasta única dentro do repositório `beach-center-ia`.
- Modified principles: none
- Modified sections:
  - Comandos de Desenvolvimento: `/speckit-documentation` — caminho corrigido para `beach-center-documentation/[nome-repo]/[nome-rota].md`.
- Added sections: none
- Removed sections: none
- Deferred TODOs: none
- Prior amendment history:
  - 1.2.0 → 1.3.0 (MINOR): Removed documentation generation from `/speckit-complete`. Reworked `/speckit-documentation` to map endpoints and generate PT-BR documentation.
  - 1.2.0 (MINOR): Added dedicated test-generation commands (`/speckit-unit-tests` and `/speckit-component-tests`) and a strict local pre-push quality gate on `/speckit-complete` enforcing 80% test coverage...
  - 1.1.1 (PATCH): Set ratification date to 2026-08-27...
  - 1.0.0 → 1.1.0 (MINOR): Added strict ESLint compliance for all generated code...
-->

# Constituição do Projeto Beach Center (Gerenciamento de Quadras)

**Propósito**: Este projeto é um Sistema de Gerenciamento de Quadras de Beach Tennis, projetado para administrar aulas, aluguéis, mensalistas e campeonatos. Todo código gerado, revisado ou validado neste repositório MUST seguir estritamente os princípios abaixo.

## Core Principles

### I. Fronteiras do Ecossistema de Micro-Serviços

O ecossistema adota uma arquitetura de micro-serviços independente. Código MUST ser alocado respeitando estritamente estas fronteiras de projetos:

- **Frontend (FE - app):**
  - `beach-center-app`: Aplicação Frontend (React/TypeScript).
- **Backend (BE - services):**
  - `services/beach-center-bff-agendamentos`: Microsserviço responsável pelas regras de agendamentos de quadras.
  - `services/beach-center-bff-pagamentos`: Microsserviço responsável pelo processamento e histórico de pagamentos.
  - `services/beach-center-bff-usuarios`: Microsserviço responsável pela gestão de usuários e autenticação (auth).
  - `services/beach-center-bff-aulas`: Microsserviço responsável pela gestão das aulas (A SER CRIADO).
  - `services/beach-center-whatsapp`: Serviço de integração e bot para WhatsApp.
- **Infraestrutura:**
  - `beach-center-server`: Configurações de gateway, orquestração de containers e scripts de infraestrutura.

**ATENÇÃO (Regra de Arquitetura):** Apesar do sufixo `bff` presente na nomenclatura de alguns repositórios legados, o projeto **NÃO** utiliza o pattern de Backend For Frontend (BFF). Todos os serviços de backend listados acima MUST atuar como micro-serviços puros e independentes.

### II. Arquitetura Hexagonal e Fluxo de Domínio

Todos os micro-serviços (Node.js/TypeScript) MUST respeitar os princípios SOLID e seguir estritamente a estrutura de pastas e regras da Arquitetura Hexagonal (Ports & Adapters) abaixo:

**Estrutura de Pastas Padrão (Backend)**:
- `applications/`: Camada de entrada/saída.
  - `controllers/`: Recebem as requisições HTTP/Eventos.
  - `routes/`: Definição dos endpoints.
  - `middlewares/`: Guards, annotations e interceptadores.
  - `dto/`: Interfaces/Tipagens para validação de dados de entrada/saída.
- `domain/`: Coração da aplicação.
  - `usecases/`: Contém TODAS as regras de negócio. Nunca mexem com schemas, apenas models.
  - `models/`: Entidades de domínio chamadas pelos usecases.
  - `ports/`: Interfaces de Input/Output.
- `infra/`: Detalhes técnicos e dependências externas.
  - `adapters/`: Chamadas para banco de dados e integrações. **NÃO podem conter regras de negócio**. Deve existir **um adapter para cada verbo/ação**.
  - `schemas/`: Estruturas de dados para infraestrutura (ex: ORM, tabelas).
  - `ports/`: Interfaces de Output (os usecases chamam estes ports para acessar os adapters).
- `config/`: Configurações gerais e variáveis de ambiente.
- `main.ts` / `index.ts`: Ponto de entrada (root) do serviço.

**Regras de Comunicação e Responsabilidade**:
1. **Isolamento via Ports**:
   - A camada de `domain` comunica com a de `infra` APENAS via ports (Usecases nunca chamam Adapters diretamente).
   - A camada de `applications` comunica com a de `domain` APENAS via ports.
2. **Responsabilidade**: Usecases = Exclusivos para regras de negócio do beach tennis (aulas, aluguéis, torneios); Adapters = Exclusivos para persistência e integrações.

### III. Test-First e Qualidade (NON-NEGOTIABLE)

- **Linting e Formatação**: Todo código TypeScript/JavaScript gerado MUST seguir estritamente as regras configuradas no `eslint` do projeto. A validação do ESLint é obrigatória.
- Backend: `Jest` (ou `Vitest`) MUST ser usado para testes unitários.
- Frontend (React): `Jest` MUST ser usado para testes unitários; `Cypress` em conjunto com `Cucumber` MUST ser usado para testes de componente/E2E.
- A cobertura de testes unitários MUST ser de **no mínimo 80%**.
- A abrangência dos testes unitários MUST envolver todos os arquivos, garantindo validação de controllers, adapters e use cases.
- Testes unitários de backend MUST mockar completamente as Portas (interfaces) de infraestrutura, focando exclusivamente na camada de Domínio.
- **Work In Progress (WIP)**: Qualquer arquivo marcado com o comentário `@wip` MUST ter seus testes unitários ignorados (skipped) na execução e desconsiderado da métrica de cobertura.

### IV. Infraestrutura Reproduzível com Hot-Reloading

A infraestrutura em `beach-center-server` MUST utilizar volumes Docker para permitir modificações em tempo real (hot-reloading) durante o desenvolvimento local.

### V. Workflow de Implementação Automatizada (Ciclo Speckit)

A implementação de novas funcionalidades MUST seguir a ordem estrita abaixo:

1. **`/speckit-task`**: Criação do contexto da tarefa e diretório.
2. **`/speckit-plan`**: Geração do checklist e **Critérios de Aceite**.
3. **`/speckit-implement`**: Geração isolada do código fonte.
4. **`/speckit-unit-tests`** / **`/speckit-component-tests`**: Geração de testes baseados no código implementado e critérios de aceite.
5. **`/speckit-validate`**: Revisão iterativa arquivo por arquivo.
6. **`/speckit-test`**: Geração de cenários de testes exploratórios (Manual/QA).
7. **`/speckit-complete`**: Validação local estrita e Push da pipeline.
8. **`/speckit-documentation`**: Atualização e criação da documentação baseada nos endpoints do repositório.

---

## Comandos de Desenvolvimento (Prompt Commands)

Quando o usuário acionar os comandos abaixo, o assistente MUST atuar da seguinte forma (respeitando a ordem do Workflow):

- `/speckit-task [descrição]`:
  - Cria uma pasta dentro do diretório `tasks/` com um nome descritivo.
  - Levanta o contexto do que precisará ser feito, solicitando mais informações sobre as regras de negócio se a descrição for insuficiente.

- `/speckit-plan`:
  - MUST ser executado após a task e antes do implement.
  - Cria um arquivo de planejamento dentro da pasta da task contendo um **checklist de implementação** detalhado.
  - MUST gerar os **Critérios de Aceite** formais da funcionalidade para guiar o desenvolvimento e os testes futuros.

- `/speckit-implement`:
  - Lê o plano/checklist gerado no passo anterior.
  - Implementa as mudanças de código solicitadas nos diretórios apropriados, garantindo a conformidade estrita com as regras do ESLint.
  - **Atenção**: Esta etapa NÃO realiza commit. Apenas altera/cria os arquivos localmente.

- `/speckit-unit-tests`:
  - Gera os testes unitários utilizando `Jest`, garantindo o mock correto das camadas de infraestrutura.
  - Os testes MUST cobrir cenários de sucesso e falha para alcançar a **cobertura mínima de 80%**.

- `/speckit-component-tests`:
  - Gera os testes de componente/E2E utilizando `Cypress` integrado com `Cucumber`.
  - MUST criar o arquivo `.feature` descrevendo os cenários baseados nos Critérios de Aceite gerados no plan, além da implementação real dos "steps".

- `/speckit-validate`:
  - Inicia o modo de revisão **arquivo por arquivo** das alterações geradas (código e testes).
  - Exibe o que foi alterado em um arquivo específico e aguarda o input do usuário.
  - Se o usuário digitar `next`: O assistente MUST avançar para o próximo arquivo editado.
  - Se o usuário digitar `not approved`: O assistente MUST perguntar o que precisa ser modificado naquele arquivo específico e gerar a correção antes de avançar.
  - Somente após a aprovação de todos os arquivos, finaliza a validação preparando-os para commit.

- `/speckit-test`:
  - Gera um arquivo `.md` contendo todos os **cenários de testes exploratórios** recomendados (caminhos felizes, fluxos de exceção e edge cases) com base nos Critérios de Aceite criados no plan.

- `/speckit-complete`:
  - **Quality Gate Local (Blocker):** ANTES de realizar qualquer push, o assistente MUST analisar e confirmar localmente que:
    1. O ESLint não acusa erros.
    2. Todos os testes unitários estão passando.
    3. Todos os testes de componente (Cypress/Cucumber) estão passando.
    4. A cobertura de código atinge a métrica mínima de 80%.
  - Se alguma destas validações locais falhar, a execução é interrompida, e os erros MUST ser corrigidos.
  - Estando tudo aprovado, gera os commits utilizando Conventional Commits e empurra (`git push`) todos os repositórios afetados.

- `/speckit-documentation`:
  - Atualiza e cria toda a documentação baseando-se no código atual do repositório.
  - MUST buscar e mapear todos os endpoints existentes na aplicação.
  - A documentação gerada MUST ser redigida em português.
  - MUST salvar o documento de cada endpoint isoladamente, seguindo a estrutura de pastas: `beach-center-documentation/[nome-repo]/[nome-rota].md`.
  - Exemplo de caminho de arquivo esperado: `beach-center-documentation/beach-center-bff-agendamentos/agendamento.md`.

### Comandos de Utilitários
- `/update-repos`: Sugerir comandos para atualizar dependências e sincronizar branches.
- `/run-server`: Ajustar os scripts em `beach-center-server` para subir os containers.

---

## Governance

- Esta Constituição prevalece sobre qualquer outra prática ou convenção.
- Emendas MUST ser registradas neste arquivo, com atualização do Sync Impact Report e da linha de versão no rodapé.
- Versionamento semântico dos princípios:
  - MAJOR: remoção ou redefinição incompatível.
  - MINOR: adição de novo princípio.
  - PATCH: esclarecimentos ou correções de redação.
- Toda revisão de PR MUST verificar conformidade com esta Constituição.
- Complexidade que viole a Arquitetura Hexagonal (Princípio II) ou a regra contra uso de BFFs (Princípio I) MUST ser explicitamente justificada ou rejeitada.

**Version**: 1.3.1 | **Ratified**: 2026-08-30 | **Last Amended**: 2026-09-08