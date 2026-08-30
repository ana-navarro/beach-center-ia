---
name: "speckit-test"
description: "Gera um .md com os cenários de testes exploratórios (caminho feliz, exceções e edge cases) a partir dos Critérios de Aceite."
argument-hint: "Slug da tarefa (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.6 / Comandos de Desenvolvimento"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before test)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_test` key
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

Sexto passo do Ciclo Speckt (Princípio V). Como o projeto não possui analistas de QA, este
comando gera um roteiro de **testes exploratórios manuais**. **NÃO** escreve código de teste
automatizado (isso é `/speckit-unit-tests` e `/speckit-component-tests`).

1. **Ler contexto**: Carregue `.specify/memory/constitution.md`, o `context.md` e o `plan.md` da
   tarefa (Critérios de Aceite `AC-*`).

2. **Gerar `tasks/<NNN>-<slug>/exploratory-tests.md`** contendo:
   - **Escopo e pré-condições**: serviço(s), dados de seed, usuários/perfis necessários.
   - **Caminhos felizes**: passo a passo por `AC-*`, com resultado esperado.
   - **Fluxos de exceção**: entradas inválidas, falhas de integração (pagamentos, WhatsApp),
     concorrência (ex.: duas reservas na mesma quadra/horário), permissões/auth.
   - **Edge cases**: limites de data/horário, fuso, mensalista x avulso, cancelamento,
     reagendamento, campeonato com vagas esgotadas, valores zerados/negativos.
   - **Checklist de regressão**: fluxos vizinhos que podem ser afetados.
   - Cada item em formato de tabela: `ID | Pré-condição | Passos | Resultado esperado`.

3. **Rastreabilidade**: cada `AC-*` MUST aparecer em pelo menos um cenário; marque os cenários
   que não têm cobertura automatizada equivalente.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_test`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_test` key.
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

- Caminho do `exploratory-tests.md` gerado.
- Contagem de cenários por categoria (feliz / exceção / edge / regressão).
- Cobertura `AC-* → cenário`.
- Próximo passo sugerido: `/speckit-complete`.

## Done When

- [ ] `tasks/<NNN>-<slug>/exploratory-tests.md` criado
- [ ] Cenários de caminho feliz, exceção e edge cases documentados
- [ ] Todos os Critérios de Aceite rastreados a pelo menos um cenário
- [ ] Nenhum código de aplicação ou teste automatizado alterado
- [ ] Extension hooks dispatched or skipped according to the rules above
