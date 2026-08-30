---
name: "speckit-component-tests"
description: "Gera testes de componente/E2E com Cypress + Cucumber, criando os arquivos .feature a partir dos Critérios de Aceite e implementando os steps."
argument-hint: "Slug da tarefa ou fluxo alvo dos testes de componente"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.4 / III (Test-First e Qualidade)"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before component_tests)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_component_tests` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

Quarto passo do Ciclo Speckt (Princípio V), em conjunto com `/speckit-unit-tests`. Gera os
testes de componente/E2E do frontend (`beach-center-app`) ou dos fluxos de ponta a ponta.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md` (Princípio III: Frontend
   MUST usar `Cypress` + `Cucumber` para testes de componente/E2E) e o `plan.md` da tarefa
   (Critérios de Aceite `AC-*`).

2. **Gerar arquivos `.feature` (Gherkin)**:
   - Um `.feature` por fluxo/funcionalidade, no diretório de e2e do projeto (ex.:
     `cypress/e2e/` conforme a convenção do repositório).
   - Cada cenário MUST derivar de um `AC-*` do plano (Given/When/Then).
   - Incluir cenários de caminho feliz e de exceção descritos nos Critérios de Aceite.

3. **Implementar os step definitions**:
   - Implementação **real** dos steps (`@badeball/cypress-cucumber-preprocessor` ou o preprocessor
     já configurado no projeto), sem steps vazios ou `pending`.
   - Seletores estáveis (`data-cy`/`data-testid`); adicionar esses atributos ao componente
     apenas se o Princípio II e o escopo da tarefa permitirem, registrando no relatório.

4. **Executar**:
   - Rode a suíte Cypress em modo headless (`npx cypress run` ou script do projeto).
   - Todos os cenários gerados MUST passar. Corrija steps/seletores até passar.

5. **Sem commit**: apenas cria/altera arquivos de teste localmente.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_component_tests`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_component_tests` key.
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue to the Completion Report.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Mandatory hook** (`optional: false`) — **You MUST emit `EXECUTE_COMMAND:` for each mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

- Arquivos `.feature` e de steps criados.
- Mapa `AC-* → cenário`.
- Resultado da execução Cypress (cenários passando/falhando).
- Atributos de teste adicionados a componentes, se houver.
- Próximo passo sugerido: `/speckit-validate`.

## Done When

- [ ] Arquivos `.feature` gerados a partir dos Critérios de Aceite
- [ ] Step definitions implementados de verdade (sem pending)
- [ ] Suíte Cypress + Cucumber passando em headless
- [ ] Nenhum commit realizado
- [ ] Extension hooks dispatched or skipped according to the rules above
