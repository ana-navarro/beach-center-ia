# Autenticação

## Visão geral
Recurso responsável pelo ciclo de vida de autenticação e pelo autoatendimento do usuário autenticado (perfil próprio, avatar, e-mail e senha) no serviço `beach-center-bff-usuarios`. As rotas combinam o Firebase Authentication (identidade/credenciais) com a persistência de usuários no MongoDB. Todas as rotas são montadas em `/auth` pelo router principal (`src/applications/routes/routes.ts`) e o serviço expõe tudo sob o prefixo global `/api/v1` (definido em `src/main.ts`).

> **Firebase Auth Emulator (desenvolvimento).** Quando a variável de ambiente `FIREBASE_AUTH_EMULATOR_HOST` está definida (ex.: no `docker-compose.dev.yml` do `beach-center-server`), o serviço **não** exige service account: `ensureFirebaseApp` inicializa o Admin SDK só com o `projectId`, e as chamadas REST de `POST /auth/login` e `POST /auth/forgot-password` são direcionadas para `http://<host>/identitytoolkit.googleapis.com/v1/...` em vez do Google (`src/infra/adapters/auth/identity-toolkit.ts`). **O contrato das rotas — corpo, resposta e códigos de status — é idêntico nos dois modos.** Sem a variável (produção, CI), o comportamento é o documentado abaixo, com service account e host do Google.

## Autenticação/autorização
- `POST /auth/register`, `POST /auth/login` e `POST /auth/forgot-password` são **públicas** (sem middleware).
- Todas as demais rotas (`/auth/me*`) exigem `authMiddleware` (`src/applications/middlewares/auth.middleware.ts`), que:
  1. Extrai o token `Bearer <token>` do header `Authorization`. Se ausente/malformado → `401` com mensagem **"Token não fornecido"**.
  2. Verifica o ID token no Firebase. Se inválido/expirado → `401` com mensagem **"Token inválido ou expirado"**.
  3. Busca o usuário correspondente no MongoDB pelo `uid` do Firebase. Se não encontrado → `403` com mensagem **"Usuário não encontrado no sistema"**.
  4. Em caso de sucesso, popula `req.authToken` (claims decodificadas, incluindo `uid`) e `req.authUser` (documento do usuário) para uso pelos controllers.
- Não há verificação de perfil (`requireRole`) nestas rotas — qualquer usuário autenticado pode acessar seu próprio recurso `/me`.

## Endpoints

### POST /auth/register
Cadastra um novo usuário: cria o usuário no Firebase Authentication e, em seguida, no MongoDB, com `user_type` fixado como `CLIENTE`. Aciona `RegisterAuthUsecase` (`container.registerAuth`). Em caso de falha após a criação no Firebase, o usuário criado no Firebase é removido (rollback).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| name | string | sim | Nome do usuário |
| email | string | sim | E-mail válido (formato de e-mail) |
| phone | string | sim | Telefone; deve ser conversível para E.164 (aceita formatos BR com/sem DDI, ou já em E.164 com `+`) |
| password | string | sim | Senha, mínimo de 6 caracteres |

**Resposta de sucesso**: `201 Created`
```json
{
  "message": "Usuario registrado com sucesso",
  "data": {
    "user": {
      "id": "<mongoId>",
      "name": "Fulano da Silva",
      "email": "fulano@example.com",
      "phone": "+5511999999999",
      "user_type": "CLIENTE",
      "avatar_base64": null,
      "avatar_content_type": null
    },
    "customToken": "<firebaseCustomToken>"
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Corpo inválido (campo ausente/formato de e-mail inválido/senha curta) — resposta `{ "message": "Dados inválidos", "errors": [{ "field", "message" }] }` |
| 400 | Telefone não conversível para E.164 — mensagem **"Telefone invalido"** |
| 409 | E-mail já cadastrado no MongoDB ou no Firebase — mensagem **"Email ja cadastrado"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X POST https://<host>/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Fulano da Silva",
    "email": "fulano@example.com",
    "phone": "11999999999",
    "password": "senha123"
  }'
```

---

### POST /auth/login
Autentica o usuário com e-mail/senha no Firebase, valida a existência do usuário correspondente no MongoDB e retorna os tokens de sessão. Aciona `LoginAuthUsecase` (`container.loginAuth`).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| email | string | sim | E-mail válido |
| password | string | sim | Senha do usuário |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Login efetuado com sucesso",
  "data": {
    "idToken": "<firebaseIdToken>",
    "refreshToken": "<firebaseRefreshToken>",
    "expiresIn": "3600",
    "customToken": "<firebaseCustomToken>",
    "user": {
      "id": "<mongoId>",
      "name": "Fulano da Silva",
      "email": "fulano@example.com",
      "phone": "+5511999999999",
      "user_type": "CLIENTE",
      "avatar_base64": null,
      "avatar_content_type": null
    }
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Corpo inválido (e-mail/senha ausentes ou formato de e-mail inválido) — `{ "message": "Dados inválidos", "errors": [...] }` |
| 403 | Autenticado no Firebase, porém sem usuário correspondente no MongoDB — mensagem **"Usuario nao encontrado no sistema"** |
| 500 | Falha de autenticação no provedor (ex.: credenciais inválidas) ou outro erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X POST https://<host>/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "fulano@example.com",
    "password": "senha123"
  }'
```

---

### POST /auth/forgot-password
Dispara o envio de e-mail de redefinição de senha via Firebase Authentication. Aciona `ForgotPasswordAuthUsecase` (`container.forgotPasswordAuth`).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| email | string | sim | E-mail do usuário que receberá o link de redefinição |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Email de redefinicao enviado com sucesso"
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | E-mail ausente ou em formato inválido — `{ "message": "Dados inválidos", "errors": [...] }` |
| 500 | Falha inesperada ao enviar o e-mail — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X POST https://<host>/api/v1/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{ "email": "fulano@example.com" }'
```

---

### GET /auth/me
Retorna o perfil do usuário autenticado (populado pelo `authMiddleware` em `req.authUser`). Não aciona um usecase próprio — apenas devolve os dados já resolvidos na autenticação.

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Perfil encontrado com sucesso",
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
| 401 | Header `Authorization` ausente/malformado — mensagem **"Token não fornecido"** |
| 401 | Token inválido ou expirado — mensagem **"Token inválido ou expirado"** |
| 401 | `req.authUser` ausente em runtime (caso defensivo no controller) — mensagem **"Usuario nao autenticado"** |
| 403 | Token válido, mas sem usuário correspondente no MongoDB — mensagem **"Usuário não encontrado no sistema"** |

**Exemplo de chamada**
```bash
curl -X GET https://<host>/api/v1/auth/me \
  -H "Authorization: Bearer <idToken>"
```

---

### PATCH /auth/me/profile
Atualiza nome e telefone do usuário autenticado, sincronizando Firebase Authentication e MongoDB. Aciona `UpdateAuthProfileUsecase` (`container.updateAuthProfile`).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| name | string | sim | Novo nome |
| phone | string | sim | Novo telefone; deve ser conversível para E.164 |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Perfil atualizado com sucesso",
  "data": {
    "id": "<mongoId>",
    "name": "Fulano da Silva Atualizado",
    "email": "fulano@example.com",
    "phone": "+5511988888888",
    "user_type": "CLIENTE",
    "avatar_base64": null,
    "avatar_content_type": null
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Corpo inválido (nome/telefone ausentes) — `{ "message": "Dados inválidos", "errors": [...] }` |
| 400 | Telefone não conversível para E.164 — mensagem **"Telefone invalido"** |
| 401 | Não autenticado (token ausente/inválido, ou `req.authToken` ausente em runtime) — mensagens do `authMiddleware` ou **"Usuario nao autenticado"** |
| 403 | Usuário do token não encontrado no MongoDB — mensagem **"Usuário não encontrado no sistema"** |
| 404 | Usuário não encontrado no MongoDB ao persistir a atualização — mensagem **"Usuario nao encontrado no MongoDB"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X PATCH https://<host>/api/v1/auth/me/profile \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{ "name": "Fulano da Silva Atualizado", "phone": "11988888888" }'
```

---

### PATCH /auth/me/avatar
Atualiza (cria ou substitui) o avatar do usuário autenticado a partir de uma imagem em base64. Aciona `UpdateUserAvatarUsecase` (`container.updateUserAvatar`).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| fileName | string | não | Nome do arquivo, máx. 120 caracteres |
| contentType | string | sim | Um de: `image/jpeg`, `image/png`, `image/webp` |
| base64 | string | sim | Conteúdo da imagem em base64 (aceita com ou sem prefixo `data:...;base64,`) |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Avatar atualizado com sucesso",
  "data": {
    "id": "<mongoId>",
    "name": "Fulano da Silva",
    "email": "fulano@example.com",
    "phone": "+5511999999999",
    "user_type": "CLIENTE",
    "avatar_base64": "data:image/png;base64,....",
    "avatar_content_type": "image/png"
  }
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Corpo inválido (contentType fora do enum, base64 ausente, fileName > 120 chars) — `{ "message": "Dados inválidos", "errors": [...] }` |
| 400 | Conteúdo base64 vazio após remover o prefixo `data:` — mensagem **"Avatar invalido"** |
| 400 | Buffer decodificado vazio — mensagem **"Avatar vazio"** |
| 400 | Imagem maior que 2MB — mensagem **"Avatar maior que 2MB"** |
| 401 | Não autenticado — mensagens do `authMiddleware` ou **"Usuario nao autenticado"** |
| 403 | Usuário do token não encontrado no MongoDB — mensagem **"Usuário não encontrado no sistema"** |
| 404 | Usuário não encontrado no MongoDB ao persistir o avatar — mensagem **"Usuario nao encontrado no MongoDB"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X PATCH https://<host>/api/v1/auth/me/avatar \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "fileName": "avatar.png",
    "contentType": "image/png",
    "base64": "iVBORw0KGgoAAAANSUhEUgA..."
  }'
```

---

### DELETE /auth/me/avatar
Remove o avatar do usuário autenticado. Aciona `RemoveUserAvatarUsecase` (`container.removeUserAvatar`).

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Avatar removido com sucesso",
  "data": {
    "id": "<mongoId>",
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
| 401 | Não autenticado — mensagens do `authMiddleware` ou **"Usuario nao autenticado"** |
| 403 | Usuário do token não encontrado no MongoDB — mensagem **"Usuário não encontrado no sistema"** |
| 404 | Usuário não encontrado no MongoDB — mensagem **"Usuario nao encontrado no MongoDB"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X DELETE https://<host>/api/v1/auth/me/avatar \
  -H "Authorization: Bearer <idToken>"
```

---

### PATCH /auth/me/email
Atualiza o e-mail do usuário autenticado, validando conflito tanto no MongoDB quanto no Firebase antes de aplicar a mudança. Aciona `UpdateAuthEmailUsecase` (`container.updateAuthEmail`). Se a atualização no MongoDB falhar após o e-mail já ter sido alterado no Firebase, o e-mail é revertido (`currentEmail`) no Firebase.

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| email | string | sim | Novo e-mail (formato válido) |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Email atualizado com sucesso",
  "data": {
    "id": "<mongoId>",
    "name": "Fulano da Silva",
    "email": "novo-email@example.com",
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
| 400 | E-mail ausente ou em formato inválido — `{ "message": "Dados inválidos", "errors": [...] }` |
| 401 | Não autenticado — mensagens do `authMiddleware` ou **"Usuario nao autenticado"** |
| 403 | Usuário do token não encontrado no MongoDB — mensagem **"Usuário não encontrado no sistema"** |
| 404 | Usuário não encontrado no MongoDB ao persistir — mensagem **"Usuario nao encontrado no MongoDB"** |
| 409 | E-mail já cadastrado por outro usuário (no MongoDB ou no Firebase) — mensagem **"Email ja cadastrado"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X PATCH https://<host>/api/v1/auth/me/email \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{ "email": "novo-email@example.com" }'
```

---

### PATCH /auth/me/password
Atualiza a senha do usuário autenticado diretamente no Firebase Authentication. Aciona `UpdateAuthPasswordUsecase` (`container.updateAuthPassword`).

**Corpo da requisição**

| campo | tipo | obrigatório | descrição |
|---|---|---|---|
| password | string | sim | Nova senha, mínimo de 6 caracteres |

**Resposta de sucesso**: `200 OK`
```json
{
  "message": "Senha atualizada com sucesso"
}
```

**Erros possíveis**

| código HTTP | causa |
|---|---|
| 400 | Senha ausente ou com menos de 6 caracteres — `{ "message": "Dados inválidos", "errors": [...] }` |
| 401 | Não autenticado — mensagens do `authMiddleware` ou **"Usuario nao autenticado"** |
| 403 | Usuário do token não encontrado no MongoDB — mensagem **"Usuário não encontrado no sistema"** |
| 500 | Erro inesperado — mensagem **"Erro interno no servidor"** |

**Exemplo de chamada**
```bash
curl -X PATCH https://<host>/api/v1/auth/me/password \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{ "password": "novaSenha123" }'
```

## Referências
- `src/main.ts` (prefixo global `/api/v1`)
- `src/applications/routes/routes.ts` (montagem de `/auth`)
- `src/applications/routes/auth.route.ts`
- `src/applications/middlewares/auth.middleware.ts` (`authMiddleware`)
- `src/applications/controllers/auth/register-auth.controller.ts`
- `src/applications/controllers/auth/login-auth.controller.ts`
- `src/applications/controllers/auth/forgot-password-auth.controller.ts`
- `src/applications/controllers/auth/read-auth-profile.controller.ts`
- `src/applications/controllers/auth/update-auth-profile.controller.ts`
- `src/applications/controllers/auth/update-auth-avatar.controller.ts`
- `src/applications/controllers/auth/remove-auth-avatar.controller.ts`
- `src/applications/controllers/auth/update-auth-email.controller.ts`
- `src/applications/controllers/auth/update-auth-password.controller.ts`
- `src/applications/controllers/shared/handle-http-error.ts`
- `src/applications/dto/auth.dto.ts`
- `src/domain/usecases/auth/register-auth.usecase.ts`
- `src/domain/usecases/auth/login-auth.usecase.ts`
- `src/domain/usecases/auth/forgot-password-auth.usecase.ts`
- `src/infra/adapters/auth/sign-in-with-password.adapter.ts` (REST `signInWithPassword` — login)
- `src/infra/adapters/auth/send-password-reset-email.adapter.ts` (REST `sendOobCode` — reset de senha)
- `src/infra/adapters/auth/identity-toolkit.ts` (`identityToolkitBaseUrl()` — host do Google vs emulador)
- `src/config/firebase.ts` (`ensureFirebaseApp` — service account vs modo emulador)
- `src/config/env.ts` (`FIREBASE_AUTH_EMULATOR_HOST`, opcional)
- `src/domain/usecases/auth/update-auth-profile.usecase.ts`
- `src/domain/usecases/auth/update-auth-email.usecase.ts`
- `src/domain/usecases/auth/update-auth-password.usecase.ts`
- `src/domain/usecases/auth/authenticate-request.usecase.ts` (`AuthenticateRequestUsecase`)
- `src/domain/usecases/user-avatar/update-user-avatar.usecase.ts`
- `src/domain/usecases/user-avatar/remove-user-avatar.usecase.ts`
- `src/domain/usecases/shared/phone-number.ts` (`normalizePhoneNumberToE164`)
- `src/domain/errors.ts` (`DomainError` e subclasses)
- `src/domain/models/user.model.ts` (`IUser`)
- `src/config/container.ts` (injeção de dependências pós-refatoração hexagonal — não altera o contrato documentado acima)
