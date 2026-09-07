# Quadra

## Visão geral

O recurso `quadra` (rota `/quadras`) representa as quadras físicas de beach tennis cadastradas no sistema. É exposto pelo microsserviço `beach-center-bff-agendamentos` e é referenciado por outros recursos do mesmo serviço (`unidade`, `agendamento`, `evento-agendado`) via `court_id`/`court`. O modelo de domínio (`ICourt`) é minimalista: apenas `id`, `name` e a flag interna `deleted` (soft delete via `mongoose-delete`, não exposta nas respostas).

## Autenticação/autorização

| Endpoint | Guard |
|---|---|
| `GET /quadras/:id` | Pública (sem middleware) |
| `POST /quadras` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /quadras/:id` | `authMiddleware` + `requireRole('ADMIN')` |
| `PATCH /quadras/:id/delete` | `authMiddleware` + `requireRole('ADMIN')` |

`authMiddleware` exige header `Authorization: Bearer <idToken>` (Firebase) válido e que o usuário exista na coleção `users` (espelho de `usuarios`); `requireRole('ADMIN')` exige `databaseUser.user_type === "ADMIN"`.

## Endpoints

### POST /api/v1/quadras

Cria uma nova quadra. Aciona `CreateCourtUsecase`, que delega diretamente ao adapter de persistência (sem regra de negócio adicional além da validação do DTO).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome da quadra. Campos desconhecidos são removidos (`stripUnknown: true`). |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Court created successfully",
  "data": { "id": "507f1f77bcf86cd799439011", "name": "Quadra 1" }
}
```

> Nota: a mensagem de sucesso está em **inglês** ("Court created successfully") — inconsistência pré-existente em relação ao restante do serviço (PT-BR), preservada deliberadamente pela refatoração de arquitetura (não é um bug desta documentação).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `name` ausente/vazio — yup `ValidationError`: `{"message": "Dados inválidos", "errors": [{"field": "name", "message": "..."}]}` |
| 401 | Token não fornecido/inválido |
| 403 | Usuário autenticado não é `ADMIN` |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**

```bash
curl -X POST "http://localhost:5000/api/v1/quadras" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Quadra 1"}'
```

---

### GET /api/v1/quadras/:id

Busca uma quadra pelo `id`. Aciona `ReadCourtUsecase`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId, 24 chars hex) | Sim | Identificador da quadra. |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Quadra encontrada com sucesso",
  "data": { "id": "507f1f77bcf86cd799439011", "name": "Quadra 1" }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é um ObjectId válido de 24 caracteres hex — `InvalidInputError("ID inválido")` |
| 404 | Nenhuma quadra ativa (não deletada) encontrada com esse `id` — `NotFoundError("Quadra não encontrada")` |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**

```bash
curl "http://localhost:5000/api/v1/quadras/507f1f77bcf86cd799439011"
```

---

### PATCH /api/v1/quadras/:id

Atualiza os dados de uma quadra existente. Aciona `UpdateCourtUsecase`: valida o `id`, atualiza e, em caso de sucesso, retorna também a lista completa de quadras ativas (reaproveitada pelo front-end para atualizar listagens sem uma nova chamada).

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId) | Sim | Identificador da quadra a atualizar. |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Novo nome da quadra. |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Quadra encontrada com sucesso",
  "data": { "id": "507f1f77bcf86cd799439011", "name": "Quadra renomeada" },
  "list": [
    { "id": "507f1f77bcf86cd799439011", "name": "Quadra renomeada" },
    { "id": "507f1f77bcf86cd799439012", "name": "Quadra 2" }
  ]
}
```

> Nota: a mensagem de sucesso reaproveita o texto de `GET` ("Quadra encontrada com sucesso"), não "Quadra atualizada com sucesso" — comportamento pré-existente preservado.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `name` ausente/vazio — `{"message": "Dados inválidos", "errors": [...]}` |
| 400 | `id` do path não é um ObjectId válido — `InvalidInputError("ID inválido")` |
| 401 | Token não fornecido/inválido |
| 403 | Usuário não é `ADMIN` |
| 404 | Quadra não encontrada (nenhuma quadra ativa com esse `id`) — `{"message": "Quadra não encontrada"}` (verificado pelo controller a partir de `!result.list.length \|\| !result.updated`, não lançado como `DomainError`) |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**

```bash
curl -X PATCH "http://localhost:5000/api/v1/quadras/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Quadra renomeada"}'
```

---

### PATCH /api/v1/quadras/:id/delete

Remove (soft delete) uma quadra. Aciona `DeleteCourtUsecase`: valida o `id`, marca a quadra como `deleted: true` (`mongoose-delete`) e retorna a lista atualizada de quadras ativas.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `id` | string (ObjectId) | Sim | Identificador da quadra a remover. |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Quadra deletada com sucesso",
  "data": [
    { "id": "507f1f77bcf86cd799439012", "name": "Quadra 2" }
  ]
}
```

> Nota: `data` aqui é a **lista** de quadras remanescentes (retorno de `DeleteCourtUsecase`), não o objeto da quadra deletada.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `id` não é um ObjectId válido — `InvalidInputError("ID inválido")` |
| 401 | Token não fornecido/inválido |
| 403 | Usuário não é `ADMIN` |
| 404 | Quadra inexistente ou já deletada — `{"message": "Quadra não encontrada"}` |
| 500 | Erro inesperado de persistência |

**Exemplo de chamada**

```bash
curl -X PATCH "http://localhost:5000/api/v1/quadras/507f1f77bcf86cd799439011/delete" \
  -H "Authorization: Bearer <idToken>"
```

## Referências

- Rota: `services/beach-center-bff-agendamentos/src/applications/routes/court.route.ts`
- Controllers: `services/beach-center-bff-agendamentos/src/applications/controllers/court/{create,read,update,delete}/*.controller.ts`
- Usecases: `services/beach-center-bff-agendamentos/src/domain/usecases/court/{create,read,update,delete}/*.usecase.ts`
- DTO: `services/beach-center-bff-agendamentos/src/applications/dto/court.dto.ts`
- Model: `services/beach-center-bff-agendamentos/src/domain/models/court.model.ts`
- Ports: `services/beach-center-bff-agendamentos/src/domain/ports/input/court.input-port.ts`, `services/beach-center-bff-agendamentos/src/domain/ports/output/court-persistence.port.ts`
- Adapters: `services/beach-center-bff-agendamentos/src/infra/adapters/court/{create,read,update,delete,list,find-active-by-id}/*.adapter.ts`
- Erros de domínio: `services/beach-center-bff-agendamentos/src/domain/errors.ts`
- Middlewares: `services/beach-center-bff-agendamentos/src/applications/middlewares/auth.middleware.ts`
