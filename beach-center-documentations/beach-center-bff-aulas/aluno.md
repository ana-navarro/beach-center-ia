# Aluno

## Visão geral

O recurso `aluno` representa um aluno matriculado numa aula. É **sempre** acessado de forma
aninhada sob uma aula — `/aulas/:aula_id/alunos` — reforçando que um aluno não existe sem uma
turma. É exposto pelo microsserviço `beach-center-bff-aulas` sob o prefixo global `/api/v1`
(`src/main.ts`, porta padrão **5002**). O sub-roteador usa `Router({ mergeParams: true })` para
enxergar o `:aula_id` do mount pai.

O modelo de domínio (`IAluno`):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (ObjectId) | Identificador do aluno |
| `aula_id` | string (ObjectId) | Aula à qual o aluno pertence — vem **sempre** da URL, nunca do corpo |
| `nome` | string | Nome do aluno |
| `telefone` | string | Telefone — texto livre, sem máscara nem validação de formato |
| `vencimento_fatura` | string (ISO date) | Data de vencimento da mensalidade — atualização **100% manual** (sem integração com `pagamentos`) |
| `ativo` | boolean | Recalculado de forma **lazy** a cada listagem (ver abaixo) |

O campo interno `deleted` (soft delete manual) não é exposto.

### Regra da capacidade (`capacidade_maxima` da aula)

Na matrícula (`POST`), conta-se o total de alunos **não deletados** da aula, **independente do
campo `ativo`**. Inadimplência (`ativo = false`) **não** libera vaga — só a remoção explícita
(`DELETE`, soft delete) devolve a vaga. Matricular com `total >= capacidade_maxima` → `409`.

### Recálculo lazy de `ativo` (`GET /aulas/:aula_id/alunos`)

Antes de listar, o `ListAlunosUsecase` chama `RecalculateAlunosStatusAdapter`, que roda dois
`updateMany` escopados por `aula_id` (mesmo padrão de `markPastSchedulingsUnavailable` em
`agendamentos`):

1. `vencimento_fatura < agora` **e** `ativo = true` → passa para `ativo = false`.
2. `vencimento_fatura >= agora` **e** `ativo = false` → volta para `ativo = true` (reativação após
   o ADMIN registrar o pagamento atualizando a data).

A comparação usa `new Date()` (instante da requisição) — um aluno cujo `vencimento_fatura` é
exatamente o dia corrente 00:00 já é considerado vencido no mesmo dia. A alteração é **persistida**;
a resposta já reflete o estado recalculado.

## Autenticação/autorização

**Todos** os endpoints exigem `authMiddleware` + `requireOwnerOrAdmin`.

| Guard | Regra | Erros |
|---|---|---|
| `authMiddleware` | `Authorization: Bearer <idToken>` Firebase válido + `uid` presente na cópia local de `usuarios` | `401` **"Token nao fornecido"**, `401` **"Token invalido ou expirado"**, `403` **"Usuario nao encontrado no sistema"** |
| `requireOwnerOrAdmin` | Libera `ADMIN`; senão carrega a aula pelo `:aula_id` da URL (via `container.readAula`) e exige `aula.professor === databaseUser.id` | `401` **"Token invalido ou expirado"** (sem `databaseUser`); `403` **"Acesso negado"** (sem `aula_id`, ou professor não é o dono); `400` **"ID invalido"** / `404` **"Aula nao encontrada"** (propagados quando o `:aula_id` é malformado ou inexistente) |

> A leitura de alunos (`GET`) é **mais restrita** que a leitura de aula: `GET /aulas/:aula_id/alunos`
> exige `ADMIN` ou o professor dono da aula, enquanto `GET /aulas/:id` só exige autenticação. Os
> dados de aluno (telefone, inadimplência) são considerados mais sensíveis.

## Endpoints

### POST /api/v1/aulas/:aula_id/alunos

Matricula um aluno na aula. Aciona `CreateAlunoUsecase`: confirma que a aula existe, checa a
capacidade, deriva `ativo` de `vencimento_fatura >= agora` e persiste.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (ObjectId) | Sim | Aula na qual matricular |

**Corpo da requisição** (`createAlunoDTO`, `stripUnknown: true` — um `aula_id` no corpo é ignorado)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Sim | |
| `telefone` | string | Sim | |
| `vencimento_fatura` | string/Date | Sim | Coagido para `Date` |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Aluno matriculado com sucesso",
  "data": {
    "id": "507f1f77bcf86cd799439012",
    "aula_id": "507f1f77bcf86cd799439011",
    "nome": "João da Silva",
    "telefone": "11987654321",
    "vencimento_fatura": "2026-12-31T00:00:00.000Z",
    "ativo": true
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido — `{ "message": "Dados inválidos", "errors": [...] }` (campo ausente, `vencimento_fatura` não é data) |
| 401 | Token não fornecido/inválido |
| 403 | Não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aula (`:aula_id`) não encontrada — **"Aula nao encontrada"** |
| 409 | Aula lotada (`total de alunos não deletados >= capacidade_maxima`) — **"Capacidade maxima de alunos atingida para esta aula"** |
| 500 | `:aula_id` malformado (CastError do Mongoose não tratado no fluxo de matrícula) ou erro inesperado de persistência — **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011/alunos" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"nome": "João da Silva", "telefone": "11987654321", "vencimento_fatura": "2026-12-31T00:00:00.000Z"}'
```

---

### GET /api/v1/aulas/:aula_id/alunos

Lista os alunos não deletados da aula, **após** recalcular o campo `ativo` (ver "Recálculo lazy"
acima). Aciona `ListAlunosUsecase`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (ObjectId) | Sim | Aula cujos alunos listar |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Alunos listados com sucesso",
  "data": [
    { "id": "507f1f77bcf86cd799439012", "aula_id": "507f1f77bcf86cd799439011", "nome": "João da Silva", "telefone": "11987654321", "vencimento_fatura": "2026-12-31T00:00:00.000Z", "ativo": true },
    { "id": "507f1f77bcf86cd799439013", "aula_id": "507f1f77bcf86cd799439011", "nome": "Maria Souza", "telefone": "11912345678", "vencimento_fatura": "2025-01-10T00:00:00.000Z", "ativo": false }
  ]
}
```
Aula sem alunos retorna `200` com `"data": []`.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `:aula_id` não é ObjectId válido — **"ID de aula inválido"** |
| 401 | Token não fornecido/inválido |
| 403 | Não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aula não encontrada (via `requireOwnerOrAdmin`, quando o solicitante não é `ADMIN`) — **"Aula nao encontrada"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011/alunos" \
  -H "Authorization: Bearer <idToken>"
```

---

### GET /api/v1/aulas/:aula_id/alunos/:id

Busca um aluno pelo `id`. Aciona `ReadAlunoUsecase`. Não recalcula `ativo` (o recálculo só ocorre
na listagem).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (ObjectId) | Sim | Usado pelos guards (ownership) |
| `id` | string (ObjectId) | Sim | Identificador do aluno |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aluno encontrado com sucesso",
  "data": { "id": "507f1f77bcf86cd799439012", "aula_id": "507f1f77bcf86cd799439011", "nome": "João da Silva", "telefone": "11987654321", "vencimento_fatura": "2026-12-31T00:00:00.000Z", "ativo": true }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `:id` não é ObjectId válido — **"ID inválido"** |
| 401 | Token não fornecido/inválido |
| 403 | Não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aluno não encontrado ou soft-deletado — **"Aluno nao encontrado"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011/alunos/507f1f77bcf86cd799439012" \
  -H "Authorization: Bearer <idToken>"
```

---

### PUT /api/v1/aulas/:aula_id/alunos/:id

Atualiza um aluno (parcial). Aciona `UpdateAlunoUsecase`. O controller monta o payload só com os
campos definidos. É o fluxo usado para **registrar o pagamento da mensalidade** — basta atualizar
`vencimento_fatura` para uma data futura; na próxima listagem o aluno volta a `ativo: true`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (ObjectId) | Sim | Usado pelos guards (ownership) |
| `id` | string (ObjectId) | Sim | Identificador do aluno |

**Corpo da requisição** (`updateAlunoDTO` — todos opcionais)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `nome` | string | Não | |
| `telefone` | string | Não | |
| `vencimento_fatura` | string/Date | Não | Coagido para `Date` |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aluno atualizado com sucesso",
  "data": { "id": "507f1f77bcf86cd799439012", "aula_id": "507f1f77bcf86cd799439011", "nome": "João da Silva", "telefone": "11999998888", "vencimento_fatura": "2027-01-31T00:00:00.000Z", "ativo": false }
}
```

> `ativo` **não** é recalculado nesta operação — só na próxima listagem (`GET`). O valor retornado
> aqui reflete o estado atual do documento.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido (`vencimento_fatura` não é data) |
| 400 | `:id` não é ObjectId válido — **"ID inválido"** |
| 401 | Token não fornecido/inválido |
| 403 | Não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aluno não encontrado ou soft-deletado — **"Aluno nao encontrado"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl -X PUT "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011/alunos/507f1f77bcf86cd799439012" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"vencimento_fatura": "2027-01-31T00:00:00.000Z"}'
```

---

### DELETE /api/v1/aulas/:aula_id/alunos/:id

Remove (soft delete) um aluno. Aciona `DeleteAlunoUsecase`: marca `deleted: true`. A partir daí o
aluno **não conta mais** para `capacidade_maxima` (libera a vaga).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `aula_id` | string (ObjectId) | Sim | Usado pelos guards (ownership) |
| `id` | string (ObjectId) | Sim | Identificador do aluno |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aluno removido com sucesso",
  "data": { "id": "507f1f77bcf86cd799439012", "aula_id": "507f1f77bcf86cd799439011", "nome": "João da Silva", "telefone": "11987654321", "vencimento_fatura": "2026-12-31T00:00:00.000Z", "ativo": true }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `:id` não é ObjectId válido — **"ID inválido"** |
| 401 | Token não fornecido/inválido |
| 403 | Não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aluno inexistente ou já deletado — **"Aluno nao encontrado"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl -X DELETE "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011/alunos/507f1f77bcf86cd799439012" \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `services/beach-center-bff-aulas/src/applications/routes/aluno.route.ts`
- Montagem: `services/beach-center-bff-aulas/src/applications/routes/routes.ts` (`/aulas/:aula_id/alunos`)
- Controllers: `services/beach-center-bff-aulas/src/applications/controllers/aluno/{create,read,update,delete,list}/*.controller.ts`
- Usecases: `services/beach-center-bff-aulas/src/domain/usecases/aluno/{create,read,update,delete,list}/*.usecase.ts`
- DTO: `services/beach-center-bff-aulas/src/applications/dto/aluno.dto.ts`
- Model: `services/beach-center-bff-aulas/src/domain/models/aluno.model.ts`
- Ports: `services/beach-center-bff-aulas/src/domain/ports/input/aluno.input-port.ts`, `.../ports/output/aluno-persistence.port.ts` (`ICountAlunosByAulaPort`, `IRecalculateAlunosStatusPort`), `.../ports/output/aula-persistence.port.ts` (`IReadAulaPort`)
- Adapters: `services/beach-center-bff-aulas/src/infra/adapters/aluno/{create,read,update,delete,list,count-by-aula,recalculate-status}/*.adapter.ts`
- Schema: `services/beach-center-bff-aulas/src/infra/schemas/aluno.schema.ts` (índice em `aula_id`)
- Middlewares: `services/beach-center-bff-aulas/src/applications/middlewares/auth.middleware.ts` (`authMiddleware`, `requireOwnerOrAdmin`)
- Erros de domínio: `services/beach-center-bff-aulas/src/domain/errors.ts`
- Tradução HTTP: `services/beach-center-bff-aulas/src/applications/controllers/shared/handle-http-error.ts`
