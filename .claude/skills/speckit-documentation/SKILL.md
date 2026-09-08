---
name: "speckit-documentation"
description: "Mapeia todos os endpoints do repositório e gera/atualiza a documentação em PT-BR, um arquivo por rota em beach-center-documentation/[nome-repo]/[nome-rota].md."
argument-hint: "Nome do repositório alvo ou rota específica (opcional)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "beach-center"
  source: "constitution:V.8 / Comandos de Desenvolvimento"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before documentation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_documentation` key
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

Oitavo e último passo do Ciclo Speckt (Princípio V). Roda **depois** de `/speckit-complete`.
Atualiza e cria a documentação da API baseando-se no **código atual** do repositório. **NÃO**
altera código de aplicação nem testes; só escreve arquivos `.md` de documentação.

1. **Ler contexto**: Carregue `.specify/memory/constitution.md` (Princípio I para a lista de
   repositórios do ecossistema, Princípio II para a leitura da arquitetura hexagonal).

2. **Definir o(s) repositório(s) alvo**:
   - Se o usuário passou um nome de repo ou rota, restrinja a ele.
   - Senão, use o(s) repositório(s) afetado(s) na tarefa atual; se não houver tarefa em
     andamento, cubra todos os serviços de backend do ecossistema.

3. **Mapear TODOS os endpoints existentes** em cada repositório alvo (obrigatório):
   - Varra `applications/routes/` (definição dos endpoints) e os `controllers/` associados.
   - Para cada endpoint, levante: método HTTP, path (incluindo prefixos/versionamento),
     middlewares/guards de auth aplicados, DTO de entrada (`applications/dto/`), DTO/forma da
     resposta, códigos de status possíveis e o(s) usecase(s) de `domain/usecases/` acionado(s).
   - Não invente rotas: a lista MUST refletir exatamente o que está no código.

4. **Agrupar por rota**: agrupe os endpoints por recurso/rota lógica (ex.: todos os métodos de
   `/agendamentos` → um único documento `agendamento`). O nome do arquivo é o nome da rota em
   `kebab-case`, no singular quando fizer sentido.

5. **Gerar/atualizar a documentação** — um arquivo por rota, redigido **em português**, em:
   `beach-center-documentation/[nome-repo]/[nome-rota].md`
   - `[nome-repo]` é o nome da pasta do serviço (ex.: `beach-center-bff-agendamentos`).
   - Exemplo de caminho: `beach-center-documentation/beach-center-bff-agendamentos/agendamento.md`.
   - Crie os diretórios que faltarem. Se o arquivo já existir, **atualize-o** preservando notas
     manuais relevantes e removendo o que não corresponde mais ao código.
   - Estrutura de cada documento:
     - **Visão geral**: o que a rota representa e qual serviço a expõe.
     - **Autenticação/autorização**: guards e perfis exigidos.
     - **Endpoints**: uma subseção por método+path, cada uma com:
       - Descrição e regra de negócio (usecase acionado).
       - Parâmetros de path/query e corpo da requisição (tabela: campo, tipo, obrigatório, descrição).
       - Corpo da resposta de sucesso (exemplo de payload) e código HTTP.
       - Erros possíveis (código HTTP + causa).
       - Exemplo de chamada (`curl` ou equivalente).
     - **Referências**: caminhos dos arquivos de rota/controller/usecase no código.

6. **Sem commit**: apenas cria/atualiza os arquivos de documentação localmente.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_documentation`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_documentation` key.
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

- Repositório(s) documentado(s).
- Tabela `método | path | rota (arquivo) | usecase`.
- Lista de arquivos criados/atualizados em `beach-center-documentation/[nome-repo]/`.
- Endpoints que não puderam ser totalmente mapeados (e por quê), se houver.

## Done When

- [ ] Todos os endpoints do(s) repositório(s) alvo mapeados a partir do código atual
- [ ] Um arquivo `.md` por rota em `beach-center-documentation/[nome-repo]/[nome-rota].md`
- [ ] Documentação redigida em português
- [ ] Documentos existentes atualizados (não duplicados) conforme o código
- [ ] Nenhum código de aplicação ou teste alterado
- [ ] Extension hooks dispatched or skipped according to the rules above
