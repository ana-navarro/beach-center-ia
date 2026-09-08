---
name: "speckit-complete"
description: "Quality gate local (ESLint, testes unitários e de componente, cobertura >=80%), depois commits Conventional, push e acompanhamento da CI."
argument-hint: "Slug da tarefa (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.7 / Comandos de Desenvolvimento"
user-invocable: true
disable-model-invocation: true
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before complete)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_complete` key
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

Sétimo passo do Ciclo Speckt (Princípio V). Executa o **Quality Gate Local** e só então
commita e faz push dos repositórios afetados. **NÃO** gera documentação — isso é
`/speckit-documentation` (passo 8).

1. **Ler contexto**: Carregue `.specify/memory/constitution.md`, o `plan.md` e (se existir) o
   `exploratory-tests.md` da tarefa. Identifique todos os repositórios do ecossistema afetados.

2. **Quality Gate Local (BLOCKER)** — para CADA repositório afetado, rodar e confirmar:
   1. **ESLint sem erros** (`npm run lint` / `npx eslint .`).
   2. **Todos os testes unitários passando** (`Jest`/`Vitest`).
   3. **Todos os testes de componente passando** (`Cypress` + `Cucumber`, headless).
   4. **Cobertura de código >= 80%** (relatório de coverage dos testes unitários).
   - Se **qualquer** item falhar: **PARE**. Reporte exatamente o que falhou (com a saída
     relevante), corrija ou peça correção, e reinicie o gate. Não prossiga para o commit.

3. **Commits (Conventional Commits)**:
   - Só após o gate 100% verde.
   - Um commit coeso por repositório afetado, mensagem no padrão Conventional Commits
     (`feat:`, `fix:`, `test:`, `docs:`, `chore:` ...), com escopo quando fizer sentido.
   - Finalize as mensagens com:
     `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>`
   - Se o branch atual for o default (`main`/`master`), crie um branch de feature antes de commitar.

4. **Push e CI**:
   - `git push` em cada repositório afetado.
   - Aguarde a pipeline de CI remota confirmar sucesso (inclui ESLint e testes obrigatórios).
   - Se a CI falhar, reporte o log, corrija e repita o gate local antes de novo push.

5. **Encerramento**: Não gere documentação aqui. Informe que o próximo passo é
   `/speckit-documentation` para mapear os endpoints e atualizar
   `beach-center-documentation/`.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_complete`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_complete` key.
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

- Resultado do Quality Gate Local por repositório (ESLint / unit / component / cobertura %).
- Commits criados (hash + mensagem) e branches.
- Status da CI por repositório.
- Próximo passo sugerido: `/speckit-documentation`.

## Done When

- [ ] Quality Gate Local 100% verde em todos os repositórios afetados (ESLint, unit, component, cobertura >= 80%)
- [ ] Commits em Conventional Commits e push realizados
- [ ] CI remota confirmada com sucesso
- [ ] Nenhuma documentação gerada por este comando
- [ ] Extension hooks dispatched or skipped according to the rules above
