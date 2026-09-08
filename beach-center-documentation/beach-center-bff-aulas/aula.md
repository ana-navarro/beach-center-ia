# Aula

## Visão geral

O recurso `aula` (rota `/aulas`) representa uma turma de aula recorrente (beach tennis, vôlei ou
outra modalidade) — classe, modalidade, dias da semana, horário, professor responsável, quadra de
referência e capacidade máxima de alunos. É exposto pelo microsserviço `beach-center-bff-aulas`,
servido sob o prefixo global `/api/v1` (`src/main.ts`, porta padrão **5002**).

O modelo de domínio (`IAula`):

| Campo | Tipo | Descrição |
|---|---|---|
| `id` | string (ObjectId) | Identificador da aula |
| `classe` | string | Nome/identificador da turma |
| `modalidade` | string | Categoria (ex.: `"Beach Tenis"`, `"Volei"`) — texto livre, sem enum |
| `dias` | string[] | Dias da semana recorrentes — texto livre, sem enum nem deduplicação |
| `hora_inicio` | string (ISO date) | Início da aula |
| `hora_fim` | string (ISO date) | Fim da aula — **não** há validação de que `hora_fim > hora_inicio` |
| `professor` | string (ObjectId) | Referencia um usuário de `beach-center-bff-usuarios` com `user_type = "PROFESSOR"` |
| `quadra` | string | Referência livre ao id de uma quadra em `agendamentos` — **sem** validação cruzada nem bloqueio de horário nesta versão (fora de escopo) |
| `capacidade_maxima` | number | Inteiro ≥ 1 — limite de alunos matriculados (não deletados) |

O campo interno `deleted` (soft delete manual) não é exposto nas respostas. A gestão de alunos da
turma fica no recurso aninhado [`aluno`](./aluno.md) (`/aulas/:aula_id/alunos`).

## Autenticação/autorização

| Endpoint | Guards |
|---|---|
| `POST /api/v1/aulas` | `authMiddleware` + `requireRole('ADMIN')` |
| `GET /api/v1/aulas` | `authMiddleware` (qualquer `user_type` autenticado) |
| `GET /api/v1/aulas/:id` | `authMiddleware` (qualquer `user_type` autenticado) |
| `PUT /api/v1/aulas/:id` | `authMiddleware` + `requireOwnerOrAdmin` |
| `DELETE /api/v1/aulas/:id` | `authMiddleware` + `requireRole('ADMIN')` |

- **`authMiddleware`** exige `Authorization: Bearer <idToken>` (Firebase) válido e que o `uid` do
  token corresponda a um usuário na cópia local de `usuarios` (`src/infra/schemas/user.schema.ts`).
  Erros: `401` **"Token nao fornecido"** (header ausente/malformado), `401` **"Token invalido ou
  expirado"** (falha na verificação Firebase), `403` **"Usuario nao encontrado no sistema"** (token
  válido, sem usuário local).
- **`requireRole('ADMIN')`** exige `databaseUser.user_type === "ADMIN"` (case-insensitive). Erros:
  `401` **"Token invalido ou expirado"** (sem `databaseUser`), `403` **"Acesso negado"** (não é ADMIN).
- **`requireOwnerOrAdmin`** libera `ADMIN` direto; caso contrário carrega a aula (`:id` da rota) via
  `container.readAula` e exige que `aula.professor === databaseUser.id`. Erros: `401` **"Token
  invalido ou expirado"** (sem `databaseUser`), `403` **"Acesso negado"** (sem id na rota, ou
  professor não é o dono), `400` **"ID invalido"** / `404` **"Aula nao encontrada"** (propagados do
  `readAula` quando o `:id` é malformado ou inexistente).

> **Criar aula é exclusivo de `ADMIN`** — não existe "dono" antes da aula existir. Apagar a turma
> também é exclusivo de `ADMIN`: o professor gerencia alunos/vagas mas não desfaz a turma.

## Endpoints

### POST /api/v1/aulas

Cria uma nova aula. Aciona `CreateAulaUsecase`: valida que `professor` existe em `usuarios` **e**
que tem `user_type = "PROFESSOR"` antes de persistir.

**Corpo da requisição** (`createAulaDTO`, `stripUnknown: true`)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `classe` | string | Sim | Nome da turma |
| `modalidade` | string | Sim | Categoria da modalidade |
| `dias` | string[] | Sim | Ao menos 1 item |
| `hora_inicio` | string/Date | Sim | Coagido para `Date` |
| `hora_fim` | string/Date | Sim | Coagido para `Date` |
| `professor` | string | Sim | ObjectId de 24 caracteres hexadecimais (regex `^[0-9a-fA-F]{24}$`) |
| `quadra` | string | Sim | Id da quadra (referência livre) |
| `capacidade_maxima` | number | Sim | Inteiro positivo, mínimo 1 |

**Resposta de sucesso** — `201 Created`
```json
{
  "message": "Aula criada com sucesso",
  "data": {
    "id": "507f1f77bcf86cd799439011",
    "classe": "Turma A",
    "modalidade": "Beach Tenis",
    "dias": ["segunda", "quarta"],
    "hora_inicio": "2026-02-01T08:00:00.000Z",
    "hora_fim": "2026-02-01T09:00:00.000Z",
    "professor": "507f1f77bcf86cd799439099",
    "quadra": "507f1f77bcf86cd7994390aa",
    "capacidade_maxima": 10
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido — `{ "message": "Dados inválidos", "errors": [{ "field", "message" }] }` (campo obrigatório ausente, `dias` vazio, `professor` fora do formato ObjectId, `capacidade_maxima` ≤ 0 ou não-inteiro) |
| 400 | `professor` existe mas `user_type` ≠ `"PROFESSOR"` — **"Usuario informado nao e um professor"** |
| 401 | Token não fornecido/inválido |
| 403 | Usuário autenticado não é `ADMIN` — **"Acesso negado"** |
| 404 | `professor` não encontrado em `usuarios` — **"Professor nao encontrado"** |
| 500 | Erro inesperado de persistência — **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5002/api/v1/aulas" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "classe": "Turma A",
    "modalidade": "Beach Tenis",
    "dias": ["segunda", "quarta"],
    "hora_inicio": "2026-02-01T08:00:00.000Z",
    "hora_fim": "2026-02-01T09:00:00.000Z",
    "professor": "507f1f77bcf86cd799439099",
    "quadra": "507f1f77bcf86cd7994390aa",
    "capacidade_maxima": 10
  }'
```

---

### GET /api/v1/aulas

Lista todas as aulas não deletadas. Aciona `ListAulasUsecase` (sem filtros — retorna tudo).

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aulas listadas com sucesso",
  "data": [
    { "id": "507f1f77bcf86cd799439011", "classe": "Turma A", "modalidade": "Beach Tenis", "dias": ["segunda"], "hora_inicio": "2026-02-01T08:00:00.000Z", "hora_fim": "2026-02-01T09:00:00.000Z", "professor": "507f1f77bcf86cd799439099", "quadra": "507f1f77bcf86cd7994390aa", "capacidade_maxima": 10 }
  ]
}
```
Lista vazia retorna `200` com `"data": []`.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Token não fornecido/inválido |
| 403 | `uid` do token sem usuário local — **"Usuario nao encontrado no sistema"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl "http://localhost:5002/api/v1/aulas" -H "Authorization: Bearer <idToken>"
```

---

### GET /api/v1/aulas/:id

Busca uma aula pelo `id`. Aciona `ReadAulaUsecase`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId, 24 hex) | Sim | Identificador da aula |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aula encontrada com sucesso",
  "data": { "id": "507f1f77bcf86cd799439011", "classe": "Turma A", "modalidade": "Beach Tenis", "dias": ["segunda", "quarta"], "hora_inicio": "2026-02-01T08:00:00.000Z", "hora_fim": "2026-02-01T09:00:00.000Z", "professor": "507f1f77bcf86cd799439099", "quadra": "507f1f77bcf86cd7994390aa", "capacidade_maxima": 10 }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é ObjectId válido de 24 hex — **"ID invalido"** |
| 401 | Token não fornecido/inválido |
| 403 | `uid` do token sem usuário local |
| 404 | Nenhuma aula ativa com esse `id` — **"Aula nao encontrada"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer <idToken>"
```

---

### PUT /api/v1/aulas/:id

Atualiza uma aula (parcial). Aciona `UpdateAulaUsecase`: valida o `id`, confirma que a aula existe
e, **se** `professor` estiver no corpo, revalida que o novo professor existe e é `PROFESSOR`. O
controller descarta campos ausentes/`undefined` antes de repassar ao usecase.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId) | Sim | Identificador da aula |

**Corpo da requisição** (`updateAulaDTO` — todos opcionais, mesmas regras do `create`)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `classe` | string | Não | |
| `modalidade` | string | Não | |
| `dias` | string[] | Não | Se presente, mínimo 1 item |
| `hora_inicio` | string/Date | Não | |
| `hora_fim` | string/Date | Não | |
| `professor` | string | Não | Se presente, ObjectId de 24 hex; revalidado como `PROFESSOR` |
| `quadra` | string | Não | |
| `capacidade_maxima` | number | Não | Inteiro positivo ≥ 1 |

Corpo vazio `{}` é aceito (nenhum campo alterado). Reduzir `capacidade_maxima` abaixo do total de
alunos já matriculados **é permitido** (não há checagem retroativa).

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aula atualizada com sucesso",
  "data": { "id": "507f1f77bcf86cd799439011", "classe": "Turma A - manhã", "modalidade": "Beach Tenis", "dias": ["segunda", "quarta"], "hora_inicio": "2026-02-01T08:00:00.000Z", "hora_fim": "2026-02-01T09:00:00.000Z", "professor": "507f1f77bcf86cd799439099", "quadra": "507f1f77bcf86cd7994390aa", "capacidade_maxima": 10 }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido (yup `ValidationError`) |
| 400 | `id` do path não é ObjectId válido — **"ID invalido"** |
| 400 | Novo `professor` não é `PROFESSOR` — **"Usuario informado nao e um professor"** |
| 401 | Token não fornecido/inválido |
| 403 | `requireOwnerOrAdmin`: não é `ADMIN` nem o professor dono da aula — **"Acesso negado"** |
| 404 | Aula não encontrada — **"Aula nao encontrada"** |
| 404 | Novo `professor` não encontrado — **"Professor nao encontrado"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl -X PUT "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"classe": "Turma A - manhã"}'
```

---

### DELETE /api/v1/aulas/:id

Remove (soft delete) uma aula. Aciona `DeleteAulaUsecase`: valida o `id` e marca `deleted: true`.
A aula deixa de aparecer em `GET /aulas` e `GET /aulas/:id`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId) | Sim | Identificador da aula |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Aula deletada com sucesso",
  "data": { "id": "507f1f77bcf86cd799439011", "classe": "Turma A", "modalidade": "Beach Tenis", "dias": ["segunda"], "hora_inicio": "2026-02-01T08:00:00.000Z", "hora_fim": "2026-02-01T09:00:00.000Z", "professor": "507f1f77bcf86cd799439099", "quadra": "507f1f77bcf86cd7994390aa", "capacidade_maxima": 10 }
}
```

> `data` é o objeto da aula soft-deletada (retorno do adapter com `deleted: true` já aplicado).
> Os alunos da turma **não** são removidos em cascata.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é ObjectId válido — **"ID invalido"** |
| 401 | Token não fornecido/inválido |
| 403 | Usuário não é `ADMIN` (mesmo sendo o professor dono) — **"Acesso negado"** |
| 404 | Aula inexistente ou já deletada — **"Aula nao encontrada"** |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**
```bash
curl -X DELETE "http://localhost:5002/api/v1/aulas/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `services/beach-center-bff-aulas/src/applications/routes/aula.route.ts`
- Montagem: `services/beach-center-bff-aulas/src/applications/routes/routes.ts` (prefixo `/api/v1` em `src/main.ts`)
- Controllers: `services/beach-center-bff-aulas/src/applications/controllers/aula/{create,read,update,delete,list}/*.controller.ts`
- Usecases: `services/beach-center-bff-aulas/src/domain/usecases/aula/{create,read,update,delete,list}/*.usecase.ts`
- DTO: `services/beach-center-bff-aulas/src/applications/dto/aula.dto.ts`
- Model: `services/beach-center-bff-aulas/src/domain/models/aula.model.ts`
- Ports: `services/beach-center-bff-aulas/src/domain/ports/input/aula.input-port.ts`, `.../ports/output/aula-persistence.port.ts`, `.../ports/output/auth.port.ts` (`IFindUserByIdPort`)
- Adapters: `services/beach-center-bff-aulas/src/infra/adapters/aula/{create,read,update,delete,list}/*.adapter.ts`, `.../adapters/user/find-user-by-id.adapter.ts`
- Schema: `services/beach-center-bff-aulas/src/infra/schemas/aula.schema.ts`
- Middlewares: `services/beach-center-bff-aulas/src/applications/middlewares/auth.middleware.ts` (`authMiddleware`, `requireRole`, `requireOwnerOrAdmin`)
- Erros de domínio: `services/beach-center-bff-aulas/src/domain/errors.ts`
- Tradução HTTP: `services/beach-center-bff-aulas/src/applications/controllers/shared/handle-http-error.ts`
