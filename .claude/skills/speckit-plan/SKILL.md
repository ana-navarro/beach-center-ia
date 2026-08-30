---
name: "speckit-plan"
description: "Gera o checklist de implementação detalhado e os Critérios de Aceite formais dentro da pasta da tarefa."
argument-hint: "Slug da tarefa ou orientações adicionais para o planejamento"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.2 / Comandos de Desenvolvimento (override do speckit-plan padrão)"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before plan)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_plan` key
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

Segundo passo do Ciclo Speckt (Princípio V). MUST rodar **depois** de `/speckit-task` e **antes**
de `/speckit-implement`. Produz o planejamento e os Critérios de Aceite. **NÃO** escreve código.

> Este comando substitui o comportamento do `speckit-plan` padrão do spec-kit para este projeto.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md`. Os Princípios I, II e III
   são gates obrigatórios do plano.

2. **Localizar a tarefa**: Identifique a pasta `tasks/<NNN>-<slug>/` (pelo argumento, ou a mais
   recente sem `plan.md`). Leia `context.md`. Se ainda houver perguntas de regra de negócio em
   aberto, **PARE** e peça as respostas antes de planejar.

3. **Gerar `tasks/<NNN>-<slug>/plan.md`** com as seções:

   ### Contexto Técnico
   - Serviço(s) alvo e justificativa da fronteira (Princípio I).
   - Stack e bibliotecas envolvidas.

   ### Constitution Check
   - Para cada princípio relevante (I, II, III, IV): CONFORME / VIOLAÇÃO + justificativa.
   - Se houver violação sem justificativa aprovada, marque **ERRO** e não prossiga.

   ### Mapa Arquitetural (Hexagonal)
   - Liste os arquivos a criar/alterar por camada: `controllers/`, `routes/`, `middlewares/`,
     `dto/`, `domain/usecases/`, `domain/models/`, `domain/ports/`, `infra/ports/`,
     `infra/adapters/` (um adapter por verbo/ação), `infra/schemas/`, `config/`.

   ### Checklist de Implementação
   - Lista de itens `- [ ]` em ordem de dependência, cada um com caminho de arquivo exato.
   - Ordem sugerida: models → ports (domain) → usecases → ports (infra) → adapters → schemas →
     dto → controllers → routes → middlewares → wiring/config.

   ### Critérios de Aceite (formais)
   - Numerados (`AC-1`, `AC-2`, ...), no formato **Given / When / Then**.
   - Devem cobrir caminho feliz, fluxos de exceção e regras de negócio do beach tennis.
   - São a base para `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.

4. **Não avançar**: Este comando termina aqui. Não implemente nada.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_plan`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_plan` key.
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

- Caminho do `plan.md` gerado.
- Resultado do Constitution Check.
- Quantidade de itens no checklist e de Critérios de Aceite.
- Próximo passo sugerido: `/speckit-implement`.

## Done When

- [ ] `tasks/<NNN>-<slug>/plan.md` criado
- [ ] Constitution Check preenchido sem violações não justificadas
- [ ] Checklist de implementação com caminhos de arquivo por camada hexagonal
- [ ] Critérios de Aceite formais no formato Given/When/Then
- [ ] Nenhum arquivo de código de aplicação criado ou modificado
- [ ] Extension hooks dispatched or skipped according to the rules above
