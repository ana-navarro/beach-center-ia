# Testes exploratórios — Task 007: Atualização do Constitution (`injection` + `llm-engine`)

> Gerado por `/speckit-test`. **Task doc-only** — não há sistema em execução para testar. Este
> roteiro é uma **conferência de leitura/revisão** do arquivo alterado
> `.specify/memory/constitution.md`, validando o DoD da US01 e os Critérios de Aceite do
> `plan.md` (AC-1..AC-12).
>
> **Nenhum cenário tem cobertura automatizada** (não há código; `/speckit-unit-tests` e
> `/speckit-component-tests` = N/A). A "execução" de cada cenário é: abrir o arquivo, localizar
> a passagem, conferir o texto.

## Escopo e pré-condições

| Item | Valor |
|---|---|
| **Repositório alvo** | `beach-center-ia` (root) |
| **Arquivo alterado** | `.specify/memory/constitution.md` (único) |
| **Artefatos de apoio** | `tasks/007-constitution-injection-llm-engine/{context.md, plan.md}` |
| **Seed / dados** | N/A — não há banco, serviço ou runtime |
| **Perfis / auth** | N/A |
| **Ferramenta** | editor de texto + `git diff` + um renderizador de Markdown (GitHub / VS Code preview) |
| **Baseline** | Constitution **v1.3.1** (antes da task); esperado após a task: **v1.4.0** |
| **Enunciado / DoD** | US01 — 5 itens de Definition of Done (ver `context.md`) |

### Referência rápida — o que a task adicionou

- **Princípio I**: 2 bullets novos na lista *Backend (BE - services)* (`beach-center-bff-injection`,
  `beach-center-bff-llm-engine`) + frase no bloco **ATENÇÃO** negando o rótulo "BFF/Backend for
  Frontend" para os dois.
- **Princípio VI** (novo): *"Arquitetura Orientada a Eventos para Validação de Comprovantes"* —
  7 blocos + bloco de fluxo sequencial de 8 passos.
- **Sync Impact Report** + **linha de versão** do rodapé atualizados (`1.3.1 → 1.4.0`, MINOR).

---

## 1. Caminhos felizes — verificação por Critério de Aceite

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **V-1** | Arquivo pós-implementação aberto | Abrir o **Princípio I**, localizar a lista *Backend (BE - services)* | A lista contém `services/beach-center-bff-injection` (responsabilidade: ingestão/upload/validação estática/envio ao storage/publicação na fila) e `services/beach-center-bff-llm-engine` (responsabilidade: worker de validação visual assíncrona por IA). Texto fiel ao enunciado. **(AC-1)** |
| **V-2** | idem | Nos mesmos bullets, procurar a **stack** de cada serviço | `beach-center-bff-injection` = **Node.js/TypeScript**; `beach-center-bff-llm-engine` = **Node.js/TypeScript, Moondream, Docker**. Nenhum dos dois é descrito como "Backend for Frontend". **(AC-2)** |
| **V-3** | idem | Percorrer os títulos `### ` da seção *Core Principles* | Existe **`### VI. Arquitetura Orientada a Eventos para Validação de Comprovantes`**, posicionado **depois** do Princípio V e **antes** da seção `## Comandos de Desenvolvimento`. **(AC-3)** |
| **V-4** | Princípio VI aberto | Localizar o **bloco de fluxo sequencial** (bloco de código) | Há 8 passos numerados: (1) Front→REST→injection, (2) injection→MinIO (putObject), (3) injection→agendamentos (REST interna, status "Em Análise" + url_arquivo), (4) injection→Fila (`{id_agendamento, url_arquivo}`), (5) llm-engine←Fila (consume), (6) llm-engine→MinIO (getObject), (7) llm-engine→Moondream (inferência), (8) llm-engine→agendamentos (REST interna: "Rejeitado"+libera quadra \| mantém "Em Análise"). Abaixo do bloco: frase afirmando que a resposta ao front (passo 1) retorna **antes** dos passos 5–8. **(AC-4)** |
| **V-5** | Princípio VI, bloco "Regras de comunicação" | Ler a 1ª regra | Afirmação normativa: a comunicação `injection ↔ llm-engine` **MUST** ser por mensageria assíncrona (**RabbitMQ** ou **BullMQ/Redis**) e **MUST NOT** ser REST síncrona. **(AC-5)** |
| **V-6** | Princípio VI, bloco "Storage — MinIO" | Ler o parágrafo | O **MinIO** (S3-compatible, self-hosted) é o **storage padrão** de comprovantes e **substitui o Google Drive**; a **justificativa registrada** é o **rate limit** da API do Drive. **(AC-6)** |
| **V-7** | Princípio VI | Procurar como `injection`/`llm-engine` gravam o status | Texto explícito: ambos gravam **via REST interna (`x-api-key`) do `beach-center-bff-agendamentos`** (dono único da coleção); **nenhum** escreve direto na base nem mantém coleção paralela de status. **(AC-7)** |
| **V-8** | Topo e rodapé do arquivo | Ler o **Sync Impact Report** e a **linha de versão** | Sync Impact Report: `Version change: 1.3.1 → 1.4.0`; rationale marcado **MINOR**; `Modified principles:` cita o Princípio I; `Added sections:` cita o Princípio VI; a entrada `1.3.0 → 1.3.1 (PATCH)` está preservada no *Prior amendment history*. Rodapé: `**Version**: 1.4.0 | **Ratified**: 2026-08-30 | **Last Amended**: 2026-09-10`. **(AC-8)** |
| **V-9** | Princípio VI, bloco "Conformidade com os demais princípios" | Ler o parágrafo | Afirma que `injection` e `llm-engine` (Node.js/TS) **MUST** seguir o Princípio II (Hexagonal) e o Princípio III (ESLint + cobertura ≥ 80%); o consumo de fila entra por `applications/controllers/` (controllers de evento). **Não** concede isenção de testes. **(AC-9)** |
| **V-10** | Terminal no repo `beach-center-ia` | `git status --porcelain` | O único arquivo modificado é `.specify/memory/constitution.md`. `beach-center-server/docker-compose.dev.yml`, os dois repos novos e `beach-center-documentation/` **não** aparecem como modificados (só como untracked pré-existentes, quando aplicável). **(AC-10)** |
| **V-11** | Princípio VI, bloco "Notas (não-normativas)" | Ler as notas | Há nota mapeando **"Em Análise" ≈ `waiting_approve`** e **"Documento Inválido / Rejeitado" ≈ `rejected`** (+ libera a quadra), com referência à task 006a. **(AC-11)** |
| **V-12** | Arquivo inteiro + preview de Markdown | Renderizar o `.md`; procurar contagens de princípios / cross-refs | Markdown válido: headings `I`→`VI` em ordem, 1 bloco de código fechado, a tabela do rodapé (`Version | Ratified | Last Amended`) intacta, o `---` separando *Core Principles* de *Comandos de Desenvolvimento* presente. Nenhuma passagem passou a mentir por causa do Princípio VI (não há "os cinco princípios", "Princípios I a V", etc.). **(AC-12)** |

---

## 2. Fluxos de exceção — o que poderia estar errado no documento

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **E-1** | Princípio VI | Procurar a palavra "BFF" / "Backend for Frontend" no texto novo, usada no **sentido do pattern** para `injection` | **Não** deve haver — `injection` é descrito como "serviço de ingestão / porta de entrada de upload", nunca como camada de composição para a UI. O único uso de "Backend For Frontend" continua sendo a negação no bloco ATENÇÃO. |
| **E-2** | Sync Impact Report | Conferir a classificação do bump | Deve ser **MINOR** (regra da Governança: "MINOR: adição de novo princípio"). **Não** pode estar como PATCH nem MAJOR. |
| **E-3** | Rodapé + Sync Impact Report | Comparar a versão nos dois lugares | Ambos dizem **1.4.0**. Não pode haver divergência (ex.: rodapé 1.4.0 e Sync Impact 1.3.2). |
| **E-4** | Princípio VI, regra de comunicação | Verificar se a regra fixa **uma** tecnologia de fila | Deve dizer **"RabbitMQ ou BullMQ/Redis"** (as duas aceitáveis; escolha na implementação). Se estiver amarrado a só uma marca como `MUST`, é desvio do plano (Q4). |
| **E-5** | Princípio VI, deveres de `injection` | Conferir os limites da validação estática | Deve constar **exatamente**: formatos `PDF`, `JPG`, `PNG` e tamanho **máx. 5 MB**. Valores diferentes (ex.: 10 MB, incluir GIF) = erro. |
| **E-6** | Princípio VI, resultado da IA | Ler o ramo "IA responde SIM" | Deve **manter "Em Análise"** para revisão manual do admin — **não** deve dizer "aprova automaticamente". Ler o ramo "IA responde NÃO" → **"Rejeitado" + libera a quadra**. |
| **E-7** | Princípio VI, item 3 (fronteira de dados) | Verificar se o texto autoriza banco compartilhado ou coleção duplicada | Deve **proibir** — só REST interna do `agendamentos`. Qualquer menção a "coleção própria de status" / "escrita direta" contradiz C5/AC-7. |
| **E-8** | Prior amendment history | Conferir se o histórico anterior foi preservado | As entradas `1.2.0 → 1.3.0`, `1.2.0`, `1.1.1`, `1.0.0 → 1.1.0` continuam presentes, com a nova `1.3.0 → 1.3.1 (PATCH)` inserida no topo do histórico. Nenhuma entrada apagada. |
| **E-9** | Princípio VI, intro | Ler a primeira frase do princípio | Deve afirmar que a validação por IA roda **fora do caminho síncrono** e que o **front-end MUST NOT aguardar** a inferência. Se essa afirmação estiver ausente, AC-4 falha parcialmente. |

---

## 3. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **X-1** | Renderização em GitHub | Abrir o arquivo no GitHub (ou preview) | O bloco de fluxo de 8 passos renderiza como bloco de código monoespaçado (fence ` ``` ` aberto e fechado); o alinhamento com espaços não quebra o layout do resto da página. |
| **X-2** | Comentário HTML do Sync Impact Report | Renderizar | O bloco `<!-- ... -->` no topo **não** aparece no HTML renderizado (continua sendo comentário); a primeira linha visível é `# Constituição do Projeto Beach Center`. |
| **X-3** | Acentuação / encoding | Abrir com `file` / editor mostrando encoding | UTF-8 preservado; "Análise", "inferência", "Comprovantes", "não-normativas" sem mojibake. Sem BOM novo. |
| **X-4** | Fim de arquivo | `git diff` da última linha | A linha de versão continua sendo a **última** linha; o `\ No newline at end of file` é o mesmo comportamento pré-existente (o arquivo nunca teve newline final) — não é regressão. |
| **X-5** | Numeração dos princípios | Ler os títulos `### ` | Sequência `I, II, III, IV, V, VI` sem pular nem repetir; nenhum princípio existente foi renumerado. |
| **X-6** | Termos EN/PT-RFC2119 | Buscar `MUST`, `MUST NOT` no Princípio VI | Uso consistente com o resto do documento (que já mistura `MUST` inglês com texto PT). "MUST NOT" aparece na intro e na regra de comunicação. |
| **X-7** | Referência cruzada | Seguir os "(Ver Princípio VI.)" dos bullets do Princípio I | Apontam para uma seção que **existe** (o Princípio VI). Nenhum link/âncora quebrada. |

---

## 4. Checklist de regressão — o que a mudança pode afetar

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **R-1** | Skills do Speckt que leem a constituição (`.claude/skills/speckit-*`) | Reler `constitution.md` como faria o `/speckit-plan` / `/speckit-implement` | O documento continua parseável e coerente; os Princípios I–V permanecem **textualmente inalterados** (exceto os 2 bullets + 1 frase no I). Nenhuma instrução operacional dos comandos mudou. |
| **R-2** | Seção "Comandos de Desenvolvimento" | Diff dessa seção | **Sem alterações** — a task não tocou nos comandos `/speckit-*`. |
| **R-3** | Seção "Governance" | Diff | **Sem alterações**. A linha que cita "a regra contra uso de BFFs (Princípio I)" continua válida (reforçada, não contradita). |
| **R-4** | `beach-center-documentation/README.md` | Verificar se precisa de linha para `injection`/`llm-engine` | **Não** nesta task — os 2 repos não têm endpoints/documentação ainda. Sem regressão. |
| **R-5** | `beach-center-server/docker-compose.dev.yml` | `git diff` | **Sem alterações** (C3). MinIO/RabbitMQ/Redis continuam ausentes do compose — é task futura, e o Princípio VI registra isso como *deferred*. |
| **R-6** | `beach-center-bff-pagamentos` / `beach-center-bff-agendamentos` | `git status` nesses repos | Limpos (na `main`, pós-006a/006b). A task 007 **não** altera o fluxo de comprovante atual — só documenta o alvo futuro. |
| **R-7** | Memória do projeto (`sistema-pagamento-futuro`) | Reler | Continua válida como "direção futura"; o Princípio VI a torna parcialmente oficial (Moondream + MinIO em vez de llama3.2-vision + Drive). Sem contradição factual no `constitution.md`. |
| **R-8** | Consumidores externos do texto (onboarding, PRs) | Ler o Princípio VI de ponta a ponta como um dev novo | O fluxo REST→MinIO→Fila→IA fica compreensível sem contexto extra; os papéis de `injection` e `llm-engine` não se confundem; fica claro o que **não** está pronto (infra, bootstrap). |

---

## Rastreabilidade AC → cenário

| Critério de Aceite | Cenário(s) | Cobertura automatizada |
|---|---|---|
| AC-1 — repos no Princípio I | V-1 | — (doc) |
| AC-2 — stacks declaradas / não-BFF | V-2, E-1 | — |
| AC-3 — Princípio VI numerado e posicionado | V-3, X-5 | — |
| AC-4 — fluxo assíncrono mapeado + front não aguarda | V-4, E-9 | — |
| AC-5 — mensageria-only entre injection↔llm-engine | V-5, E-4 | — |
| AC-6 — MinIO substitui Drive (rate limit) | V-6 | — |
| AC-7 — fronteira de dados via REST interna | V-7, E-7 | — |
| AC-8 — emenda registrada (Sync Impact + versão) | V-8, E-2, E-3, E-8 | — |
| AC-9 — conformidade com II/III, sem isenção de testes | V-9 | — |
| AC-10 — só constitution.md alterado | V-10, R-2, R-3, R-5 | — |
| AC-11 — nota de mapeamento com a 006a | V-11, E-6 | — |
| AC-12 — coerência interna / Markdown válido | V-12, X-1, X-2, X-5, X-7, R-1 | — |

**Todos os 12 ACs cobertos.** Nenhum cenário tem equivalente automatizado — é uma task de
documentação; a verificação é 100% por leitura/`git diff`.

## Contagem

| Categoria | Qtd |
|---|---|
| Caminhos felizes (V-*) | 12 |
| Fluxos de exceção (E-*) | 9 |
| Edge cases (X-*) | 7 |
| Regressão (R-*) | 8 |
| **Total** | **36** |

## Próximo passo

`/speckit-complete` — quality gate (aqui: ESLint/testes = N/A; a "validação" é a conferência
deste roteiro + o `git diff`), commit e push do `beach-center-ia`.
