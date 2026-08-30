# Testes Exploratórios Manuais — 001 Conformidade Hexagonal (serviço `beach-center-bff-usuarios`)

> Gerado por `/speckit-test`. Roteiro de QA manual. Não substitui os testes automatizados
> (`/speckit-unit-tests` — 191 testes, 99,5% cobertura).
>
> **Natureza da task:** refatoração estrutural **sem mudança de contrato**. O foco destes
> cenários é (A) **não-regressão** dos endpoints de `usuarios` e (B) **conformidade arquitetural
> observável** (Princípio II). `pagamentos` e `agendamentos` ficam para as fases 2 e 3.

---

## Escopo e pré-condições

| Item | Detalhe |
|---|---|
| Serviço | `services/beach-center-bff-usuarios` (porta padrão 5000, base `/api/v1`) |
| Subir | `npm run dev` (ts-node `src/main.ts`) **ou** `npm run build && npm start` |
| Banco | MongoDB acessível via `DB` no `.env`; collection `usuarios` |
| Firebase | `.env` com `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`, `FIREBASE_API_KEY` |
| Ferramenta | Postman/Insomnia/`curl`; para os fluxos autenticados é preciso um `idToken` do Firebase (via `POST /auth/login`) |
| Seed | 1 usuário `ADMIN` (`user_type: "ADMIN"`), 1 usuário `CLIENTE`, ambos com registro no Firebase Auth **e** no Mongo (mesmo `id_firestore`) |
| Baseline | Se possível, capturar as respostas dos endpoints **antes** do merge (branch `master`) para comparar shape/status/mensagem |

### Endpoints sob teste

| # | Método | Rota | Auth |
|---|---|---|---|
| E1 | POST | `/api/v1/auth/register` | pública |
| E2 | POST | `/api/v1/auth/login` | pública |
| E3 | POST | `/api/v1/auth/forgot-password` | pública |
| E4 | GET | `/api/v1/auth/me` | Bearer |
| E5 | PATCH | `/api/v1/auth/me/profile` | Bearer |
| E6 | PATCH | `/api/v1/auth/me/avatar` | Bearer |
| E7 | DELETE | `/api/v1/auth/me/avatar` | Bearer |
| E8 | PATCH | `/api/v1/auth/me/email` | Bearer |
| E9 | PATCH | `/api/v1/auth/me/password` | Bearer |
| E10 | GET | `/api/v1/usuarios` | Bearer + ADMIN |
| E11 | GET | `/api/v1/usuarios/:id` | Bearer + ADMIN |
| E12 | PATCH | `/api/v1/usuarios/:id` | Bearer + ADMIN |
| E13 | DELETE | `/api/v1/usuarios/:id` | Bearer + ADMIN |

---

## A. Caminhos felizes (não-regressão de contrato)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-01 | Nenhum usuário com o e-mail `qa.novo@example.com` (Mongo **e** Firebase) | `POST /auth/register` com `{name, email: "QA.Novo@Example.com ", phone: "(11) 98765-4321", password: "senha123"}` | **201**; `message: "Usuario registrado com sucesso"`; `data.user` contém `id, name, email (minúsculo, sem espaço), phone, user_type: "CLIENTE", avatar_base64, avatar_content_type` e **NÃO** contém `id_firestore`; `data.customToken` presente; usuário criado no Mongo e no Firebase com o mesmo `id_firestore` |
| H-02 | Usuário `CLIENTE` seed existe | `POST /auth/login` com `{email, password}` corretos | **200**; `message: "Login efetuado com sucesso"`; `data` = `{idToken, refreshToken, expiresIn, customToken, user}`; `data.user` tem shape `IUser` (`id` string, sem `__v`) e `user_type` correto |
| H-03 | `FIREBASE_API_KEY` válida; e-mail de usuário existente | `POST /auth/forgot-password` com `{email}` | **200**; `message: "Email de redefinicao enviado com sucesso"`; e-mail de redefinição chega na caixa |
| H-04 | `idToken` válido de um usuário | `GET /auth/me` com `Authorization: Bearer <idToken>` | **200**; `message: "Perfil encontrado com sucesso"`; `data` = `{id, id_firestore, name, email, phone, user_type, avatar_base64, avatar_content_type}` (mesmos campos de antes) |
| H-05 | `idToken` válido | `PATCH /auth/me/profile` com `{name: "Nome Novo", phone: "11 91111-2222"}` | **200**; `message: "Perfil atualizado com sucesso"`; `data` reflete `name`/`phone` novos; Firebase `displayName`/`phoneNumber` atualizados; Mongo atualizado |
| H-06 | `idToken` válido; imagem PNG pequena (< 2MB) em base64 | `PATCH /auth/me/avatar` com `{fileName: "a.png", contentType: "image/png", base64: "<data URL ou base64 puro>"}` | **200**; `message: "Avatar atualizado com sucesso"`; `data.avatar_base64` = `data:image/png;base64,...`; `data.avatar_content_type: "image/png"` |
| H-07 | Usuário com avatar definido; `idToken` válido | `DELETE /auth/me/avatar` | **200**; `message: "Avatar removido com sucesso"`; `data.avatar_base64: null`, `data.avatar_content_type: null` |
| H-08 | `idToken` válido; e-mail `qa.trocado@example.com` livre | `PATCH /auth/me/email` com `{email: "QA.Trocado@Example.com"}` | **200**; `message: "Email atualizado com sucesso"`; `data.email` minúsculo; Firebase e Mongo com o novo e-mail |
| H-09 | `idToken` válido | `PATCH /auth/me/password` com `{password: "novaSenha123"}` | **200**; `message: "Senha atualizada com sucesso"`; login subsequente só funciona com a nova senha |
| H-10 | `idToken` de **ADMIN** | `GET /usuarios` | **200**; `message: "Usuários listados com sucesso"`; `data` = array de `IUser` com `id` string |
| H-11 | `idToken` de ADMIN; `?name=an&email=exam&user_type=CLIENTE` | `GET /usuarios?name=an&email=exam&user_type=CLIENTE` | **200**; filtra por `user_type` na query + substring **insensível a acento/caixa** em nome e e-mail (ex.: `name=nautica` acha "Náutica") |
| H-12 | `idToken` de ADMIN; `:id` válido (24 hex) de usuário existente | `GET /usuarios/:id` | **200**; `message: "Usuário encontrado com sucesso"`; `data` = `IUser` |
| H-13 | `idToken` de ADMIN; `:id` de um usuário `CLIENTE` | `PATCH /usuarios/:id` com `{name, email, phone, user_type: "ADMIN"}` | **200**; `message: "Usuário encontrado com sucesso"` (mensagem pré-existente, mantida); `data.user_type: "ADMIN"`; Firebase `displayName/email/phoneNumber` atualizados |
| H-14 | `idToken` de ADMIN; `:id` de usuário descartável (Mongo + Firebase) | `DELETE /usuarios/:id` | **200**; `message: "Usuário deletado com sucesso"`; `data` = snapshot do usuário; **removido do Firebase antes** do Mongo; ambos apagados |

---

## B. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-01 | — | `POST /auth/register` com `body: {}` | **400**; `message: "Dados inválidos"`; `errors: [...]` (lista de campos) |
| X-02 | — | `POST /auth/register` com `phone: "123"` (inválido) | **400**; `message: "Telefone invalido"`; **nada** criado no Firebase/Mongo |
| X-03 | E-mail já existe no **Mongo** | `POST /auth/register` com esse e-mail | **409**; `message: "Email ja cadastrado"`; nada criado |
| X-04 | E-mail já existe no **Firebase** mas não no Mongo | `POST /auth/register` com esse e-mail | **409**; `message: "Email ja cadastrado"` |
| X-05 | Simular falha do Mongo após criar no Firebase (ex.: derrubar Mongo entre as chamadas, ou índice único de e-mail em corrida) | `POST /auth/register` | Erro (409/500); **usuário do Firebase sofre rollback** (`deleteUser`) — não fica órfão |
| X-06 | Credenciais erradas | `POST /auth/login` com senha errada | **401**; `message: "Credenciais invalidas"` |
| X-07 | Usuário existe no Firebase mas **não** no Mongo | `POST /auth/login` com credenciais válidas | **403**; `message: "Usuario nao encontrado no sistema"` |
| X-08 | `.env` **sem** `FIREBASE_API_KEY` | `POST /auth/login` | **500**; `message: "FIREBASE_API_KEY nao configurada"` |
| X-09 | `.env` sem `FIREBASE_API_KEY` | `POST /auth/forgot-password` | **500**; `message: "FIREBASE_API_KEY nao configurada"` |
| X-10 | E-mail inexistente / API key inválida | `POST /auth/forgot-password` | **400**; `message: "Nao foi possivel enviar o email de redefinicao"` |
| X-11 | — | `GET /auth/me` **sem** header `Authorization` | **401**; `message: "Token não fornecido"` |
| X-12 | — | `GET /auth/me` com `Authorization: Token abc` (sem `Bearer `) | **401**; `message: "Token não fornecido"` |
| X-13 | — | `GET /auth/me` com `Authorization: Bearer token-invalido` | **401**; `message: "Token inválido ou expirado"` |
| X-14 | `idToken` válido de um usuário que foi apagado do Mongo | `GET /auth/me` | **403**; `message: "Usuário não encontrado no sistema"` |
| X-15 | `idToken` válido de um **CLIENTE** | `GET /usuarios` | **403**; `message: "Acesso negado"` |
| X-16 | `idToken` de ADMIN; `:id` = `"123"` (não é ObjectId) | `GET /usuarios/123` | **400**; `message: "ID inválido"` |
| X-17 | `idToken` de ADMIN; `:id` bem formado mas inexistente | `GET /usuarios/<24hex inexistente>` | **404**; `message: "Usuário não encontrado"` |
| X-18 | `idToken` de ADMIN; `PATCH /usuarios/:id` com `phone` inválido | `PATCH /usuarios/:id` | **400**; `message: "Telefone invalido"` |
| X-19 | `idToken` de ADMIN; `PATCH /usuarios/:id` com `user_type: "ROOT"` | `PATCH /usuarios/:id` | **400**; `message: "Dados inválidos"` (yup `oneOf`) |
| X-20 | `idToken` válido; e-mail que já pertence a **outro** usuário | `PATCH /auth/me/email` | **409**; `message: "Email ja cadastrado"` |
| X-21 | `idToken` válido; simular falha do Mongo no update de e-mail | `PATCH /auth/me/email` | Erro (500); **e-mail no Firebase sofre rollback** para o `currentEmail` |
| X-22 | `idToken` válido; imagem base64 > 2MB | `PATCH /auth/me/avatar` | **400**; `message: "Avatar maior que 2MB"` |
| X-23 | `idToken` válido; `base64: " "` (sem bytes) | `PATCH /auth/me/avatar` | **400**; `message: "Avatar vazio"` |
| X-24 | `idToken` válido; `contentType: "image/gif"` | `PATCH /auth/me/avatar` | **400**; `message: "Dados inválidos"` (yup `oneOf`) |
| X-25 | `idToken` válido; senha `"123"` (< 6) | `PATCH /auth/me/password` | **400**; `message: "Dados inválidos"` |

---

## C. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| EC-01 | — | `POST /auth/register` com telefone em vários formatos: `+5511987654321`, `5511987654321`, `11987654321`, `(11) 98765-4321`, `1133334444` (10 díg.) | Todos normalizados para E.164 (`+55...`); `phone` salvo no Mongo é o **valor original** enviado, `phoneNumber` no Firebase é o E.164 |
| EC-02 | — | `POST /auth/register` com telefone de 9 dígitos (`987654321`) | **400** "Telefone invalido" |
| EC-03 | — | `GET /usuarios?name=` (vazio) e `?name=%20` (só espaço) | Retorna **todos** (filtro textual ignorado quando vazio/trim vazio) |
| EC-04 | Usuário com nome "José da Silva" | `GET /usuarios?name=jose da silva` | Encontra (case + acento insensível) |
| EC-05 | — | `POST /auth/register` com `email: "  ana@x.com  "` e depois `POST /auth/login` com `email: "ANA@X.COM"` | Registro normaliza (trim+lower); login usa o e-mail como digitado (o Identity Toolkit trata) — validar que login funciona |
| EC-06 | ADMIN tenta se auto-rebaixar | `PATCH /usuarios/<próprio id>` com `user_type: "CLIENTE"` | **200** — a API permite (não há regra de proteção); documentar como comportamento atual |
| EC-07 | Dois `PATCH /auth/me/email` simultâneos para o mesmo e-mail novo, de usuários diferentes | Executar em paralelo | No máximo um sucede; o outro → **409** (índice único de e-mail) ou rollback do Firebase |
| EC-08 | `DELETE /usuarios/:id` de usuário que existe no Mongo mas **não** no Firebase | `DELETE /usuarios/:id` | **200** — `auth/user-not-found` é engolido (idempotente); Mongo apagado |
| EC-09 | `PATCH /auth/me/avatar` com data URL completa (`data:image/png;base64,iVBOR...`) **e** com base64 puro (`iVBOR...`) | Ambos | Resultado idêntico: `avatar_base64` sempre `data:image/png;base64,<dados>` |
| EC-10 | Token expirado (aguardar > 1h ou forjar `exp` passado) | `GET /auth/me` | **401** "Token inválido ou expirado" |
| EC-11 | `requireRole` — token válido cujo usuário foi apagado do Mongo entre `authMiddleware` e `requireRole` | `GET /usuarios` | **403** "Usuario nao encontrado no sistema" |
| EC-12 | Mongo indisponível durante uma requisição autenticada | `GET /auth/me` com Mongo down | **500** "Erro interno no servidor" (⚠️ **mudança**: antes retornava 401) |

---

## D. Conformidade arquitetural (Princípio II — específico desta task)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| CA-01 | Repo `usuarios` | `npm run lint` | **0 erros**. A regra `no-restricted-imports` está ativa |
| CA-02 | — | `grep -rn "infra/" src/domain src/applications` | **Nenhuma** ocorrência (AC-1/AC-2) |
| CA-03 | — | Adicionar temporariamente `import x from "../../infra/schemas/user.schema"` em qualquer arquivo de `src/domain/` e rodar `npm run lint` | ESLint **falha** com a mensagem "domain nao pode importar de infra..."; reverter |
| CA-04 | — | Inspecionar `src/infra/adapters/**/*.adapter.ts` | Cada um `implements` um port de `domain/ports/output`; só acessa `schemas`/`firebase-admin`/`axios`; **sem** `if` de regra de negócio (limites, validações, cálculo) |
| CA-05 | — | `ls src/infra/adapters/*/` e `src/infra/adapters/auth/` | Um arquivo `.adapter.ts` por verbo/ação (AC-5); nenhum adapter multi-ação |
| CA-06 | — | `ls src/config/` e `cat src/main.ts` | Existem `env.ts`, `container.ts`, `firebase.ts` e `main.ts`; **não** há `src/applications/config/**` (AC-7) |
| CA-07 | — | `npm run test:coverage` | Todos passam; cobertura global ≥ 80% (real: ~99,5%); `coverageThreshold` falha o comando se cair abaixo (AC-12) |
| CA-08 | — | `npm run build` | Gera `dist/`; `node dist/main.js` sobe o servidor e conecta no Mongo |

---

## E. Checklist de regressão (fluxos vizinhos)

| ID | Fluxo | Verificação |
|---|---|---|
| R-01 | `beach-center-app` — tela de **Login** | Login pelo app funciona; `databaseUser`/`user_type` populados no contexto; redireciona conforme o perfil |
| R-02 | `beach-center-app` — tela de **Cadastro** | Registro pelo app → `signInWithCustomToken` com `data.customToken` → `updateProfile(displayName)` funciona |
| R-03 | `beach-center-app` — **Esqueci a senha** | Recebe o e-mail; troca a senha; login com a nova |
| R-04 | `beach-center-app` — **Perfil** (`/auth/me`, editar nome/telefone/e-mail/senha, avatar) | Todos os fluxos da página `Profile.page.tsx` funcionam; `get-profile.service`, `update-profile/email/password/avatar` retornam o esperado |
| R-05 | `beach-center-app` — **Gestão de usuários (admin)** | Listar/filtrar/editar/excluir usuários; `list-users.service` + `user-normalizer` (`id ?? _id`) continua ok mesmo com `id` agora sempre string |
| R-06 | Outros serviços que chamam `bff-usuarios` | `agendamentos`/`pagamentos` que validam token ou buscam usuário via este serviço — confirmar que o contrato `/auth/me` / headers não mudou |
| R-07 | Infra local (`beach-center-server`) | Container de `usuarios` sobe com o novo entrypoint (`src/main.ts` via ts-node **ou** `dist/main.js`); hot-reload funciona (Princípio IV) |
| R-08 | Deploy / CI | Pipeline roda `npm ci && npm run lint && npm test` sem erro; `npm run build` gera artefato |

---

## Rastreabilidade AC → cenário

| AC | Cenários | Cobertura automatizada equivalente? |
|---|---|---|
| **AC-1** Domínio não conhece infra | CA-02, CA-03 | ✅ ESLint `no-restricted-imports` + `container.spec` |
| **AC-2** Applications não conhece infra | CA-02, CA-03 | ✅ ESLint + `*.controller.spec` (mock do container) |
| **AC-3** Usecases dependem de ports | CA-04 (inspeção) | ✅ `*.usecase.spec` (mocks `jest.Mocked<IPort>`) |
| **AC-4** Adapters implementam ports, sem regra | CA-04 | ✅ `*.adapter.spec` |
| **AC-5** Um adapter por verbo/ação | CA-05 | ➖ estrutural (inspeção) |
| **AC-6** Regra de negócio migrada (agendamentos) | — | ⏳ **Fase 3** — fora do escopo de `usuarios` |
| **AC-7** `config/` + `main.ts` padronizados | CA-06 | ✅ `env.spec`, `firebase.spec`, `container.spec` |
| **AC-8** Nomes de arquivo (agendamentos) | — | ⏳ **Fase 3** |
| **AC-9** Contratos de endpoint inalterados | H-01..H-14, X-01..X-25, EC-* | 🟡 parcial — `*.controller.spec` afirmam status+message+shape; **comparação com baseline `master` é manual** |
| **AC-10** Middlewares de auth preservados | X-11..X-15, EC-11, `routes.spec` | ✅ `routes.spec` + `auth.middleware.spec` |
| **AC-11** ESLint configurado e limpo | CA-01 | ✅ |
| **AC-12** Cobertura ≥ 80% | CA-07 | ✅ 99,5% |
| **AC-13** Testes de domínio mockam portas | — | ✅ `*.usecase.spec` |
| **AC-14** Controllers e adapters cobertos | — | ✅ 14 + 18 specs |
| **AC-15** `@wip` ignorado | — | ✅ `jest.config` (nenhum `@wip` presente) |
| **AC-16** Ambiente local sobe com hot-reload | CA-08, R-07 | ➖ manual (`/run-server`) |

### Cenários SEM cobertura automatizada equivalente (prioridade de QA manual)

- **AC-9 / não-regressão real:** H-01..H-14 e X-* — os specs testam os controllers isolados com mock do container; **só o teste manual contra `master` confirma que o shape/mensagem do endpoint real não mudou** (em especial: `login`/`profile`/`email` que antes devolviam doc Mongo cru e agora `IUser`).
- **Rollbacks:** X-05 (register), X-21 (email) — exigem injeção de falha no Mongo, difícil de reproduzir em unit test com fidelidade.
- **Concorrência:** EC-07.
- **Mudança de comportamento EC-12** (erro de infra no auth → 500, era 401) — confirmar que nenhum cliente depende do 401.
- **Integração com o `beach-center-app`:** R-01..R-06.
- **Infra/CI:** R-07, R-08, CA-08, AC-16.
