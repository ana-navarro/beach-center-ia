---
name: "speckit-task"
description: "Cria a pasta e o contexto de uma nova tarefa de feature em tasks/, levantando as regras de negócio necessárias."
argument-hint: "Descrição da tarefa/feature a ser desenvolvida"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.1 / Comandos de Desenvolvimento"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before task)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_task` key
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

Primeiro passo do Ciclo Speckt (Princípio V da constituição). Cria o diretório e o contexto
inicial de uma nova tarefa. **NÃO** escreve código de aplicação nem testes.

1. **Ler a constituição**: Carregue `.specify/memory/constitution.md` e mantenha os Princípios I
   (fronteiras dos repositórios), II (arquitetura hexagonal) e III (qualidade) como restrições
   de contexto para todo o restante do ciclo.

2. **Derivar o slug da tarefa**: A partir da descrição do usuário, gere um nome curto em
   `kebab-case` (ex.: `agendamento-recorrente-mensalista`). Se a descrição estiver vazia ou
   ambígua, **PARE** e pergunte ao usuário o que a tarefa deve entregar.

3. **Criar o diretório**: `tasks/<NNN>-<slug>/` na raiz do repositório, onde `<NNN>` é o próximo
   número sequencial de 3 dígitos (`001`, `002`, ...) após varrer `tasks/`. Se `tasks/` não
   existir, crie.

4. **Levantar o contexto**: Crie `tasks/<NNN>-<slug>/context.md` contendo:
   - **Título e descrição** da tarefa.
   - **Serviço(s) alvo**: qual(is) repositório(s) do ecossistema (Princípio I) será(ão)
     afetado(s). Justifique.
   - **Regras de negócio conhecidas**: o que já foi informado.
   - **Perguntas em aberto**: lista numerada de tudo que ainda falta esclarecer sobre regras de
     negócio, integrações, dados e edge cases. Se houver perguntas, apresente-as ao usuário
     agora e aguarde as respostas antes de considerar a task pronta.
   - **Impacto arquitetural**: camadas hexagonais previstas (controllers/routes/usecases/
     models/ports/adapters/schemas).

5. **Não avançar**: Este comando termina aqui. Não gere plano, código ou testes.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_task`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_task` key.
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

- Caminho do diretório e do `context.md` criados.
- Serviço(s) alvo identificado(s).
- Lista de perguntas em aberto pendentes de resposta do usuário.
- Próximo passo sugerido: `/speckit-plan`.

## Done When

- [ ] Diretório `tasks/<NNN>-<slug>/` criado
- [ ] `context.md` gerado com serviço alvo, regras conhecidas e perguntas em aberto
- [ ] Perguntas de regra de negócio apresentadas ao usuário (se houver)
- [ ] Nenhum arquivo de código/teste de aplicação criado ou modificado
- [ ] Extension hooks dispatched or skipped according to the rules above
