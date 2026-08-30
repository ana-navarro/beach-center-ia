---
name: "speckit-unit-tests"
description: "Gera testes unitários com Jest para o código implementado, mockando a infraestrutura e mirando 80% de cobertura."
argument-hint: "Slug da tarefa ou caminho/arquivo alvo dos testes"
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

**Check for extension hooks (before unit_tests)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_unit_tests` key
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

Quarto passo do Ciclo Speckt (Princípio V), em conjunto com `/speckit-component-tests`. Gera os
testes unitários do código produzido em `/speckit-implement`.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md` (Princípio III) e o
   `plan.md` da tarefa (Critérios de Aceite `AC-*`).

2. **Localizar alvos**: Todos os arquivos criados/alterados na tarefa. Excluir da geração e da
   métrica de cobertura qualquer arquivo marcado com o comentário `@wip`.

3. **Gerar testes com `Jest`** (Backend: `Jest`/`Vitest` conforme o serviço; Frontend: `Jest`):
   - Um arquivo de teste por arquivo de produção (`*.spec.ts` / `*.test.ts` conforme convenção
     do serviço), no local esperado pelo projeto.
   - **Backend**: mockar **completamente** as Portas (interfaces) de infraestrutura. Focar na
     camada de **Domínio** (usecases). Testar também controllers e adapters conforme exigido
     pelo Princípio III ("validação de controllers, adapters e use cases").
   - Cada `AC-*` do plano MUST ter ao menos um caso de teste correspondente.
   - Cobrir **cenários de sucesso e de falha** (inputs inválidos, exceções de port, regras de
     negócio violadas).

4. **Verificar cobertura**:
   - Rode a suíte com cobertura (`npm test -- --coverage` ou equivalente do serviço).
   - A cobertura MUST ser **>= 80%** (linhas/statements). Se abaixo, adicione casos até atingir.
   - Todos os testes gerados MUST passar.

5. **Sem commit**: apenas cria/altera arquivos de teste localmente.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_unit_tests`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_unit_tests` key.
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

- Arquivos de teste criados.
- Mapa `AC-* → teste(s)`.
- Resultado da execução (passando/falhando) e **percentual de cobertura**.
- Arquivos `@wip` ignorados.
- Próximo passo sugerido: `/speckit-component-tests` (se ainda não rodou) ou `/speckit-validate`.

## Done When

- [ ] Testes unitários Jest gerados para todos os arquivos da tarefa (exceto `@wip`)
- [ ] Portas de infraestrutura totalmente mockadas no backend
- [ ] Cada Critério de Aceite coberto por ao menos um teste
- [ ] Cenários de sucesso e falha cobertos
- [ ] Suíte passando com cobertura >= 80%
- [ ] Nenhum commit realizado
- [ ] Extension hooks dispatched or skipped according to the rules above
