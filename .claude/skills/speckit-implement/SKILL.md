---
name: "speckit-implement"
description: "Implementa o código-fonte a partir do checklist do plano, com conformidade estrita ao ESLint e sem realizar commit."
argument-hint: "Slug da tarefa ou filtro de itens do checklist"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.3 / Comandos de Desenvolvimento (override do speckit-implement padrão)"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before implement)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_implement` key
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

Terceiro passo do Ciclo Speckt (Princípio V). Lê o `plan.md` da tarefa e implementa o código.
**NÃO** gera testes (isso é `/speckit-unit-tests` e `/speckit-component-tests`) e **NÃO** faz
commit.

> Este comando substitui o comportamento do `speckit-implement` padrão do spec-kit para este projeto.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md`. Cumpra:
   - **Princípio I**: só edite arquivos dentro do(s) repositório(s) alvo definido(s) no plano.
   - **Princípio II**: respeite a Arquitetura Hexagonal.
     - `domain/usecases/` contém TODAS as regras de negócio; nunca acessam schemas, só models.
     - `domain` fala com `infra` APENAS via ports; `applications` fala com `domain` APENAS via ports.
     - `infra/adapters/` NÃO contêm regra de negócio; **um adapter por verbo/ação**.
   - **Princípio III**: todo código gerado MUST passar no ESLint do projeto.

2. **Localizar a tarefa**: Identifique `tasks/<NNN>-<slug>/` (argumento ou a mais recente com
   `plan.md`). Leia `plan.md` (checklist + Mapa Arquitetural + Critérios de Aceite). Se não
   houver `plan.md`, **PARE** e peça para rodar `/speckit-plan`.

3. **Implementar por item do checklist**, na ordem de dependência:
   - Crie/edite os arquivos nos caminhos exatos do plano.
   - Siga o estilo e as convenções do código existente no serviço alvo.
   - Marque cada item concluído como `- [x]` no `plan.md`.

4. **Conformidade ESLint (obrigatória)**:
   - Rode o linter do serviço alvo (`npm run lint` / `npx eslint .` conforme o projeto).
   - Corrija todos os erros. Warnings devem ser resolvidos ou justificados no relatório.
   - Não use `eslint-disable` sem justificativa explícita no relatório final.

5. **Sem commit**: NÃO execute `git commit`, `git add -A` com intenção de commit, nem `git push`.
   Apenas altere/crie arquivos localmente.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_implement`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_implement` key.
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

- Lista de arquivos criados/alterados, agrupados por camada hexagonal.
- Itens do checklist concluídos vs. pendentes.
- Resultado do ESLint (limpo / itens corrigidos / justificativas).
- Próximo passo sugerido: `/speckit-unit-tests` e `/speckit-component-tests`.

## Done When

- [ ] Itens do checklist do `plan.md` implementados e marcados `[x]`
- [ ] Código alocado nas camadas hexagonais corretas, dentro do repositório alvo
- [ ] ESLint sem erros no serviço alvo
- [ ] Nenhum commit ou push realizado
- [ ] Extension hooks dispatched or skipped according to the rules above
