---
name: "speckit-validate"
description: "Revisão interativa arquivo por arquivo das alterações da tarefa (código e testes), avançando com next e corrigindo com not approved."
argument-hint: "Slug da tarefa (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.5 / Comandos de Desenvolvimento"
user-invocable: true
disable-model-invocation: true
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before validate)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_validate` key
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

Quinto passo do Ciclo Speckt (Princípio V). Revisão **iterativa, arquivo por arquivo**, de tudo
que foi gerado nas etapas `/speckit-implement`, `/speckit-unit-tests` e `/speckit-component-tests`.
É um comando **interativo** e dirigido pelo usuário.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md` (Princípios II e III) e o
   `plan.md` da tarefa (checklist + Critérios de Aceite) como base da revisão.

2. **Montar a lista de arquivos alterados**:
   - Use `git status --porcelain` e `git diff --name-only` (incluindo untracked) no(s)
     repositório(s) alvo.
   - Ordene: código de produção primeiro (por camada hexagonal: models → ports → usecases →
     adapters → schemas → dto → controllers → routes → middlewares → config), depois testes.

3. **Loop de revisão** — para CADA arquivo, um de cada vez:
   - Exiba o caminho do arquivo e o **diff** completo daquele arquivo.
   - Comente objetivamente: conformidade com Princípios I/II/III, aderência ao plano,
     cobertura dos Critérios de Aceite, riscos.
   - **Aguarde o input do usuário** e trate:
     - `next` → registre o arquivo como **aprovado** e avance para o próximo.
     - `not approved` → pergunte **o que precisa mudar** naquele arquivo, aplique a correção,
       reexiba o diff atualizado e volte a aguardar (`next` / `not approved`).
     - Qualquer outra entrada → trate como instrução de ajuste para o arquivo atual.
   - Não avance sozinho: só passa de arquivo com `next`.

4. **Encerramento**: Quando todos os arquivos estiverem aprovados:
   - Rode o ESLint do(s) serviço(s) alvo para confirmar que as correções não quebraram o lint.
   - Deixe as alterações prontas para commit (staged), **sem** fazer commit.
   - Informe que a tarefa está pronta para `/speckit-test` e depois `/speckit-complete`.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_validate`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_validate` key.
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

- Lista de arquivos revisados e status (aprovado / corrigido e aprovado).
- Correções aplicadas por arquivo.
- Resultado do ESLint pós-correções.
- Próximo passo sugerido: `/speckit-test`.

## Done When

- [ ] Todos os arquivos alterados revisados um a um
- [ ] Cada arquivo aprovado explicitamente pelo usuário (`next`)
- [ ] Correções de `not approved` aplicadas e reexibidas
- [ ] ESLint limpo após as correções
- [ ] Alterações staged, sem commit
- [ ] Extension hooks dispatched or skipped according to the rules above
