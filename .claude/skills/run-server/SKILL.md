---
name: "run-server"
description: "Ajusta e executa os scripts de beach-center-server para subir os containers do ambiente local com hot-reloading."
argument-hint: "Serviços específicos a subir (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution: Comandos de Utilitários / Princípio IV"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before run_server)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_run_server` key
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

Comando utilitário. Prepara e sobe o ambiente local via `beach-center-server`, respeitando o
Princípio IV (volumes Docker para hot-reloading).

1. **Localizar `beach-center-server`**: na raiz atual ou em repositório irmão
   (`../beach-center-server`). Se não encontrar, **PARE** e peça o caminho.

2. **Inspecionar** (somente leitura): `docker-compose*.yml` / `compose*.yaml`, scripts em
   `package.json`/`Makefile`/`scripts/`, e `.env`/`.env.example`.

3. **Verificar/ajustar hot-reloading (Princípio IV)**:
   - Cada serviço de código MUST montar o código-fonte como **volume** (`./service:/app` +
     `/app/node_modules`), não `COPY` imutável.
   - Comando do container em modo watch (`npm run dev`, `nodemon`, `ts-node-dev`, Vite).
   - Se algum serviço não tiver volume/watch, proponha o patch mínimo no compose e peça
     confirmação antes de editar.

4. **Preparar env**: se faltar `.env`, copie de `.env.example` e liste as variáveis a preencher.

5. **Subir os containers**:
   - `docker compose up -d --build` (ou o script equivalente do repo; subconjunto de serviços
     se o usuário especificou).
   - Aguarde ficarem `healthy`; mostre `docker compose ps`.
   - Em falha, mostre `docker compose logs --tail=50 <serviço>` e diagnostique.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_run_server`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_run_server` key.
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

- Ajustes feitos no compose (se houver) e confirmação de hot-reloading por serviço.
- `docker compose ps` final (portas e health).
- Comandos úteis: logs, restart de um serviço, `down`.

## Done When

- [ ] `beach-center-server` localizado e inspecionado
- [ ] Hot-reloading via volumes verificado/ajustado (Princípio IV)
- [ ] Containers no ar e saudáveis (ou falha diagnosticada com logs)
- [ ] Extension hooks dispatched or skipped according to the rules above
