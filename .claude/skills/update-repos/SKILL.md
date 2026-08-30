---
name: "update-repos"
description: "Sugere os comandos para atualizar dependências e sincronizar as branches dos repositórios do ecossistema Beach Center."
argument-hint: "Nome(s) de repositório(s) alvo (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution: Comandos de Utilitários"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before update_repos)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_update_repos` key
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

Comando utilitário (Princípio I). **Sugere** comandos para atualizar dependências e sincronizar
branches. Não executa mudanças destrutivas nem `push` sem confirmação do usuário.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md` e liste os repositórios do
   ecossistema (Princípio I). Se o usuário passou nomes, restrinja a eles; senão, cubra todos.

2. **Diagnóstico (somente leitura)** por repositório, quando acessível localmente:
   - `git fetch --all --prune`
   - `git status -sb` e branch atual vs. `origin/main`
   - Gerenciador de pacotes detectado (`package-lock.json` → npm, `yarn.lock` → yarn,
     `pnpm-lock.yaml` → pnpm) e desatualizações (`npm outdated`).

3. **Sugerir (NÃO executar)** um bloco de comandos por repositório:
   - Sincronizar branch: `git switch main && git pull --ff-only && git switch - && git rebase main`
   - Atualizar deps: `npm ci` (reprodutível) ou `npm update` / bump pontual + `npm install`
   - Rodar `npm run lint` e `npm test` após atualizar (Princípio III).
   - Auditar: `npm audit` / `npm audit fix` (revisar antes de aplicar).

4. Apresente tudo como checklist copiável, ordenado por repositório, destacando quais comandos
   são potencialmente destrutivos (rebase, audit fix) e exigem revisão.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_update_repos`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_update_repos` key.
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

- Tabela `repo | branch atual | atrás/à frente de origin/main | deps desatualizadas`.
- Blocos de comandos sugeridos por repositório.

## Done When

- [ ] Repositórios do ecossistema identificados
- [ ] Diagnóstico somente-leitura apresentado
- [ ] Comandos sugeridos (não executados) em checklist copiável
- [ ] Extension hooks dispatched or skipped according to the rules above
