# Testes Exploratórios — 002 Sistema de Gestão de Aulas

> Gerado por `/speckit-test`. Roteiro de teste **manual** (o projeto não tem QA dedicado).
> Não substitui os testes automatizados de `/speckit-unit-tests` — complementa com fluxos de
> ponta a ponta, exceções de integração e edge cases de negócio.
>
> Base de referência: `context.md` + `plan.md` (Critérios de Aceite `AC-1`..`AC-14`).

---

## 1. Escopo e pré-condições

### Serviços envolvidos

| Serviço | Papel nesta task | Porta local |
|---|---|---|
| `beach-center-bff-aulas` | **alvo** — CRUD de aula/aluno, matrícula, recálculo lazy de `ativo` | `5002` (default `env.ts`) |
| `beach-center-bff-usuarios` | fornece o enum `user_type` (agora com `PROFESSOR`); dono do login Firebase | `5000` |
| `beach-center-bff-agendamentos` | **fora de escopo** — sem chamada `aulas → agendamentos` nesta rodada | `5000` |
| MongoDB | banco compartilhado (`aulas`, `alunos`, `usuarios` na mesma instância) | `27017` |
| Firebase Auth (emulador ou projeto real) | emissão/verificação de ID token | — |

> **Pendência de infra (Princípio IV)**: `beach-center-server` ainda não expõe `aulas` no gateway
> nginx. Enquanto isso, os testes batem **direto** em `http://localhost:5002/api/v1`.

### Variáveis de ambiente mínimas (`beach-center-bff-aulas/.env`)

```
DB=mongodb://localhost:27017/beach-center
PORT=5002
FIREBASE_PROJECT_ID=<mesmo projeto de usuarios>
```

### Dados de seed

| Alias | Coleção | Campos relevantes | Uso |
|---|---|---|---|
| `ADMIN_1` | `usuarios` | `user_type: "ADMIN"`, `id_firestore` válido no Firebase | Gestão total |
| `PROF_A` | `usuarios` | `user_type: "PROFESSOR"` | Professor dono da `AULA_1` |
| `PROF_B` | `usuarios` | `user_type: "PROFESSOR"` | Professor **não** dono da `AULA_1` |
| `CLIENTE_1` | `usuarios` | `user_type: "CLIENTE"` | Usuário sem privilégio de gestão |
| `AULA_1` | `aulas` | `professor = PROF_A._id`, `capacidade_maxima = 3` | Aula base |
| `ALUNO_ATIVO` | `alunos` | `aula_id = AULA_1._id`, `vencimento_fatura` futuro, `ativo: true` | Recálculo |
| `ALUNO_VENCIDO` | `alunos` | `aula_id = AULA_1._id`, `vencimento_fatura` no passado, `ativo: true` (propositalmente desatualizado) | AC-8 |

### Como obter um ID token (pré-condição de toda requisição autenticada)

1. Autentique o usuário no Firebase (SDK client / emulador / `signInWithPassword`).
2. Use o `idToken` retornado no header: `Authorization: Bearer <idToken>`.
3. O `uid` do token deve corresponder ao `id_firestore` de um documento em `usuarios` (senão → 403, AC-12).

### Convenção das tabelas

`ID | Pré-condição | Passos | Resultado esperado`. Coluna **Auto?** indica se há teste unitário
equivalente (`sim` = coberto em `/speckit-unit-tests`; `manual` = só aqui).

---

## 2. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado | Auto? | AC |
|---|---|---|---|---|---|
| H-01 | `ADMIN_1` autenticado; `PROF_A` existe com `user_type=PROFESSOR` | `POST /aulas` com `{ classe:"Turma A", modalidade:"Beach Tenis", dias:["segunda","quarta"], hora_inicio:"2026-02-01T08:00:00Z", hora_fim:"2026-02-01T09:00:00Z", professor:"<PROF_A._id>", quadra:"<qualquer id 24-hex>", capacidade_maxima:3 }` | `201`; body `data` com todos os campos + `id`; documento criado em `aulas` | sim | AC-1 |
| H-02 | H-01 executado | `GET /aulas/<AULA_1._id>` com token de `CLIENTE_1` | `200`; retorna a aula (leitura só exige autenticação) | sim | AC-3 |
| H-03 | ≥ 2 aulas não deletadas | `GET /aulas` com token de `CLIENTE_1` | `200`; array com todas as aulas não soft-deletadas | sim | AC-3 |
| H-04 | `AULA_1` existe; `PROF_A` é o dono | `PUT /aulas/<AULA_1._id>` `{ classe:"Turma A - manhã" }` com token de `PROF_A` | `200`; `data.classe` atualizado; demais campos intactos | sim | AC-3 |
| H-05 | `AULA_1` existe | `PUT /aulas/<AULA_1._id>` `{ classe:"Turma A v2" }` com token de `ADMIN_1` | `200`; atualizado | sim | AC-3 |
| H-06 | `AULA_1` existe, sem alunos | `DELETE /aulas/<AULA_1._id>` com token de `ADMIN_1` | `200`; aula com `deleted:true`; some de `GET /aulas` | sim | AC-3 |
| H-07 | `AULA_1` com `capacidade_maxima=3` e 2 alunos não deletados | `POST /aulas/<AULA_1._id>/alunos` `{ nome:"Joao", telefone:"11987654321", vencimento_fatura:"2026-12-31T00:00:00Z" }` com token de `PROF_A` | `201`; aluno criado com `aula_id=<AULA_1._id>`, `ativo:true` | sim | AC-6 |
| H-08 | Igual H-07, mas requisição feita por `ADMIN_1` | idem H-07 | `201` | sim | AC-6, AC-4 |
| H-09 | `ALUNO_VENCIDO` no banco com `ativo:true` e `vencimento_fatura` no passado | `GET /aulas/<AULA_1._id>/alunos` com token de `PROF_A` | `200`; `ALUNO_VENCIDO` retornado com `ativo:false`; **releitura direta no Mongo** confirma `ativo:false` persistido | sim | AC-8 |
| H-10 | Aluno com `ativo:false`; `PUT` altera `vencimento_fatura` para data futura | 1) `PUT /aulas/<AULA_1._id>/alunos/<id>` `{ vencimento_fatura:"2027-01-31T00:00:00Z" }`; 2) `GET /aulas/<AULA_1._id>/alunos` | `200` nos dois; no passo 2 o aluno volta com `ativo:true` | sim | AC-9 |
| H-11 | Aluno existente na `AULA_1` | `GET`, `PUT` (`{ telefone:"11999998888" }`) e `DELETE` em `/aulas/<AULA_1._id>/alunos/<id>` por `PROF_A` | cada um `200`; `DELETE` marca `deleted:true`; aluno some da listagem | sim | AC-10 |
| H-12 | `AULA_1` lotada (3/3); um aluno removido via H-11 | `POST /aulas/<AULA_1._id>/alunos` novo aluno | `201`; a vaga liberada pelo soft-delete é reaproveitada | parcial | AC-7, AC-10 |
| H-13 | `ADMIN_1` autenticado; `CLIENTE_1` existe | `PATCH /usuarios/<CLIENTE_1._id>` `{ name, email, phone, user_type:"PROFESSOR" }` no serviço `usuarios` | `200`; `CLIENTE_1` vira `PROFESSOR`; agora pode ser `professor` de uma aula (H-01) | sim (dto) | context #2 |

---

## 3. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado | Auto? | AC |
|---|---|---|---|---|---|
| E-01 | `ADMIN_1` autenticado | `POST /aulas` com `professor` = ObjectId 24-hex que **não existe** em `usuarios` | `404`; mensagem indicando professor inválido; nenhuma aula criada | sim | AC-2 |
| E-02 | `ADMIN_1` autenticado; `CLIENTE_1` existe | `POST /aulas` com `professor = <CLIENTE_1._id>` (`user_type=CLIENTE`) | `400` (`InvalidInputError` "nao e um professor"); nenhuma aula criada | sim | AC-2 |
| E-03 | `ADMIN_1` autenticado | `POST /aulas` faltando `capacidade_maxima` (ou `dias:[]`, ou `professor:"abc"`) | `400` (`Dados inválidos`) com lista de `errors`; nada criado | sim | AC-1 |
| E-04 | `AULA_1` do `PROF_A` | `PUT /aulas/<AULA_1._id>` `{ classe:"hack" }` com token de **`PROF_B`** | `403` (`Acesso negado`); aula intacta | sim | AC-4 |
| E-05 | `AULA_1` do `PROF_A` | `POST /aulas/<AULA_1._id>/alunos` (matrícula) com token de **`PROF_B`** | `403` (`Acesso negado`) | sim | AC-4 |
| E-06 | `AULA_1` do `PROF_A` | `DELETE /aulas/<AULA_1._id>` com token de **`PROF_A`** (o próprio dono) | `403` — apagar a turma é exclusivo de `ADMIN` | sim | AC-5 |
| E-07 | `CLIENTE_1` autenticado | `POST /aulas` (qualquer body válido) | `403` — criar aula exige `ADMIN` | sim | AC-5 (variação) |
| E-08 | `AULA_1` com `capacidade_maxima=3` e **3 alunos não deletados** (independente de `ativo`) | `POST /aulas/<AULA_1._id>/alunos` novo aluno | `409` (conflito de vagas); nenhum aluno criado | sim | AC-7 |
| E-09 | `AULA_1` lotada com **3 alunos, sendo 2 `ativo:false`** | `POST /aulas/<AULA_1._id>/alunos` | `409` — inadimplência **não** libera vaga (decisão de design #1) | sim | AC-7 |
| E-10 | — | Qualquer rota de `aulas`/`alunos` **sem** header `Authorization` | `401` (`Token nao fornecido`) | sim | AC-11 |
| E-11 | — | Rota autenticada com `Authorization: Bearer <token_expirado_ou_lixo>` | `401` (`Token invalido ou expirado`) | sim | AC-11 |
| E-12 | Token Firebase válido cujo `uid` **não** tem documento em `usuarios` | Qualquer rota autenticada | `403` (`Usuario nao encontrado no sistema`) | sim | AC-12 |
| E-13 | — | `GET /aulas/<id inexistente mas 24-hex>` | `404` (`Aula nao encontrada`) | sim | AC-3 |
| E-14 | — | `GET /aulas/123` (id não é ObjectId) | `400` (`ID invalido`) — validação antes de tocar o banco | sim | AC-3 |
| E-15 | `AULA_1` existe | `POST /aulas/<AULA_1._id>/alunos` com body faltando `telefone` | `400` (`Dados inválidos`) | sim | AC-6 |
| E-16 | `AULA_1` existe | `POST /aulas/<AULA_1._id>/alunos` com `aula_id` **no body** apontando outra aula | `201`, mas o aluno é vinculado à `aula_id` **da URL** (body é ignorado / `stripUnknown`) | sim | context #1 |
| E-17 | `aula_id` da URL não existe | `POST /aulas/<id 24-hex inexistente>/alunos` body válido | `404` (`Aula nao encontrada`) | sim | AC-6 |
| E-18 | Mongo indisponível / derrubado | `GET /aulas` | `500` (`Erro interno no servidor`); log de erro no serviço; sem stack trace vazando no body | parcial | — |
| E-19 | `PROF_B` autenticado | `GET /aulas/<AULA_1._id>/alunos` (listar alunos de aula alheia) | `403` — **atenção**: leitura de aluno é protegida por ownership (mais restrito que `GET /aulas`). Confirmar se é o comportamento desejado | sim | obs. validate #2 |

---

## 4. Edge cases

| ID | Pré-condição | Passos | Resultado esperado | Auto? |
|---|---|---|---|---|
| B-01 | — | `POST /aulas` com `capacidade_maxima: 0` | `400` (deve ser `>= 1`) | sim |
| B-02 | — | `POST /aulas` com `capacidade_maxima: -5` ou `2.5` | `400` (inteiro positivo) | sim |
| B-03 | — | `POST /aulas` com `hora_fim` **anterior** a `hora_inicio` | **Comportamento atual**: aceito (`201`) — não há validação de ordem das horas. Registrar como gap se o negócio exigir | manual |
| B-04 | — | `POST /aulas` com `dias: ["segunda","segunda"]` (duplicado) ou `["Segunda-feira"]` (fora do vocabulário) | **Comportamento atual**: aceito — `dias` é `string[]` livre, sem enum. Registrar se precisar normalizar | manual |
| B-05 | Aluno com `vencimento_fatura` **exatamente hoje** (00:00 do dia corrente) | `GET /aulas/<id>/alunos` no mesmo dia às 14h | **Comportamento atual**: `ativo:false` — `recalculate` usa `$lt: new Date()` (instante), então o dia do vencimento já conta como vencido. Confirmar regra de negócio (vencimento = "pagar até o início do dia" vs "ativo durante todo o dia") | manual |
| B-06 | Aluno com `vencimento_fatura` em outro fuso / string sem `Z` (`"2026-12-31"`) | `POST`/`PUT` do aluno | `yup.date()` coage; guardar como `Date` UTC. Verificar se a data exibida bate com a esperada pelo ADMIN (risco de -1 dia por fuso `America/Sao_Paulo`) | manual |
| B-07 | `AULA_1` com 3 alunos ativos; listar | `GET /aulas/<id>/alunos` chamado **2x em paralelo** | Ambas `200`; os dois `updateMany` do recálculo são idempotentes — sem corrupção de `ativo`; contagem final consistente | manual |
| B-08 | 2 requisições de matrícula concorrentes na `AULA_1` com **1 vaga** restante | `POST .../alunos` x2 simultâneos | **Risco conhecido**: `count` + `create` não são atômicos → possível estouro de `capacidade_maxima` por corrida. Documentar como limitação (sem lock/transação nesta rodada) | manual |
| B-09 | Aula soft-deletada (`deleted:true`) | `POST /aulas/<id da aula deletada>/alunos` | `404` (`readAulaPort` filtra `deleted`) | sim |
| B-10 | Aluno soft-deletado | `GET /aulas/<aula>/alunos/<id do aluno deletado>` | `404` | sim |
| B-11 | Aluno soft-deletado | `PUT` / `DELETE` no mesmo aluno | `404` (não reativa nem re-deleta) | sim |
| B-12 | `PUT /aulas/<id>` com body **vazio** `{}` | — | `200`; nenhum campo alterado (update parcial) | sim |
| B-13 | `PUT /aulas/<id>` trocando `professor` para outro `PROFESSOR` válido | — | `200`; `professor` atualizado; a partir daí o **novo** professor passa a ser o "dono" para `requireOwnerOrAdmin` | sim |
| B-14 | `PUT /aulas/<id>` trocando `professor` para um `CLIENTE` | — | `400` (`nao e um professor`) | sim |
| B-15 | `capacidade_maxima` reduzida via `PUT` para valor **menor** que a matrícula atual (ex.: 3→1 com 3 alunos) | `PUT /aulas/<id>` `{ capacidade_maxima:1 }` | **Comportamento atual**: aceito (`200`); os 3 alunos permanecem; novas matrículas bloqueadas até cair abaixo de 1. Confirmar se o negócio quer impedir a redução abaixo do ocupado | manual |
| B-16 | `telefone` com formato livre (`"(11) 98765-4321"`, `"+55 11..."`, `"abc"`) | `POST .../alunos` | Aceito — `telefone` é `string` sem máscara/validação. Registrar se precisar validar | manual |
| B-17 | `GET /aulas` quando **não há** nenhuma aula | — | `200` com `data: []` (não `404`) | sim |

---

## 5. Checklist de regressão (fluxos vizinhos)

| ID | Área | O que verificar | Motivo do risco |
|---|---|---|---|
| R-01 | `beach-center-bff-usuarios` — `POST /auth/register` | Cadastro público continua criando `user_type:"CLIENTE"` (não é possível auto-atribuir `PROFESSOR`) | `register-auth.usecase` hardcoda `CLIENTE`; a task só mexeu no enum |
| R-02 | `usuarios` — `PATCH /usuarios/:id` | `user_type` aceita `"PROFESSOR"` além de `CLIENTE`/`ADMIN`; rejeita valores fora do enum | Alteração do `oneOf` no `updateUserDTO` |
| R-03 | `usuarios` — suíte completa | `npm test` verde (198/198) após a adição do enum | Mudança em `models`/`schema`/`dto` de serviço em produção |
| R-04 | `usuarios` — login / perfil de `PROFESSOR` | Um usuário `PROFESSOR` consegue autenticar e ler o próprio perfil (`GET /me`) como qualquer outro tipo | Novo valor de enum não pode quebrar middlewares que comparam `user_type` |
| R-05 | `beach-center-bff-agendamentos` | Nenhuma mudança de comportamento — `aulas` **não** chama `agendamentos` nesta task | Garantir que o campo `quadra` da aula é só referência (sem validação cruzada, sem bloqueio de horário) |
| R-06 | Coleção `usuarios` compartilhada | `aulas` lê `usuarios` via mirror read-only (`user.schema.ts`), sem escrever | `aulas` nunca deve criar/alterar documento de usuário |
| R-07 | Mongo — nomes de coleção | `aulas` usa `aulas` / `alunos` / `usuarios`; sem colisão com coleções de outros serviços | Banco é compartilhado |
| R-08 | `beach-center-server` (infra) | Após configurar o gateway: `aulas` responde em `/<prefixo>/api/v1/aulas` via nginx com hot-reload | Pendência registrada (Princípio IV) — validar quando `/run-server` rodar |

---

## 6. Rastreabilidade AC → cenário

| AC | Descrição resumida | Cenários | Cobertura automatizada |
|---|---|---|---|
| **AC-1** | Criar aula com professor válido | H-01, E-03, B-01, B-02 | sim (`create-aula.usecase/controller.spec`) |
| **AC-2** | Professor inexistente / não-PROFESSOR | E-01, E-02, B-14 | sim |
| **AC-3** | Ler / listar / atualizar / deletar aula | H-02..H-06, E-13, E-14, B-12, B-17 | sim |
| **AC-4** | Professor não gerencia aula alheia | E-04, E-05, H-08 | sim (`auth.middleware.spec` → `requireOwnerOrAdmin`) |
| **AC-5** | Só ADMIN deleta a aula | E-06, E-07 | sim (`auth.middleware.spec` → `requireRole`) |
| **AC-6** | Matricular dentro da capacidade | H-07, H-08, E-15, E-17 | sim |
| **AC-7** | Matricular além da capacidade | E-08, E-09, H-12 | sim (`create-aluno.usecase.spec`, `count-alunos-by-aula.adapter.spec`) |
| **AC-8** | Recálculo lazy de `ativo` na listagem | H-09, B-05 | sim (`list-alunos.usecase.spec`, `recalculate-alunos-status.adapter.spec`) |
| **AC-9** | Reativação após atualizar `vencimento_fatura` | H-10 | sim |
| **AC-10** | CRUD básico de aluno | H-11, H-12, B-10, B-11 | sim |
| **AC-11** | Rota exige autenticação | E-10, E-11 | sim (`authenticate-request.usecase.spec`, `auth.middleware.spec`) |
| **AC-12** | Usuário autenticado sem registro local | E-12 | sim |
| **AC-13** | Isolamento de camadas (ESLint) | — (gate de build) | `npx eslint .` = 0 no CI local; sem cenário manual |
| **AC-14** | Um adapter por ação | — (revisão estrutural) | verificado em `/speckit-implement` e `/speckit-validate` (15 adapters) |

**Cenários sem equivalente automatizado (só manual):** B-03, B-04, B-05, B-06, B-07, B-08, B-15, B-16, E-18 (parcial), R-01..R-08.

---

## 7. Gaps / decisões pendentes levantados pelos testes

1. **B-03** — `hora_fim < hora_inicio` é aceito. Definir se o domínio deve rejeitar.
2. **B-04** — `dias` é `string[]` livre (sem enum `segunda..domingo`, sem dedupe). Definir normalização.
3. **B-05 / B-06** — semântica de `vencimento_fatura` no dia do vencimento e sensibilidade a fuso
   (`America/Sao_Paulo`). Alinhar com quem registra o pagamento manualmente.
4. **B-08** — corrida em matrícula concorrente pode furar `capacidade_maxima` (sem transação).
   Aceitável para o volume atual? Se não, exige `findOneAndUpdate` com contador ou índice único.
5. **B-15** — reduzir `capacidade_maxima` abaixo do total já matriculado é permitido.
6. **E-19** — leitura de alunos exige ownership (mais restrito que leitura de aula). Confirmar
   intenção antes de `/speckit-complete`.
7. **R-08** — gateway nginx para `aulas` em `beach-center-server` ainda não existe (Princípio IV).
