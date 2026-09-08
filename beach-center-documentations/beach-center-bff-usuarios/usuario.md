# Usuário

## Visão geral
Recurso de administração de usuários do serviço `beach-center-bff-usuarios`. Permite listar, consultar, atualizar e remover usuários cadastrados (MongoDB + Firebase Authentication). Rotas montadas em `/usuarios` pelo router principal (`src/applications/routes/routes.ts`), servidas sob o prefixo global `/api/v1` (`src/main.ts`).

## Autenticação/autorização
Todas as rotas exigem, nesta ordem:
1. `authMiddleware` (`src/applications/middlewares/auth.middleware.ts`) — token Bearer válido no header `Authorization`. Erros: `401` **"Token não fornecido"** (header ausente/malformado), `401` **"Token inválido ou expirado"** (falha na verificação do token no Firebase), `403` **"Usuário não encontrado no sistema"** (token válido, mas sem usuário correspondente no MongoDB).
2. `requireRole("ADMIN")` (mesmo arquivo) — exige que o usuário autenticado tenha `user_type` igual a `ADMIN` (comparação case-insensitive). Erros: `401` **"Token invalido ou expirado"** (sem `uid` no token decodificado), `403` **"Usuario nao encontrado no sistema"** (usuário não encontrado ao reconsultar por `uid`), `403` **"Acesso negado"** (usuário autenticado não é `ADMIN`).

Ou seja: todas as rotas abaixo são restritas a usuários com perfil `ADMIN`.

## Endpoints

### DELETE /usuarios/:id
Remove um usuário do MongoDB e sua conta correspondente no Firebase Authentication. Aciona `DeleteUserUsecase` (`container.deleteUser`).

**Parâmetros de path/query**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| id | string (path) | sim | ObjectId do MongoDB (24 caracteres hexadecimais) do usuário a remover |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Usuário deletado com sucesso",
  "data": {
    "id": "<mongoId>",
    "id_firestore": "<firebaseUid>",
    "name": "Fulano da Silva",
    "email": "fulano@example.com",
    "phone": "+5511999999999",
    "user_type": "CLIENTE",
    "avatar_base64": null,
    "avatar_content_type": null
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | `id` não é um ObjectId válido — mensagem **"ID inválido"** |
| 401 | Não autenticado (ver seção de autenticação) |
| 403 | Autenticado, mas sem perfil `ADMIN` (ver seção de autenticação) |
| 404 | Usuário não encontrado no MongoDB — mensagem **"Usuário não encontrado"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X DELETE https://<host>/api/v1/usuarios/64f1a2b3c4d5e6f7a8b9c0d1 \
  -H "Authorization: Bearer <idToken>"
```

---

### GET /usuarios
Lista usuários cadastrados, com filtros opcionais por nome, e-mail e tipo de usuário. Aciona `ListUsersUsecase` (`container.listUsers`).

**Parâmetros de path/query**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| name | string (query) | não | Filtra usuários pelo nome |
| email | string (query) | não | Filtra usuários pelo e-mail |
| user_type | string (query) | não | Filtra por tipo de usuário (`CLIENTE`, `ADMIN` ou `PROFESSOR`); repassado como texto, sem validação de enum na rota |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Usuários listados com sucesso",
  "data": [
    {
      "id": "<mongoId>",
      "id_firestore": "<firebaseUid>",
      "name": "Fulano da Silva",
      "email": "fulano@example.com",
      "phone": "+5511999999999",
      "user_type": "CLIENTE",
      "avatar_base64": null,
      "avatar_content_type": null
    }
  ]
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 401 | Não autenticado (ver seção de autenticação) |
| 403 | Autenticado, mas sem perfil `ADMIN` (ver seção de autenticação) |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X GET "https://<host>/api/v1/usuarios?user_type=CLIENTE&name=Fulano" \
  -H "Authorization: Bearer <idToken>"
```

---

### GET /usuarios/:id
Consulta um usuário específico pelo seu ObjectId no MongoDB. Aciona `ReadUserUsecase` (`container.readUser`).

**Parâmetros de path/query**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| id | string (path) | sim | ObjectId do MongoDB (24 caracteres hexadecimais) |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Usuário encontrado com sucesso",
  "data": {
    "id": "<mongoId>",
    "id_firestore": "<firebaseUid>",
    "name": "Fulano da Silva",
    "email": "fulano@example.com",
    "phone": "+5511999999999",
    "user_type": "CLIENTE",
    "avatar_base64": null,
    "avatar_content_type": null
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | `id` não é um ObjectId válido — mensagem **"ID inválido"** |
| 401 | Não autenticado (ver seção de autenticação) |
| 403 | Autenticado, mas sem perfil `ADMIN` (ver seção de autenticação) |
| 404 | Usuário não encontrado no MongoDB — mensagem **"Usuário não encontrado"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X GET https://<host>/api/v1/usuarios/64f1a2b3c4d5e6f7a8b9c0d1 \
  -H "Authorization: Bearer <idToken>"
```

---

### PATCH /usuarios/:id
Atualiza nome, e-mail, telefone e tipo de usuário (`user_type`) de um usuário, sincronizando Firebase Authentication e MongoDB. Aciona `UpdateUserUsecase` (`container.updateUser`). Se a atualização no MongoDB falhar após o Firebase já ter sido alterado, os dados do Firebase (nome, e-mail e telefone) são revertidos para os valores anteriores.

**Parâmetros de path/query**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| id | string (path) | sim | ObjectId do MongoDB (24 caracteres hexadecimais) |

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| name | string | sim | Nome do usuário |
| email | string | sim | E-mail (formato válido) |
| phone | string | sim | Telefone; deve ser conversível para E.164 |
| user_type | string | sim | Um de: `CLIENTE`, `ADMIN`, `PROFESSOR` (o valor `PROFESSOR` habilita o usuário como responsável por uma turma em `beach-center-bff-aulas` — ver [`aula`](../beach-center-bff-aulas/aula.md)) |

**Resposta de sucesso**: `200 OK`

Observação: a mensagem de sucesso retornada pelo código atual é a mesma da consulta ("Usuário encontrado com sucesso"), mesmo em uma operação de atualização — comportamento fiel ao controller (`update-user.controller.ts`).
```json
{
  "message": "Usuário encontrado com sucesso",
  "data": {
    "id": "<mongoId>",
    "id_firestore": "<firebaseUid>",
    "name": "Fulano da Silva Atualizado",
    "email": "novo-email@example.com",
    "phone": "+5511988888888",
    "user_type": "ADMIN",
    "avatar_base64": null,
    "avatar_content_type": null
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Corpo inválido (campo ausente, e-mail inválido, `user_type` fora do enum) — `{ "message": "Dados inválidos", "errors": [{ "field", "message" }] }` |
| 400 | `id` não é um ObjectId válido — mensagem **"ID inválido"** |
| 400 | Telefone não conversível para E.164 — mensagem **"Telefone invalido"** |
| 401 | Não autenticado (ver seção de autenticação) |
| 403 | Autenticado, mas sem perfil `ADMIN` (ver seção de autenticação) |
| 404 | Usuário não encontrado no MongoDB — mensagem **"Usuário não encontrado"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X PATCH https://<host>/api/v1/usuarios/64f1a2b3c4d5e6f7a8b9c0d1 \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Fulano da Silva Atualizado",
    "email": "novo-email@example.com",
    "phone": "11988888888",
    "user_type": "ADMIN"
  }'
```

## Referências
- `src/main.ts` (prefixo global `/api/v1`)
- `src/applications/routes/routes.ts` (montagem de `/usuarios`)
- `src/applications/routes/user.route.ts`
- `src/applications/middlewares/auth.middleware.ts` (`authMiddleware`, `requireRole`)
- `src/applications/controllers/delete-user/delete-user.controller.ts`
- `src/applications/controllers/list-users/list-users.controller.ts`
- `src/applications/controllers/read-user/read-user.controller.ts`
- `src/applications/controllers/update-user/update-user.controller.ts`
- `src/applications/controllers/shared/handle-http-error.ts`
- `src/applications/dto/user.dto.ts`
- `src/domain/usecases/delete-user/delete-user.usecase.ts`
- `src/domain/usecases/list-users/list-users.usecase.ts`
- `src/domain/usecases/read-user/read-user.usecase.ts`
- `src/domain/usecases/update-user/update-user.usecase.ts`
- `src/domain/usecases/auth/authenticate-request.usecase.ts` (`AuthorizeRoleUsecase`)
- `src/domain/usecases/shared/object-id.ts` (`assertValidObjectId`)
- `src/domain/usecases/shared/phone-number.ts` (`normalizePhoneNumberToE164`)
- `src/domain/errors.ts` (`DomainError` e subclasses)
- `src/domain/models/user.model.ts` (`IUser`)
- `src/config/container.ts` (injeção de dependências pós-refatoração hexagonal — não altera o contrato documentado acima)
