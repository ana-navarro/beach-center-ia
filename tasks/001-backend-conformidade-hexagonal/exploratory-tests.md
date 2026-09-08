# Testes Exploratórios Manuais — 001 Conformidade Hexagonal

> Gerado por `/speckit-test`. Roteiro de QA manual dos 3 serviços da task (`usuarios`,
> `pagamentos`, `agendamentos`). Não substitui os testes automatizados de cada serviço
> (`/speckit-unit-tests`).
>
> **Natureza da task:** refatoração estrutural **sem mudança de contrato** (Princípio II —
> arquitetura hexagonal). O foco de todos os cenários abaixo é (A) **não-regressão** dos
> endpoints existentes e (B) **conformidade arquitetural observável**. Nenhuma regra de negócio
> nova foi introduzida — onde um comportamento pré-existente parecia um bug, ele foi
> **preservado deliberadamente** (AC-9) e sinalizado no cenário correspondente em vez de
> corrigido silenciosamente.

---

## Fase 1 — usuarios (serviço `beach-center-bff-usuarios`)

> Roteiro de QA manual. Não substitui os testes automatizados (`/speckit-unit-tests` — 191
> testes, 99,5% cobertura).

### Escopo e pré-condições

| Item | Detalhe |
|---|---|
| Serviço | `services/beach-center-bff-usuarios` (porta padrão 5000, base `/api/v1`) |
| Subir | `npm run dev` (ts-node `src/main.ts`) **ou** `npm run build && npm start` |
| Banco | MongoDB acessível via `DB` no `.env`; collection `usuarios` |
| Firebase | `.env` com `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`, `FIREBASE_API_KEY` |
| Ferramenta | Postman/Insomnia/`curl`; para os fluxos autenticados é preciso um `idToken` do Firebase (via `POST /auth/login`) |
| Seed | 1 usuário `ADMIN` (`user_type: "ADMIN"`), 1 usuário `CLIENTE`, ambos com registro no Firebase Auth **e** no Mongo (mesmo `id_firestore`) |
| Baseline | Se possível, capturar as respostas dos endpoints **antes** do merge (branch `master`) para comparar shape/status/mensagem |

#### Endpoints sob teste

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

### A. Caminhos felizes (não-regressão de contrato)

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

### B. Fluxos de exceção

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

### C. Edge cases

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

### D. Conformidade arquitetural (Princípio II — específico desta task)

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

### E. Checklist de regressão (fluxos vizinhos)

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

### Rastreabilidade AC → cenário

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

#### Cenários SEM cobertura automatizada equivalente (prioridade de QA manual)

- **AC-9 / não-regressão real:** H-01..H-14 e X-* — os specs testam os controllers isolados com mock do container; **só o teste manual contra `master` confirma que o shape/mensagem do endpoint real não mudou** (em especial: `login`/`profile`/`email` que antes devolviam doc Mongo cru e agora `IUser`).
- **Rollbacks:** X-05 (register), X-21 (email) — exigem injeção de falha no Mongo, difícil de reproduzir em unit test com fidelidade.
- **Concorrência:** EC-07.
- **Mudança de comportamento EC-12** (erro de infra no auth → 500, era 401) — confirmar que nenhum cliente depende do 401.
- **Integração com o `beach-center-app`:** R-01..R-06.
- **Infra/CI:** R-07, R-08, CA-08, AC-16.


---

## Fase 2 — pagamentos (serviço `beach-center-bff-pagamentos`)

> Gerado por `/speckit-test`. Roteiro de QA manual. Não substitui os testes automatizados
> (`/speckit-unit-tests` — 37 arquivos `*.spec.ts`, **156 testes**, cobertura **98,3% stmts /
> 82,2% branch / 96,3% funcs / 98,8% lines**).
>
> **Natureza da task:** refatoração estrutural **sem mudança de contrato** (decisão 6 do plano).
> O foco destes cenários é (A) **não-regressão** dos endpoints de `pagamentos` e (B)
> **conformidade arquitetural observável** (Princípio II). `usuarios` (fase 1) já foi coberto em
> `exploratory-tests.md`; `agendamentos` fica para a fase 3.

---

### Escopo e pré-condições

| Item | Detalhe |
|---|---|
| Serviço | `services/beach-center-bff-pagamentos` (porta padrão **5001**, base `/api/v1`) |
| Subir | `npm run dev` (nodemon + `ts-node src/main.ts`) **ou** `npm run build && npm start` |
| Banco | MongoDB acessível via `DB` no `.env` — **mesma base compartilhada** de `usuarios`: coleção `usuarios` (lida via `UserModel`, `find-user-by-firestore-id.adapter.ts`, **nunca escrita** por este serviço), `manual_payment_validations` (comprovantes manuais), `payment_transaction_history` (log de auditoria interno — não exposto por nenhum endpoint hoje, ver nota em E) |
| Serviço dependente | `services/beach-center-bff-agendamentos` **precisa estar rodando** — `pagamentos` chama de volta via HTTP (`AGENDAMENTOS_API_URL`, default `http://localhost:5001/api/v1` ⚠️ conferir se não colide com a própria porta do serviço; `AGENDAMENTOS_INTERNAL_API_KEY` para as chamadas internas de leitura/atualização) |
| Provedor de pagamento | `PAYMENT_PROVIDER=mock` (padrão, se ausente) → `MockCheckoutAdapter` aprova o checkout automaticamente sem chamar a Getnet. `PAYMENT_PROVIDER=getnet` → `GetnetCheckoutAdapter` real (sandbox), exige `GETNET_CLIENT_ID`, `GETNET_CLIENT_SECRET`, `GETNET_SELLER_ID`. **Atenção:** o **reembolso está sempre mockado** (`MockRefundAdapter`) **independente** de `PAYMENT_PROVIDER` — é uma decisão deliberada registrada em `config/container.ts` ("O reembolso permanece mockado; a integração Getnet real fica isolada no adapter"), não um bug. Não há hoje um caminho de reembolso real via Getnet acessível pela API. |
| Env vars (`config/env.ts`) | `PORT` (opcional, default 5001), `DB` (obrigatório), `PAYMENT_PROVIDER`, `GETNET_ENV` (`sandbox`\|`production`), `GETNET_CLIENT_ID`/`GETNET_CLIENT_SECRET`/`GETNET_SELLER_ID` (obrigatórios só se `PAYMENT_PROVIDER=getnet`), `GETNET_WEBHOOK_TOKEN` **ou** `PAGAMENTOS_WEBHOOK_TOKEN` (token do webhook — ver ⚠️ em X-16), `GETNET_REFUND_PATH_TEMPLATE`/`GETNET_REFUND_METHOD` (só usados pelo adapter real, não roteado hoje), `FRONTEND_URL`, `API_URL` (usada para montar a `notification_url` que a Getnet chama de volta), `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY`, `PAGAMENTOS_INTERNAL_API_KEY` (chave que o **próprio** `pagamentos` aceita em `/refunds` via `authOrInternalApiKey`), `GOOGLE_DRIVE_CLIENT_EMAIL`/`GOOGLE_DRIVE_PRIVATE_KEY`/`GOOGLE_DRIVE_FOLDER_ID` (se ausentes, upload de comprovante cai em modo `storage: "database"` — grava o base64 direto no Mongo, sem chamar a API do Drive), `FIREBASE_PROJECT_ID`/`FIREBASE_CLIENT_EMAIL`/`FIREBASE_PRIVATE_KEY` (verificação de `idToken`) |
| Firebase / seed de usuário | Reaproveitar os usuários seed da fase 1 (`usuarios`): 1 `ADMIN`, 1 `CLIENTE`, ambos com `idToken` válido — como `pagamentos` lê a **mesma** coleção `usuarios`, não é preciso recriar nada |
| Seed de reserva | Uma reserva `pending` em `agendamentos` é necessária para testar refund/webhook/comprovante manual **isoladamente** — pode ser obtida (a) criando via `POST /checkout` com `PAYMENT_PROVIDER=getnet` (fica pending) ou (b) diretamente pela API de `agendamentos` (`POST /reservas`). Para testar o caminho feliz completo de checkout, basta ter horários (`scheduling_id`) livres em `agendamentos` |
| Ferramenta | Postman/Insomnia/`curl`; para rotas autenticadas, `idToken` do Firebase (via `POST /auth/login` no serviço `usuarios`) |
| Baseline | Capturar respostas antes do merge (branch `master`) para comparar shape/status/mensagem, igual à fase 1 |

#### Endpoints sob teste

| # | Método | Rota | Auth |
|---|---|---|---|
| E1 | GET | `/api/v1/payment-methods` | Bearer (qualquer usuário autenticado) |
| E2 | POST | `/api/v1/checkout` | Bearer |
| E3 | POST | `/api/v1/refunds` | Bearer **ou** `x-api-key: PAGAMENTOS_INTERNAL_API_KEY` |
| E4 | POST | `/api/v1/transaction-history` | Bearer (submissão de comprovante manual autenticada) |
| E5 | POST | `/api/v1/public/transaction-history` | pública (submissão via link público de reserva, requer `public_reserve_token`) |
| E6 | GET | `/api/v1/transaction-history` | Bearer + ADMIN (lista comprovantes manuais) |
| E7 | GET | `/api/v1/transaction-history/:id` | Bearer + ADMIN |
| E8 | GET | `/api/v1/transaction-history/:id/proof` | Bearer + ADMIN |
| E9 | PATCH | `/api/v1/transaction-history/:id/review` | Bearer + ADMIN |
| E10 | POST | `/api/v1/webhooks/getnet` | `?token=` query **ou** `x-api-key` = `GETNET_WEBHOOK_TOKEN`/`PAGAMENTOS_WEBHOOK_TOKEN` |

> **Nota de nomenclatura:** apesar do nome `transaction-history`, os endpoints E4–E9 operam sobre
> **comprovantes de pagamento manual** (`IManualPayment`, coleção `manual_payment_validations`),
> não sobre o log de auditoria interno (`ITransactionHistory`/`payment_transaction_history`).
> Isso é comportamento **pré-existente**, preservado pela refatoração (AC-9) — ver nota em E.

---

### A. Caminhos felizes (não-regressão de contrato)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-01 | `idToken` válido (qualquer `user_type`) | `GET /payment-methods` | **200**; `message: "Formas de pagamento consultadas com sucesso"`; `data: [{value:"PIX",label:"PIX",enabled:true},{value:"CREDIT_CARD",label:"Cartao de credito",enabled:true}]` |
| H-02 | `PAYMENT_PROVIDER=mock` (ou ausente); `idToken` válido; `scheduling_id` de 1–3 horários livres em `agendamentos` | `POST /checkout` com `{name, email, phone, scheduling_id, total, equipment, payment_method: "PIX"}` | **201**; `message: "Checkout gerado com sucesso"`; `data.payment_status: "APPROVED"`; `data.mock: true`; `data.payment_id` iniciando com `mock_`; `data.payment_url` **ausente** (redirect vazio); `data.reserve.status: "approved"`; reserva criada em `agendamentos` (`POST /reservas`) e depois `PATCH /reservas/:id/status {status:"approved"}` **e** `PATCH /reservas/:id/payment` com `checkout_id`/`payment_id`/`payment_method` |
| H-03 | `PAYMENT_PROVIDER=getnet` (sandbox); credenciais Getnet válidas; `idToken` válido | `POST /checkout` com `payment_method: "CREDIT_CARD"` | **201**; `message: "Checkout gerado com sucesso"`; `data.payment_status: "PENDING"`; `data.payment_url` presente (URL de redirecionamento Getnet); `data.reserve.status` continua `"pending"` (só vira `approved` via webhook) |
| H-04 | Reserva com `payment_id` (de um checkout aprovado, H-02) | `POST /refunds` com `{reserve_id, payment_id, amount, payment_method: "PIX"}`, `Authorization: Bearer <idToken ADMIN>` | **200**; `message: "Reembolso processado com sucesso"`; `data: {refund_id: "refund_<payment_id>_<8 chars>", refund_status: "approved", refunded_at: <ISO>}` (sempre via mock, ver nota em Escopo); `agendamentos` recebe `PATCH /reservas/:id/payment` com `refund_id`/`refund_status`/`refunded_at` |
| H-04b | Mesmo cenário de H-04 | `POST /refunds` com header `x-api-key: <PAGAMENTOS_INTERNAL_API_KEY>` (sem `Authorization`) | **200** — mesmo resultado; confirma que `authOrInternalApiKey` aceita a chave interna sem exigir Firebase |
| H-05 | Reserva `pending` (order_id conhecido) | `POST /webhooks/getnet?token=<GETNET_WEBHOOK_TOKEN>` com `{"order_id": "<reserve_id>", "payment_id": "getnet-pay-1", "status": "APPROVED"}` | **200** (corpo vazio, sempre, mesmo se algo falhar internamente); reserva em `agendamentos` passa a `status: "approved"`; `PATCH /reservas/:id/payment {payment_id}` chamado |
| H-06 | `idToken` de CLIENTE dono de uma reserva `pending`; duração 1h → `amount:80` | `POST /transaction-history` com `{reserve_id, reserve_number, amount:80, duration_hours:1, user, slots:[<1 slot batendo com scheduling_id da reserva>], proof_file:{file_name,mime_type:"image/png",base64}}` | **201**; `message: "Comprovante enviado para validacao"`; `data`: `IManualPayment` com `status:"pending"`, `proof.storage:"database"` (se Drive não configurado) ou `"google_drive"` |
| H-06b | Reserva com link público válido (`public_reserve_token`), sem autenticação | `POST /public/transaction-history` com o mesmo corpo de H-06 + `public_reserve_token` | **201** — mesmo shape de H-06; confirma que o fluxo público (sem Bearer) autoriza via `checkPublicLinkAuthorizationPort` contra `agendamentos` |
| H-07 | Comprovante de H-06 existente | `GET /transaction-history?status=pending` (ADMIN) | **200**; `message: "Comprovantes listados com sucesso"`; `data` inclui o comprovante submetido, ordenado por `createdAt desc`, limitado a 200 |
| H-08 | `:id` do comprovante | `GET /transaction-history/:id` (ADMIN) | **200**; `message: "Comprovante encontrado com sucesso"`; `data` = `IManualPayment` completo |
| H-09a | Comprovante com `proof.storage: "database"` | `GET /transaction-history/:id/proof` (ADMIN) | **200**; `Content-Type` = `mime_type` do comprovante; `Content-Disposition: inline; filename*=UTF-8''<nome>`; `Cache-Control: private, no-store`; corpo = bytes do arquivo |
| H-09b | Comprovante com `proof.storage: "google_drive"` (Drive configurado) | `GET /transaction-history/:id/proof` (ADMIN) | **302** redirect para `proof.view_url` (link do Drive) |
| H-10 | Comprovante `pending`; reserva ainda `pending` | `PATCH /transaction-history/:id/review` com `{status:"approved"}` (ADMIN) | **200**; `message: "Comprovante revisado com sucesso"`; `data.status:"approved"`, `reviewed_at` setado; reserva em `agendamentos` recebe `PATCH .../payment {payment_method:"PIX"}` e depois `PATCH .../status {status:"approved"}` |
| H-11 | Outro comprovante `pending`; reserva `pending` | `PATCH /transaction-history/:id/review` com `{status:"rejected", admin_note:"comprovante ilegível"}` (ADMIN) | **200**; `data.status:"rejected"`, `data.admin_note:"comprovante ilegível"`; reserva vira `status:"rejected"` em `agendamentos` (sem chamada de `payment metadata`, pois só ocorre para `approved`) |

---

### B. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-01 | `idToken` válido | `POST /checkout` com `body: {}` | **400**; `message`: mensagem real do yup para o primeiro campo obrigatório que falhar (⚠️ **diferente de `usuarios`**, que sempre retorna `"Dados inválidos"` — aqui `handle-http-error.ts` repassa `error.message` do yup sem sobrescrever); `errors: [...]` com uma entrada por campo inválido |
| X-02 | `idToken` válido | `POST /checkout` com `scheduling_id` de 4 horários | **400**; `errors` contém `"E permitido selecionar no maximo 3 horarios"` |
| X-03 | `idToken` válido | `POST /checkout` com `equipment: {self_equipment: false}` (sem array `equipment`) | **400** (yup `.min(1).required()` na branch `self_equipment:false`) |
| X-04 | `idToken` válido; `scheduling_id` de um horário **inexistente** em `agendamentos` | `POST /checkout` | **404**; `message: "Agendamento nao encontrado"` (mapeado no `create-reserve.adapter.ts` a partir do 404 remoto); nenhuma cobrança/gateway chamado |
| X-05 | `idToken` válido; `scheduling_id` de horário **já reservado/pago** | `POST /checkout` | Depende da regra de `agendamentos` (provavelmente 409 do lado de lá, propagado como erro genérico 500 aqui, pois `create-reserve.adapter` só trata 404 especialmente) — **confirmar manualmente** o status/mensagem reais |
| X-06 | — | `GET /payment-methods` sem `Authorization` | **401**; `message: "Token nao fornecido"` |
| X-07 | — | `GET /payment-methods` com `Authorization: Bearer token-invalido` | **401**; `message: "Token invalido ou expirado"` |
| X-08 | `idToken` de um Firebase UID sem registro na coleção `usuarios` | `GET /payment-methods` | **403**; `message: "Usuario nao encontrado no sistema"` |
| X-09 | — | `POST /refunds` sem `Authorization` e sem `x-api-key` | **401**; `message: "Token nao fornecido"` (cai no `authMiddleware` padrão, já que não bate a `x-api-key`) |
| X-10 | — | `POST /refunds` com `x-api-key` **errada** | **401** (a chave errada não passa no `authOrInternalApiKey`, cai para `authMiddleware`, que exige Bearer) |
| X-11 | `idToken` de CLIENTE (não ADMIN) | `GET /transaction-history` | **403**; `message: "Acesso negado"` (`requireRole("ADMIN")`) |
| X-12 | — | `POST /webhooks/getnet` com `token` de query **errado** e sem `x-api-key` (com `GETNET_WEBHOOK_TOKEN` configurado) | **401**; `{message: "Unauthorized"}` |
| X-13 | Reserva **não** `pending` (já aprovada/rejeitada) | `POST /webhooks/getnet` com `order_id` dessa reserva, `status:"APPROVED"` | **200** (sempre) mas **nenhuma alteração** ocorre — o usecase interrompe no branch `reserve.status !== "pending"` (idempotência); confirmar via log/consulta que o status não mudou |
| X-14 | `order_id` de reserva inexistente | `POST /webhooks/getnet` com esse `order_id` | **200** (sempre); nada é alterado; usecase retorna `false` internamente (`readReservePort` devolve `null`) |
| X-15 | payload sem `order_id` | `POST /webhooks/getnet` com `{"payment_id":"x","status":"APPROVED"}` (sem `order_id`) | **200** (sempre); usecase ignora (`webhook_without_order_id`), nada é alterado |
| X-16 | `GETNET_WEBHOOK_TOKEN`/`PAGAMENTOS_WEBHOOK_TOKEN` **não configurados** | `POST /webhooks/getnet` sem `token`/`x-api-key` | ⚠️ **200 e processa normalmente** — `webhookTokenMiddleware` só bloqueia se `env.webhookToken` estiver setado; sem ele, o endpoint fica **aberto** (qualquer um pode simular um webhook). Documentar como comportamento atual, não corrigir aqui |
| X-17 | `idToken` válido; reserva de outro usuário, **sem** `public_reserve_token` | `POST /transaction-history` para essa reserva | **403**; `message: "Acesso negado para esta reserva"` |
| X-18 | Reserva já com `status: "approved"` | `POST /transaction-history` para essa reserva | **409**; `message: "A reserva nao esta pendente de pagamento"` |
| X-19 | Reserva `pending`, já com um comprovante manual submetido | `POST /transaction-history` novamente para a mesma reserva | **409**; `message: "Ja existe um comprovante para esta reserva"` |
| X-20 | Reserva `pending` de 1h (`total:80`) | `POST /transaction-history` com `amount:120` ou `duration_hours:2` ou `slots` que não batem com `scheduling_id` da reserva | **400**; `message: "Os dados do pagamento nao correspondem a reserva"` |
| X-21 | `GOOGLE_DRIVE_*` configurado mas com credenciais **inválidas** (ou pasta inexistente) | `POST /transaction-history` com proof válido | **500** (erro não tratado como `DomainError`, cai no genérico `"Erro interno no servidor"`); a reserva é automaticamente **liberada** (`updateReserveStatusPort.execute(reserve, "rejected")`, "compensação") — confirmar que a reserva não fica presa em estado intermediário |
| X-22 | Comprovante já revisado (`status: "approved"` ou `"rejected"`) | `PATCH /transaction-history/:id/review` novamente | **409**; `message: "Este comprovante ja foi revisado"` |
| X-23 | `:id` de comprovante inexistente | `PATCH /transaction-history/:id/review` | **404**; `message: "Comprovante nao encontrado"` |
| X-24 | Comprovante `pending` | `PATCH /transaction-history/:id/review` com `{status: "invalido"}` | **400**; `message: "Status de revisao invalido"` |
| X-25 | `:id` ausente/malformado nas rotas `read`/`proof`/`review` (dificilmente reproduzível via Express, mas testar `:id` vazio) | `GET /transaction-history//proof` (barra dupla) | Comportamento depende do roteamento do Express — validar que não quebra com 500 |

---

### C. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| EC-01 | `idToken` válido | `POST /checkout` com `total: 0` | **201** aceito (yup só valida `min(0)`) — confirmar se `agendamentos` aceita reserva de valor zero; documentar comportamento real |
| EC-02 | `idToken` válido | `POST /checkout` com `total: -10` | **400** (`yup.number().min(0)`) |
| EC-03 | Checkout aprovado (H-02) | `POST /refunds` com `amount` **maior** que o valor pago | **200** — **não há validação de domínio** que impeça reembolsar um valor diferente do pago; o mock aceita qualquer `amount`. Isso é uma lacuna pré-existente, não introduzida por esta task — documentar para acompanhamento |
| EC-04 | Reserva **nunca paga** (`status: "pending"`, sem `payment_id`) | `POST /refunds` com `reserve_id`/`payment_id` inventados | **200** — o usecase `create-refund` **não consulta a reserva nem verifica se ela foi paga antes de reembolsar** (chama direto `refundGatewayPort.execute`); com o gateway mockado, sempre "aprova". **Achado relevante para QA**: reembolso de transação não-paga/não-refundável não é bloqueado hoje |
| EC-05 | Checkout com `payment_method: "PIX"` vs `"CREDIT_CARD"` seguido de `POST /refunds` para cada um | Repetir H-04 trocando `payment_method` | **Nenhuma diferença observável de comportamento** entre PIX e cartão no reembolso atual — ambos passam pelo mesmo `MockRefundAdapter`, que ignora `payment_method` no resultado. Se a regra de negócio esperada é que PIX e cartão tenham fluxos de estorno diferentes (ex.: PIX é síncrono, cartão pode levar dias), **isso não está implementado**; documentar como gap conhecido, não como bug desta refatoração |
| EC-06 | Reserva `pending` | Disparar **dois** `POST /webhooks/getnet` simultâneos com o mesmo `order_id`/`status:"APPROVED"` | Idealmente só um efetiva a aprovação; como a leitura (`GET reserva`) e a escrita (`PATCH status`) não são atômicas **neste serviço**, ambas as chamadas podem ler `status:"pending"` antes que a primeira escreva — investigar se `agendamentos` tem alguma proteção do lado dele. Reportar se houver dupla contabilização no log (`payment_transaction_history`) |
| EC-07 | Getnet reenvia o **mesmo** webhook após a reserva já ter sido aprovada (replay real, não concorrência) | `POST /webhooks/getnet` repetido depois que a reserva já está `approved` | **200**, sem novo efeito colateral — idempotência garantida pelo check `reserve.status !== "pending"` (ver X-13); não há verificação de assinatura HMAC nem deduplicação por `payment_id` — a proteção contra replay é **inteiramente baseada no estado da reserva** |
| EC-08 | Comprovante manual — `duration_hours` fora de `[1,2,3]` ou `amount` fora de `[80,120,160]` | `POST /transaction-history` | **400** direto no DTO (`yup.oneOf`), antes de chegar no usecase |
| EC-09 | `PATCH /transaction-history/:id/review` para um comprovante `pending` cuja reserva **não** está mais `pending` (ex.: cancelada por outro fluxo nesse meio-tempo) e o novo `status` do review **diverge** do status atual da reserva | Revisar um comprovante nessas condições | **409**; `message: "A reserva nao esta pendente de revisao"` |
| EC-10 | Comprovante `proof.mime_type: "application/pdf"` | Ciclo completo H-06 → H-09a | Content-Type do download deve ser `application/pdf`, não forçar `image/*` |

---

### D. Conformidade arquitetural (Princípio II — específico desta task)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| CA-01 | Repo `pagamentos` | `npm run lint` (`eslint .`) | **0 erros**. `eslint.config.mjs` ativa `no-restricted-imports` para `src/domain/**` (bloqueia `infra`/`applications`) e `src/applications/**` (bloqueia `infra`) |
| CA-02 | — | `grep -rn "infra/" src/domain src/applications` (excluindo `*.spec.ts`) | **Nenhuma** ocorrência fora de testes (AC-1/AC-2) |
| CA-03 | — | Adicionar temporariamente `import x from "../../infra/schemas/user.schema"` em qualquer arquivo de `src/domain/usecases/**` e rodar `npm run lint` | ESLint **falha** com "domain nao pode importar de infra/applications (Principio II). Use um port."; reverter |
| CA-04 | — | Inspecionar as 15 classes em `src/infra/adapters/**/*.adapter.ts` | Cada uma declara `implements <PortDeOutput>` (ex.: `GetnetCheckoutAdapter implements IPaymentGatewayPort`); acessam só `schemas/`, `axios`, `fetch`/`crypto` (Drive), ou outro adapter auxiliar (`GetnetCheckoutAdapter` usa `GetnetAuthAdapter` por composição, não por herança de regra); **sem** `if` de regra de negócio (limites de valor, validação de reserva, cálculo de expectativa de valor — tudo isso está nos usecases) |
| CA-05 | — | `ls src/infra/adapters/*/` | Um arquivo `.adapter.ts` por verbo/ação; as únicas duplas são `getnet/checkout/{getnet,mock}-checkout.adapter.ts` e `getnet/refund/{getnet,mock}-refund.adapter.ts` — **duas implementações do mesmo port** (real vs. mock), escolhidas no `container.ts` por `PAYMENT_PROVIDER`, não um adapter multi-ação (AC-5 respeitado) |
| CA-06 | — | `ls src/config/` e `cat src/main.ts` | Existem `env.ts`, `container.ts`, `firebase.ts`; **não** há mais `src/applications/config/**` (movido, AC-7); `main.ts` só monta o Express (`cors`, `express.json({limit:'12mb'})`, `/api/v1` → `routes`, `mongoose.connect`) |
| CA-07 | — | `npm run test:coverage` | 156 testes passam; cobertura real **98,3% stmts / 82,2% branch / 96,3% funcs / 98,8% lines** (gate `coverageThreshold` 80% ✅); branch é a métrica mais próxima do limite — checar no relatório HTML quais branches de `submit-manual-payment`/`review-manual-payment` ficaram descobertas |
| CA-08 | — | `npm run build && node dist/main.js` (com `.env` válido) | Gera `dist/`; processo sobe, loga `"DB is connected!"` e `"Backend is running on 5001"` |
| CA-09 | — | Inspecionar `src/applications/middlewares/auth.middleware.ts` | `internalApiKeyMiddleware` está definido e testado (`auth.middleware.spec.ts`) mas **não é usado em nenhuma rota** de `routes.ts` (só `authOrInternalApiKey` é usado em `/refunds`) — código morto pré-existente, não introduzido por esta task; confirmar que não é regressão (comparar com `master`) |

---

### E. Checklist de regressão (fluxos vizinhos)

| ID | Fluxo | Verificação |
|---|---|---|
| R-01 | `beach-center-app` — **Nova reserva → Pagamento** (`CreateReserveForm` → `ReservePayment.component.tsx`) | `paymentApiService.listMethods()` popula os rádios PIX/cartão; `createCheckout` — sucesso mostra "Reserva paga" quando `payment_status === "APPROVED"` (mock) ou redireciona para `payment_url` quando `PENDING` (Getnet real) |
| R-02 | `beach-center-app` — **Envio de comprovante manual** (`CreateReserveForm` → `manualPaymentService.create`) | Upload de arquivo (PDF/JPEG/PNG) até 5MB em base64; conferir mensagem de erro amigável se o backend devolver 400/403/409 |
| R-03 | `beach-center-app` — **Admin: Comprovantes** (`TransactionHistory.page.tsx`) | Usa `manualPaymentService.{list,review,getProofBlobUrl}` — shape bate 1:1 com `IManualPayment` do backend; listar, filtrar por `status`/busca textual, aprovar/rejeitar, visualizar comprovante (blob) — todos devem funcionar sem alteração |
| R-04 | `beach-center-app` — código **não wireado** (`useTransactionHistory.hook.ts`, `transaction-history-api.service.ts`, `ITransactionHistory` domain) | Esse hook/serviço espera o shape `ITransactionHistory` (operation/product/stage/events) do log de auditoria (`payment_transaction_history`), mas **nenhuma rota expõe esse log** — `GET /transaction-history` sempre devolve `IManualPayment[]`. Confirmar (`grep -rn "useTransactionHistory" beach-center-app/src`) que esse hook não está montado em nenhuma página ativa (só `TransactionHistory.page.tsx`, que usa `manualPaymentService`, está na rota real) — se for código morto pré-existente, **não é regressão desta task**; se alguma página nova passou a usá-lo, é bug a reportar |
| R-05 | Contrato de callback com `agendamentos` | `pagamentos` chama: `POST {AGENDAMENTOS_API_URL}/reservas` (criar), `GET .../reservas/:id` (ler, `x-api-key`), `PATCH .../reservas/:id/status {status}` (Bearer do usuário **ou** `x-api-key`), `PATCH .../reservas/:id/payment {...}` (`x-api-key`), `GET .../links-reserva-publica/:token/reservas/:id/authorization` (`x-api-key`) — confirmar que `agendamentos` (já refatorado na fase 3) mantém esses mesmos paths/verbos/shapes de resposta (`{message, data}` ou `{message, data:{reserve, scheduling}}`, ambos tratados via `unwrap()`) |
| R-06 | Webhook real da Getnet | URL de notificação enviada no checkout: `{API_URL}/api/v1/webhooks/getnet?token=<GETNET_WEBHOOK_TOKEN>` (`getnet-checkout.adapter.ts#buildNotificationUrl`) — em ambiente de sandbox/homologação, `API_URL` precisa ser publicamente alcançável pela Getnet (não `localhost`); validar com um checkout real em sandbox que o webhook chega e processa |
| R-07 | Infra local (`beach-center-server`) | Container de `pagamentos` sobe com `src/main.ts` (ts-node) ou `dist/main.js`; hot-reload ao salvar `.ts` (Princípio IV) |
| R-08 | Deploy / CI | `npm ci && npm run lint && npm test` sem erro; `npm run build` gera artefato |

---

### Rastreabilidade AC → cenário

| AC | Cenários | Cobertura automatizada equivalente? |
|---|---|---|
| **AC-1** Domínio não conhece infra | CA-02, CA-03 | ✅ ESLint `no-restricted-imports` + specs de usecase |
| **AC-2** Applications não conhece infra | CA-02, CA-03 | ✅ ESLint + `*.controller.spec` (mock do container) |
| **AC-3** Usecases dependem de ports | CA-04 (inspeção) | ✅ `*.usecase.spec` (mocks `jest.Mocked<IPort>`) |
| **AC-4** Adapters implementam ports, sem regra | CA-04 | ✅ `*.adapter.spec` (15 arquivos) |
| **AC-5** Um adapter por verbo/ação | CA-05 | ➖ estrutural (inspeção) |
| **AC-6** Regra de negócio migrada (agendamentos) | — | ⏳ **Fase 3** — fora do escopo de `pagamentos` |
| **AC-7** `config/` + `main.ts` padronizados | CA-06 | ✅ `env.spec`, `firebase.spec`, `container.spec` |
| **AC-8** Nomes de arquivo (agendamentos) | — | ⏳ **Fase 3** |
| **AC-9** Contratos de endpoint inalterados | H-01..H-11, X-01..X-25, EC-* | 🟡 parcial — `*.controller.spec` afirmam status+shape com o container mockado; **comparação com baseline `master` é manual**, especialmente para: mensagem real do yup em vez de "Dados inválidos" (X-01), e o gap de validação de X-13/EC-03/EC-04 no refund |
| **AC-10** Middlewares de auth preservados | X-06..X-12, `routes.spec`, `auth.middleware.spec` | ✅ `routes.spec` + `auth.middleware.spec` (inclui `authOrInternalApiKey`, `webhookTokenMiddleware`) |
| **AC-11** ESLint configurado e limpo | CA-01 | ✅ |
| **AC-12** Cobertura ≥ 80% | CA-07 | ✅ 98,3% stmts (branch mais apertada: 82,2%) |
| **AC-13** Testes de domínio mockam portas | — | ✅ `*.usecase.spec` — todas as portas de output mockadas |
| **AC-14** Controllers e adapters cobertos | — | ✅ 5 controller specs + 15 adapter specs |
| **AC-15** `@wip` ignorado | — | ✅ `jest.config.ts` (nenhum `@wip` presente no serviço) |
| **AC-16** Ambiente local sobe com hot-reload | CA-08, R-07 | ➖ manual (`/run-server`) |

#### Cenários SEM cobertura automatizada equivalente (prioridade de QA manual)

- **AC-9 / não-regressão real:** H-01..H-11 e X-* — specs testam os controllers isolados com mock
  do container; só o teste manual contra `master` confirma shape/mensagem reais, em especial a
  mudança de comportamento do erro 400 de validação (X-01, mensagem yup real em vez de "Dados
  inválidos" fixo).
- **Lacunas de regra de negócio identificadas durante a leitura do código (não corrigir aqui, só
  confirmar/reportar):** reembolso sem checar se a reserva foi paga ou se já foi reembolsada
  (EC-03, EC-04); nenhuma diferenciação de fluxo entre PIX e cartão no reembolso (EC-05); webhook
  sem token configurado fica sem autenticação nenhuma (X-16); concorrência entre webhooks
  simultâneos não é protegida neste serviço (EC-06).
- **Integração real com a Getnet:** H-03, R-06 — exige credenciais sandbox e endpoint
  publicamente alcançável; não reproduzível em unit test.
- **Integração com o `beach-center-app`:** R-01..R-04.
- **Infra/CI:** R-05, R-07, R-08, CA-08, CA-09, AC-16.

---

## Fase 3 — agendamentos (serviço `beach-center-bff-agendamentos`)

> Roteiro de QA manual. Não substitui os testes automatizados (`/speckit-unit-tests` — 812
> testes, 99,24% cobertura de statements). Organizado por **recurso** (court, unit, scheduling,
> events_scheduled, day, reserva, public-reserve-link) dentro de cada seção, por ser uma
> superfície ~3x maior que `usuarios` (37 endpoints vs. 13).

### Escopo e pré-condições

| Item | Detalhe |
|---|---|
| Serviço | `services/beach-center-bff-agendamentos` (porta padrão **5000**, base `/api/v1`) |
| Subir | `npm run dev` (nodemon + ts-node `src/main.ts`) **ou** `npm run build && npm start` |
| Banco | MongoDB acessível via `DB` no `.env`; collections `courts`, `units`, `schedulings`, `events_scheduled`, `dias`, `reservas`, `public_reserve_links`, `users` (mirror read-only) |
| Env obrigatórias | `DB`, `FIREBASE_PROJECT_ID` (o serviço **não sobe** sem essas duas — `loadEnv()` lança `Error` síncrono) |
| Env opcionais/defaults | `PORT` (default `5000`, **não** `5001` como pagamentos), `PAGAMENTOS_API_URL` (default `http://localhost:5001/api/v1`), `PAGAMENTOS_INTERNAL_API_KEY`, `AGENDAMENTOS_INTERNAL_API_KEY`, `PUBLIC_APP_URL` (se ausente, `url` some do payload do link público — sem fallback) |
| Firebase | Só usa `getAuth().verifyIdToken` — sem `FIREBASE_CLIENT_EMAIL`/`FIREBASE_PRIVATE_KEY` (config lazy, mesma abordagem de `pagamentos`) |
| Ferramenta | Postman/Insomnia/`curl`; para fluxos autenticados é preciso um `idToken` Firebase de um usuário existente também no Mongo de `usuarios` (o serviço lê a coleção `users` espelhada, `id_firestore`) |
| Seed | 2 unidades (uma delas com `_id = 6a440a931094fad2f585011b` — **caso especial** `unitOneId`, horário 18:00–22:00 sem almoço; qualquer outra unidade usa 08:00–23:00 com almoço 10:00–15:00); pelo menos 2 quadras vinculadas a essas unidades; 1 usuário `ADMIN`, 1 usuário `CLIENTE` (mirror da collection de `usuarios`); um dia aberto (`POST /dias` ou `POST /dias/:day/schedulings`) com agendamentos gerados para "hoje" e para "amanhã"; opcionalmente um evento agendado (`eventos-agendados`) `CONFIRMED` conflitando com algum desses horários |
| Cross-serviço | `services/beach-center-bff-pagamentos` rodando e configurado com `AGENDAMENTOS_API_URL`/`AGENDAMENTOS_INTERNAL_API_KEY` apontando para este serviço — necessário para os fluxos de reembolso (`delete-reserve`) e para a checklist de regressão (seção E) |
| Baseline | Se possível, capturar as respostas dos endpoints **antes** desta refatoração (branch anterior ao commit da Fase 3) para comparar shape/status/mensagem |

#### Endpoints sob teste (37)

##### `court` (`/api/v1/quadras`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E1 | GET | `/quadras/:id` | pública |
| E2 | POST | `/quadras` | Bearer + ADMIN |
| E3 | PATCH | `/quadras/:id` | Bearer + ADMIN |
| E4 | PATCH | `/quadras/:id/delete` | Bearer + ADMIN |

##### `unit` (`/api/v1/unidades`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E5 | GET | `/unidades/:id` | pública |
| E6 | GET | `/unidades` | pública |
| E7 | POST | `/unidades` | Bearer + ADMIN |
| E8 | PATCH | `/unidades/:id` | Bearer + ADMIN |
| E9 | PATCH | `/unidades/:id/delete` | Bearer + ADMIN |

##### `scheduling` (`/api/v1/agendamentos`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E10 | POST | `/agendamentos` | pública |
| E11 | GET | `/agendamentos/:id` | Bearer + ADMIN |
| E12 | GET | `/agendamentos` | Bearer + ADMIN |
| E13 | PATCH | `/agendamentos/:id` | Bearer + ADMIN |
| E14 | PATCH | `/agendamentos/:id/delete` | Bearer + ADMIN |

##### `events_scheduled` (`/api/v1/eventos-agendados`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E15 | POST | `/eventos-agendados` | Bearer + ADMIN |
| E16 | GET | `/eventos-agendados/:id` | Bearer + ADMIN |
| E17 | GET | `/eventos-agendados` | Bearer + ADMIN |
| E18 | PATCH | `/eventos-agendados/:id` | Bearer + ADMIN |
| E19 | PATCH | `/eventos-agendados/:id/delete` | Bearer + ADMIN |

##### `day` (`/api/v1/dias`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E20 | POST | `/dias` | Bearer + ADMIN |
| E21 | PATCH | `/dias/close-date` | Bearer + ADMIN |
| E22 | PATCH | `/dias/open-date` | Bearer + ADMIN |
| E23 | GET | `/dias` | pública |
| E24 | POST | `/dias/:day/schedulings` | pública |

##### `reserva` (`/api/v1/reservas`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E25 | POST | `/reservas` | Bearer |
| E26 | GET | `/reservas/protocol/:number` | pública |
| E27 | PATCH | `/reservas/protocol/:number/cancel` | pública |
| E28 | PATCH | `/reservas/:id/payment` | internal API key (`x-api-key`) — chamada por `pagamentos` |
| E29 | PATCH | `/reservas/:id/delete` | Bearer + ADMIN |
| E30 | GET | `/reservas/:id` | Bearer + ADMIN **ou** internal API key |
| E31 | GET | `/reservas` | Bearer + ADMIN |
| E32 | PATCH | `/reservas/:id` | Bearer + ADMIN |
| E33 | PATCH | `/reservas/:id/status` | Bearer + ADMIN **ou** internal API key — chamada por `pagamentos` |

##### `public-reserve-link` (`/api/v1/links-reserva-publica`)

| # | Método | Rota | Auth |
|---|---|---|---|
| E34 | POST | `/links-reserva-publica` | Bearer + ADMIN |
| E35 | GET | `/links-reserva-publica/:token` | pública |
| E36 | POST | `/links-reserva-publica/:token/reservas` | pública |
| E37 | GET | `/links-reserva-publica/:token/reservas/:reserveId/authorization` | internal API key (`x-api-key`) — chamada por `pagamentos` |

---

### A. Caminhos felizes (não-regressão de contrato)

#### `court`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-01 | — | `POST /quadras` (ADMIN) com `{name: "Quadra QA 1"}` | **201**; `message: "Court created successfully"` (mensagem em inglês, inconsistência pré-existente — AC-9 proíbe traduzir); `data` = `ICourt` |
| H-02 | Quadra existente | `GET /quadras/:id` (sem auth) | **200**; `message: "Quadra encontrada com sucesso"`; `data` = `ICourt` |
| H-03 | Quadra existente | `PATCH /quadras/:id` (ADMIN) com `{name: "Quadra QA renomeada"}` | **200**; `message: "Quadra encontrada com sucesso"` (mensagem pré-existente, reaproveitada do read); `data` = `ICourt` atualizado; `list` = array de todas as quadras não deletadas |
| H-04 | Quadra descartável, sem agendamentos vinculados | `PATCH /quadras/:id/delete` (ADMIN) | **200**; `message: "Quadra deletada com sucesso"`; `data` = quadra (soft-deleted via `mongoose-delete`) |

#### `unit`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-05 | Quadra(s) existente(s) | `POST /unidades` (ADMIN) com `{name, location: {street, quarter, number, city, state, cep}, courts: [courtId]}` | **201**; `message: "Unit created successfully"` (idem H-01, inglês); `data` = `IUnit` |
| H-06 | Unidade existente | `GET /unidades/:id` (sem auth) | **200**; `message: "Unidade encontrada com sucesso"` |
| H-07 | Várias unidades | `GET /unidades` (sem auth, sem query) | **200**; `message: "Unidades listadas com sucesso"`; `data` = todas as não deletadas |
| H-08 | Unidade "Praia Náutica" existe | `GET /unidades?search=nautica` | **200**; encontra por busca fuzzy (normaliza acento/case, ignora pontuação) sobre `name` + campos de `location` |
| H-09 | Unidade existente | `PATCH /unidades/:id` (ADMIN) | **200**; `message: "Unidade encontrada com sucesso"`; `data` + `list` |
| H-10 | Unidade descartável | `PATCH /unidades/:id/delete` (ADMIN) | **200**; `message: "Unidade deletada com sucesso"` |

#### `scheduling`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-11 | Quadra pertence à unidade; horário dentro do funcionamento; data futura | `POST /agendamentos` (pública) com `{date, start_time: "09:00", end_time: "10:00", court, unit}` | **201**; `message: "Agendamento criado com sucesso"`; `data.available` = `false` se coincide com evento CONFIRMED ou é passado, senão o valor enviado (default `false`) |
| H-12 | Agendamento existente | `GET /agendamentos/:id` (ADMIN) | **200**; `message: "Agendamento encontrado com sucesso"` |
| H-13 | — | `GET /agendamentos?unit=&court=&date=&available=true` (ADMIN) | **200**; `message: "Agendamentos listados com sucesso"`; filtra pelos 4 parâmetros |
| H-14 | Agendamento existente | `PATCH /agendamentos/:id` (ADMIN) | **200**; `message: "Agendamento encontrado com sucesso"`; `data` + `list` |
| H-15 | Agendamento sem reserva vinculada | `PATCH /agendamentos/:id/delete` (ADMIN) | **200**; `message: "Agendamento deletado com sucesso"`; removido também de `dias.scheduling_ids` (via `IRemoveSchedulingFromDaysPort`) |

#### `events_scheduled` — fluxo composto crítico (bloqueio de agendamentos)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-16 | Quadra+unidade válidas; nenhum evento CONFIRMED conflitante | `POST /eventos-agendados` (ADMIN) com `{event_type: "MENSALISTA", day_of_week: "segunda-feira", start_time: "18:00", end_time: "20:00", court, unit, status: "CONFIRMED"}` | **201**; `message: "Evento agendado criado com sucesso"`; horários arredondados (`formatRoundedTime`) — `end_time` com minutos > 0 arredonda pra cima; **todos os agendamentos existentes** de segunda-feira, 18:00–20:00, nessa quadra/unidade, a partir de hoje, viram `available: false` |
| H-17 | Evento CONFIRMED do H-16 criado; agendamento de segunda 18:00–19:00 existe e está `available: false` por causa dele | `PATCH /eventos-agendados/:id/delete` (ADMIN) | **200**; `message: "Evento agendado deletado com sucesso"`; `data` = **lista completa** de eventos (não apenas o deletado — `list` port retorna todos, incluindo o agora `CANCELLED`); o agendamento de segunda 18:00–19:00 volta a `available: true` (contanto que não tenha reserva ativa nem outro evento CONFIRMED conflitante) |
| H-18 | Evento existente | `GET /eventos-agendados/:id` (ADMIN) | **200**; `message: "Evento agendado encontrado com sucesso"` |
| H-19 | — | `GET /eventos-agendados?event_type=&day_of_week=&status=` (ADMIN) | **200**; `message: "Eventos agendados listados com sucesso"` |
| H-20 | Evento existente | `PATCH /eventos-agendados/:id` (ADMIN) mudando horário | **200**; `message: "Evento agendado atualizado com sucesso"`; `data.updated` + `data.list`; libera os agendamentos do horário antigo (`releaseEventFromSchedulings`) e bloqueia os do horário novo (`applyEventToSchedulings`); checagem de conflito usa `excludeEventId` = o próprio evento (não conflita consigo mesmo) |

#### `day` — orquestração completa

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-21 | Quadra+unidade válidas; dia futuro; `begin_hour`/`end_hour`/almoço batendo com a configuração da unidade | `POST /dias` (ADMIN) com array `[{day, begin_hour, start_lunch_time?, end_lunch_time?, end_hour, court, unit}]` | **201**; `message: "Dias criados com sucesso"`; `data` = array de `{id, day, opened, scheduling_ids, schedulings}`; gera slots de 60min entre `begin_hour` e `end_hour`, pulando o intervalo de almoço quando configurado, marcando `available:false` nos que colidem com evento CONFIRMED |
| H-22 | — | `POST /dias/:day/schedulings` (pública) com `{court, unit}` no body | **200**; `message: "Agendamentos do dia listados com sucesso"`; `data` = agendamentos do dia (cria o dia/slots faltantes on-the-fly se ainda não existirem — mesma lógica de `POST /dias`); `exception_conflicts` = lista de conflitos evento×agendamento **disponível** |
| H-23 | Dia aberto sem escopo | `GET /dias` (pública) | **200**; `message: "Dias listados com sucesso"`; marca agendamentos passados como indisponíveis antes de listar |
| H-24 | Dia com agendamentos e **sem** reservas ativas | `PATCH /dias/close-date` (ADMIN) com `{day}` (sem `court`/`unit` — fecha o dia inteiro) | **200**; `message: "Dia fechado com sucesso"`; `data.opened: false`; todos os agendamentos do dia viram `available:false` |
| H-25 | Dia fechado previamente pelo H-24 | `PATCH /dias/open-date` (ADMIN) com `{day}` | **200**; `message: "Dia aberto com sucesso"`; `data.opened: true`; agendamentos elegíveis voltam a `available:true` (`releaseSchedulingsIfAllowedStrict`) |
| H-26 | Dia com agendamentos vinculados a reservas `pending`/`approved`; `cancel_reserves: true` | `PATCH /dias/close-date` com `{day, cancel_reserves: true}` | **200**; `message: "Dia fechado com sucesso"`; **todas** as reservas ativas dos agendamentos afetados são canceladas (`DeleteReserveUsecase.execute(id, {skipCancellationTimeValidation: true})` — a janela de 2h **não** se aplica aqui) e reembolsadas quando aplicável; dia fecha normalmente |
| H-27 | `court`+`unit` informados (fechamento escopado) | `PATCH /dias/close-date` com `{day, court, unit}` | **200**; só os agendamentos daquela quadra/unidade entram em `closed_scheduling_ids`; `data.opened` **não muda** (fica como estava — só fecha tudo quando nem `court` nem `unit` são informados) |

#### `reserva` — fluxo composto crítico (limite de 3 horários, disponibilidade)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-28 | `idToken` de CLIENTE ou ADMIN; 1–3 `scheduling_id` disponíveis, futuros, sem conflito de evento | `POST /reservas` com `{name, email, phone, scheduling_id: [id1, id2], price, total, equipment: {self_equipment: true}, discount?}` | **201**; `message: "Reserva criada com sucesso"`; `data.status: "pending"`; `data.number` = protocolo gerado via `nanoid` (10 dígitos numéricos) se não enviado; os agendamentos usados ficam bloqueados (indisponíveis para outra reserva) |
| H-29 | Reserva `approved` existente, primeiro agendamento com `start_time` > agora + 2h | `PATCH /reservas/:id/delete` (ADMIN) | **200**; `message: "Reserva deletada com sucesso"`; `data` = lista de reservas atualizadas (status `cancelled`); se `payment_id` existe, estorno é solicitado via `IRefundPaymentPort` (chamada a `pagamentos`); agendamentos liberados se elegíveis (`releaseSchedulingsIfAllowed`) |
| H-30 | Reserva `pending` (sem pagamento) | `PATCH /reservas/protocol/:number/cancel` (pública, sem auth) | **200**; `message: "Reserva cancelada com sucesso"`; mesma lógica de `delete`, mas localizando por protocolo; se já `cancelled`, ver X-30 |
| H-31 | Protocolo válido | `GET /reservas/protocol/:number` (pública) | **200**; `message: "Reserva encontrada com sucesso"`; `data.can_cancel` = `true` se dentro da janela de 2h antes do agendamento mais cedo; `data.cancellation_deadline` presente |
| H-32 | ADMIN ou `x-api-key` válido | `GET /reservas` (ADMIN) com `?status=&name=&email=&phone=&scheduling_id=` | **200**; `message: "Reservas listadas com sucesso"` |
| H-33 | `x-api-key` = `AGENDAMENTOS_INTERNAL_API_KEY` | `GET /reservas/:id` **sem** Bearer, só header `x-api-key` | **200**; `message: "Reserva encontrada com sucesso"` — rota liberada tanto para ADMIN quanto para a chave interna (`adminOrInternalApiKey`) |
| H-34 | Reserva existente, ADMIN | `PATCH /reservas/:id` alterando `scheduling_id` para incluir um novo horário disponível | **200**; `message: "Reserva encontrada com sucesso"`; só valida disponibilidade dos horários **adicionados** (`getAddedSchedulingIds`), não dos que já estavam na reserva; libera os removidos, bloqueia os atuais |
| H-35 | `pagamentos` chamando com `x-api-key` válido | `PATCH /reservas/:id/payment` com `{payment_id, checkout_id, payment_method: "PIX"}` | **200**; `message: "Metadados de pagamento atualizados com sucesso"` |
| H-36 | `pagamentos` (webhook Getnet) chamando com `x-api-key` válido | `PATCH /reservas/:id/status` com `{status: "approved"}` | **200**; `message: "Status atualizado com sucesso"`; `data.reserve` + `data.scheduling`; ao aprovar, os agendamentos ficam bloqueados (`blockSchedulings`); ao rejeitar/cancelar, liberados |

#### `public-reserve-link` — fluxo completo criar→validar→reservar

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| H-37 | ADMIN; `PUBLIC_APP_URL` configurada | `POST /links-reserva-publica` (ADMIN) com `{expires_at?: futuro}` | **201**; `message: "Link publico criado com sucesso"`; `data.token`, `data.status: "active"`, `data.url = "${PUBLIC_APP_URL}/agendar/${token}"` (omitido se `PUBLIC_APP_URL` não configurada) |
| H-38 | Link `active`, não expirado, não usado | `GET /links-reserva-publica/:token` (pública) | **200**; `message: "Link valido"`; `data = {token, expires_at}` |
| H-39 | Link `active`; scheduling_id disponíveis | `POST /links-reserva-publica/:token/reservas` (pública) com mesmo payload de H-28 | **201**; `message: "Reserva criada com sucesso"`; internamente: `claimLinkPort` (muda status pra `processing`) → `createReserveUseCase.execute` → sucesso → `markUsedLinkPort` (status `used`, `reserve_id` setado); se a criação da reserva falhar/retornar `null`, o link é liberado de volta (`releaseLinkPort`) |
| H-40 | Reserva criada via H-39; `x-api-key` válido | `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` | **200**; `message: "Link autorizado"`; `data.authorized: true` |

---

### B. Fluxos de exceção

#### `court` / `unit`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-01 | — | `POST /quadras` com `{}` | **400**; `message: "Dados inválidos"`; `errors: [{field: "name", message: ...}]` |
| X-02 | ADMIN | `GET /quadras/<24hex inexistente>` | **200** com `data: null` (⚠️ **read não retorna 404** — só update/delete verificam `!result`; documentar como comportamento pré-existente, não regressão) |
| X-03 | ADMIN | `PATCH /quadras/<24hex inexistente>` | **404**; `message: "Quadra não encontrada"` |
| X-04 | — | Sem token | `POST /quadras` | **401**; `message: "Token nao fornecido"` |
| X-05 | `idToken` de CLIENTE | `POST /quadras` | **403**; `message: "Acesso negado"` |
| X-06 | ADMIN; `courts: ["id-invalido"]` | `POST /unidades` | **400**; `message: "Dados inválidos"`; erro no campo `courts[0]` ("court_id deve ser um ObjectId válido de 24 caracteres") |
| X-07 | ADMIN | `PATCH /unidades/<24hex inexistente>` | **404**; `message: "Unidade não encontrada"` |

#### `scheduling`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-08 | `court`/`unit` inexistentes | `POST /agendamentos` | **404**; `message: "Quadra nao encontrada"` **ou** `"Unidade nao encontrada"` (o que faltar primeiro no `Promise.all`) |
| X-09 | Quadra existe, mas não pertence à unidade informada | `POST /agendamentos` com `court`/`unit` de unidades diferentes | **400**; `message: "Quadra nao pertence a unidade informada"` |
| X-10 | `start_time`/`end_time` fora de 08:00–23:00 (unidade padrão) | `POST /agendamentos` com `start_time: "07:00"` | **400**; `message: "Horario fora do funcionamento da unidade. Use 08:00 ate 23:00"` |
| X-11 | Unidade padrão; horário cruzando o almoço (10:00–15:00) | `POST /agendamentos` com `{start_time: "09:30", end_time: "10:30"}` | **400**; `message: "Horario indisponivel no almoco da unidade. Use antes de 10:00 ou depois de 15:00"` |
| X-12 | `end_time` <= `start_time` | `POST /agendamentos` com `{start_time: "10:00", end_time: "09:00"}` | **400** já no DTO (yup `timeRangeValidation`): `message: "Dados inválidos"`, erro em `end_time` "start_time deve ser anterior ao end_time" |
| X-13 | Data já passada | `POST /agendamentos` com `date` de ontem | **400**; `message: "Nao e permitido criar agendamentos para datas que ja passaram"` |
| X-14 | ADMIN; agendamento com reserva vinculada | `PATCH /agendamentos/:id/delete` | **409**; `message: "Nao e possivel deletar um agendamento que possui reserva vinculada"` |
| X-15 | `id` malformado (não 24-hex) | `PATCH /agendamentos/abc/delete` | **400**; `message: "ID inválido"` |
| X-16 | ADMIN; `?available=talvez` | `GET /agendamentos?available=talvez` | **400**; `message: "available deve ser true ou false"` (validação feita direto no controller, não no DTO) |

#### `events_scheduled` — conflito com agendamento já bloqueado

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-17 | Evento CONFIRMED já existe segunda 18:00–20:00 na mesma quadra/unidade | `POST /eventos-agendados` com outro evento CONFIRMED segunda 19:00–21:00 (mesma quadra/unidade) | **400** (não 409!); `message: "Conflito de excecao existente na mesma quadra, unidade e horario."` — `EventConflictError` é mapeado para statusCode **400**, preservando o comportamento pré-refatoração mesmo parecendo um 409 semanticamente |
| X-18 | Evento CONFIRMED existente; atualização não muda horário | `PATCH /eventos-agendados/:id` reenviando os mesmos dados | **200** — não conflita consigo mesmo (`excludeEventId`) |
| X-19 | `event_type` fora do enum | `POST /eventos-agendados` com `event_type: "FUTEBOL"` | **400**; `message: "Dados inválidos"` (yup `oneOf`) |
| X-20 | ADMIN; evento inexistente | `PATCH /eventos-agendados/<24hex inexistente>` | **404**; `message: "Evento agendado nao encontrado"` (controller checa `!result.list.length \|\| !result.updated`) |
| X-21 | Sem `x-api-key` nem Bearer | `POST /eventos-agendados` | **401**; `message: "Token nao fornecido"` |

#### `day` — o caso mais crítico: conflito de reservas ao fechar

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-22 | Dia com agendamentos vinculados a reservas `pending`/`approved`; **sem** `cancel_reserves` | `PATCH /dias/close-date` com `{day}` (sem `cancel_reserves` ou `cancel_reserves: false`) | **409**; corpo **não** segue o formato padrão de erro — shape especial: `{message: "Existem reservas vinculadas aos horarios deste dia. Confirme para cancelar e estornar antes de fechar.", code: "CLOSE_DATE_RESERVE_CONFLICT", reserves: [{id, number, name, email, phone, scheduling_id: string[], total, status, payment_id?, refund_status?}]}` — verificar que `handleHttpError` **não** é usado aqui, o controller intercepta `CloseDateReserveConflictError` antes |
| X-23 | Apenas um de `court`/`unit` informado | `PATCH /dias/close-date` com `{day, court}` (sem `unit`) | **400**; `message: "Informe unidade e quadra para fechar um recorte especifico"` |
| X-24 | Dia sem nenhum agendamento gerável (quadra/unidade sem agendamentos e fora da janela de funcionamento) | `PATCH /dias/close-date` com `{day, court, unit}` de um recorte vazio | **404**; `message: "Nenhum agendamento encontrado para fechar"` |
| X-25 | Dia inexistente, sem `court`/`unit` | `PATCH /dias/close-date` com `{day}` de um dia nunca criado | **404**; `message: "Dia nao encontrado"` |
| X-26 | `day` malformado | `POST /dias` com `[{day: "01-01-2026", ...}]` | **400**; `message: "Dados inválidos"` (yup regex `YYYY-MM-DD`) |
| X-27 | `begin_hour`/`end_hour` divergentes da configuração real da unidade | `POST /dias` com `begin_hour: "07:00"` para unidade padrão (que é `08:00`) | **400**; `message: "Horario de funcionamento invalido para a unidade. Use 08:00 ate 23:00"` |
| X-28 | Unidade sem almoço configurado (ex.: `unitOneId`) mas payload envia `start_lunch_time` | `POST /dias` com `start_lunch_time` para a unidade `6a440a931094fad2f585011b` | **400**; `message: "Unidade nao possui horario de almoco configurado"` |
| X-29 | ADMIN; sem token | `POST /dias` | **401**; `message: "Token nao fornecido"` |

#### `reserva`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-30 | 4 `scheduling_id` no payload | `POST /reservas` | **400** (validado já no DTO yup `.max(3, ...)` **antes** de chegar ao usecase); `message: "Dados inválidos"`; erro em `scheduling_id` "E permitido selecionar no maximo 3 horarios" |
| X-31 | `scheduling_id` repetido `[a, a]` | `POST /reservas` | **400**; DTO `.test('unique-scheduling-id', ...)`: "Nao e permitido repetir horarios na mesma reserva" |
| X-32 | 1+ `scheduling_id` inexistente(s) | `POST /reservas` | **404**; `message: "Agendamento não encontrado"`; `id: [scheduling_id enviados]` (shape especial do controller, não `handleHttpError`) |
| X-33 | `scheduling_id` de agendamento de data passada | `POST /reservas` | **400**; `message: "Nao e permitido criar reserva para datas que ja passaram"` |
| X-34 | `scheduling_id` de agendamento com evento CONFIRMED conflitante (bloqueado por H-16) | `POST /reservas` | **409**; `message: "Um ou mais horarios selecionados possuem excecao de agendamento"` — este é o cenário-chave "reservar horário já bloqueado por evento" |
| X-35 | `scheduling_id` com `available: false` (já reservado por outra pessoa) | `POST /reservas` | **409**; `message: "Um ou mais horarios selecionados nao estao disponiveis"` |
| X-36 | Reserva com primeiro agendamento a menos de 2h de distância | `PATCH /reservas/:id/delete` (ADMIN) | **400**; `message: "Reserva so pode ser cancelada ate 2 horas antes do primeiro agendamento"` |
| X-37 | Reserva já `cancelled` | `PATCH /reservas/protocol/:number/cancel` novamente | **409**; `message: "Reserva ja foi cancelada"` |
| X-38 | Reserva `approved`, `total > 0`, sem `payment_id`, `payment_method` != `"PIX"` | `PATCH /reservas/:id/delete` | **409**; `message: "Nao foi possivel estornar: reserva aprovada sem identificador de pagamento"` |
| X-39 | Reserva `approved` com `payment_id`; gateway de pagamento retorna falha no estorno | `PATCH /reservas/:id/delete` | **502**; `message: "Nao foi possivel estornar o pagamento da reserva"` |
| X-40 | `id` malformado | `PATCH /reservas/xyz/delete` | **400**; `message: "ID inválido"` |
| X-41 | `PATCH /reservas/:id/payment` sem header `x-api-key` (ou errado) | — | **401**; `message: "Unauthorized"` (⚠️ nota: mensagem em **inglês**, diferente do padrão PT-BR do resto do serviço — `internalApiKeyMiddleware` tem resposta própria, não passa por `handleHttpError`) |
| X-42 | `GET /reservas/:id` sem Bearer nem `x-api-key` | — | **401**; `message: "Token nao fornecido"` (cai no fluxo `authMiddleware` de `adminOrInternalApiKey`) |
| X-43 | `idToken` de CLIENTE tentando `GET /reservas` (lista, ADMIN-only) | — | **403**; `message: "Acesso negado"` |
| X-44 | ADMIN; `status: "invalido"` em `PATCH /reservas/:id/status` | — | **400**; `message` = mensagem yup padrão (`error.message`, não `"Dados inválidos"` — este controller usa um catch de yup ligeiramente diferente dos demais, ver nota na seção D) |

#### `public-reserve-link`

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| X-45 | Token inexistente/expirado/já usado | `GET /links-reserva-publica/:token` | **404**; `message: "Link invalido, expirado ou ja utilizado"` |
| X-46 | Token já `used` (ou `processing` de outra requisição concorrente) | `POST /links-reserva-publica/:token/reservas` | **409**; `message: "Link invalido, expirado ou ja utilizado"` (o `claimLinkPort` falha, lança antes mesmo do try/catch de `handleUsecaseError`) |
| X-47 | Token válido, mas `scheduling_id` já indisponíveis | `POST /links-reserva-publica/:token/reservas` | **404** `"Agendamento nao encontrado"` **e** o link é liberado de volta para `active` (`releaseLinkPort`) — confirmar que o token pode ser reutilizado depois |
| X-48 | CLIENTE tentando criar link | `POST /links-reserva-publica` | **403**; `message: "Acesso negado"` |
| X-49 | `authorizePublicReserve` sem `x-api-key` | `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` | **401**; `message: "Unauthorized"` |
| X-50 | Reserva não vinculada a esse token | `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` (com `x-api-key` correta) | **200** (não 403!) com `data.authorized: false`, `message: "Link nao autorizado"` — a rota sempre responde 200/403 conforme o boolean, nunca lança erro de domínio para esse caso |

---

### C. Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| EC-01 | Unidade `unitOneId` (`6a440a931094fad2f585011b`) | `POST /agendamentos` com `start_time: "18:00"`, `end_time: "22:00"` nessa unidade | Permitido — janela é 18:00–22:00, **sem** almoço (`resolveOperatingHours` retorna o caso especial hardcoded, não o default 08:00–23:00) |
| EC-02 | Unidade `unitOneId` | `POST /agendamentos` com `start_time: "17:00"` | **400** "Horario fora do funcionamento da unidade. Use 18:00 ate 22:00" — confirma que o caso especial é aplicado e não o default |
| EC-03 | Qualquer unidade | `POST /agendamentos` com `start_time: "10:00"`, `end_time: "10:00"` (mesmo horário) | **400** "Horario final deve ser maior que o horario inicial" (viola tanto o DTO yup `is-greater` quanto a validação de usecase — confirmar que o DTO barra primeiro) |
| EC-04 | Agendamento de "hoje" já passou (horário atual > `end_time`) | `POST /agendamentos` com `date` = hoje, `start_time` no passado | `available` força `false` mesmo se `available: true` foi enviado (`hasException \|\| isPastScheduling(date) ? false : data.available`) — **note**: `isPastScheduling` compara só a **data** (00:00 local), não a hora exata, então um agendamento de hoje às 08:00 com "agora" sendo 20:00 ainda é considerado "não passado" pela regra de criação (ela olha o dia, não o horário) |
| EC-05 | `day_of_week` enviado acentuado (`"Terça-Feira"`) vs. sem acento (`"terca-feira"`) vs. maiúsculo | `POST /eventos-agendados` com cada variação | Todos normalizados igual via `normalizeDayOfWeek` (NFD + remove diacríticos + lowercase + trim) — o evento casa com agendamentos independente da forma exata digitada |
| EC-06 | Evento `end_time: "20:30"` | `POST /eventos-agendados` | `formatRoundedTime` arredonda **para cima** quando minutos > 0 no `end_time`: vira `"21:00"` na resposta — comportamento pré-existente, não um bug |
| EC-07 | Evento `start_time: "18:30"` | `POST /eventos-agendados` | `formatRoundedTime(start)` **não** arredonda o início (só o fim recebe tratamento especial) — vira `"18:00"` (trunca, não arredonda) — checar a mensagem exata gerada e comparar com o comportamento documentado em `shared/date-time.ts` |
| EC-08 | Reserva cujo agendamento mais cedo começa **exatamente** 2h a partir de agora | `PATCH /reservas/:id/delete` no instante exato do limite | `isWithinCancellationWindow` usa `<=` — no limite exato ainda é permitido (`new Date().getTime() <= deadline.getTime()`); 1 segundo depois, bloqueado |
| EC-09 | Reserva com 3 `scheduling_id`, todos no limite | `POST /reservas` | Aceito (limite é `> 3`, não `>= 3`) |
| EC-10 | Agendamento `available: false` por já ter uma reserva `approved` vinculada (zero capacidade restante) | Tentar `POST /reservas` reaproveitando o mesmo `scheduling_id` em outra reserva | **409** "Um ou mais horarios selecionados nao estao disponiveis" — o campo `available` na collection é o único controle de capacidade (não há contagem de vagas, é binário) |
| EC-11 | Data/hora do servidor próxima da virada de meia-noite em `America/Sao_Paulo` (ex.: 23:55 local = 02:55 UTC do dia seguinte) | Criar agendamento para "hoje" e verificar `getLocalDateKey`/`getTodayStart` | A comparação de "hoje" deve usar o fuso `America/Sao_Paulo` (via `Intl.DateTimeFormat`), não UTC — testar logo antes/depois da meia-noite local para confirmar que a data não "pula" um dia incorretamente |
| EC-12 | `GET /unidades?search=` (vazio) e `?search=%20` (só espaço) | — | Retorna **todas** as unidades (filtro ignorado quando vazio/trim vazio), igual ao padrão já usado em `usuarios` |
| EC-13 | `PATCH /reservas/:id` (update parcial) enviando **apenas** `{name: "Novo Nome"}`, **sem** o campo `equipment` | — | ⚠️ **Bug pré-existente, preservado deliberadamente (AC-9)**: `equipment.self_equipment` é `yup.boolean().required()` **dentro** do shape, mesmo o objeto `equipment` inteiro sendo `.optional()` — o yup ainda valida o shape interno quando `equipment` está ausente, e o PATCH falha com **400** "Dados inválidos" em vez de aplicar o merge parcial esperado. Documentado em `plan.md` (achado do `/speckit-unit-tests`) — **não corrigir**, apenas confirmar que o comportamento é idêntico ao pré-refatoração |
| EC-14 | `close-date` com `cancel_reserves: true` mas **nenhuma** reserva ativa vinculada | `PATCH /dias/close-date` | **200** normal — o laço de cancelamento simplesmente não executa (0 iterações), sem erro |
| EC-15 | `list-day-schedulings` (`POST /dias/:day/schedulings`) chamado duas vezes seguidas no mesmo dia | — | Idempotente — a segunda chamada não duplica agendamentos (`filterMissingSchedulings` usa chave `start_time+court+unit`) nem duplica `scheduling_ids` no dia |
| EC-16 | Link público com `expires_at` no passado | `POST /links-reserva-publica` com `expires_at` = ontem | **400** já no DTO yup (`expires_at deve ser uma data futura`) — nunca chega a criar o link |
| EC-17 | `PATCH /agendamentos/:id` mudando `court`/`unit` para uma combinação onde a quadra não pertence à unidade | — | **400** "Quadra nao pertence a unidade informada" — mesma validação de create, reaplicada no update |

---

### D. Conformidade arquitetural (Princípio II — específico desta task)

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| CA-01 | Repo `agendamentos` | `npm run lint` | **0 erros**. A regra `no-restricted-imports` está ativa bloqueando `domain/**`→`infra/**` e `applications/**`→`infra/**` |
| CA-02 | — | `grep -rn "infra/" src/domain src/applications` (excluindo `.spec.ts`) | **Nenhuma** ocorrência (AC-1/AC-2) |
| CA-03 | — | Adicionar temporariamente `import x from "../../infra/schemas/court.schema"` em qualquer arquivo de `src/domain/` e rodar `npm run lint` | ESLint **falha**; reverter |
| CA-04 | — | Inspecionar `src/infra/adapters/**/*.adapter.ts` (60 arquivos) | Cada um `implements` um port de `domain/ports/output`; só acessa `schemas`/`axios`; **sem** `if` de regra de negócio — em particular, `event-conflict`, `apply/release-event-to-schedulings`, `scheduling-availability` e `scheduling-references` **não devem mais existir** em `infra/adapters/**/shared` ou `infra/adapters/**/validators` (migrados para `domain/usecases/shared/*.service.ts`, AC-6) |
| CA-05 | — | `ls src/infra/adapters/reserva/` e `src/infra/adapters/public-reserve-link/` | Um arquivo `.adapter.ts` por verbo/ação (AC-5) — confirmar especificamente que `DeleteReserveAdapter` (antes 4 métodos), `UpdateReserveAdapter` (antes 3) e `PublicReserveLinkAdapter` (antes 6) foram splitados em adapters de 1 método cada; nenhum adapter multi-ação restante em nenhum recurso |
| CA-06 | — | `find src/infra/adapters -name "*.adapter.ts" \| wc -l` | 60 adapters (cresceu de 54 do plano original por causa dos splits do AC-5) |
| CA-07 | — | `ls src/applications/routes/` | Todos os arquivos seguem `<recurso>.route.ts` — confirmar especificamente **ausência** de `court.rotes.ts` e `unit.routes.ts` (nomes antigos com typo, AC-8); confirmar `court.route.ts` e `unit.route.ts` presentes |
| CA-08 | — | `grep -rn "\.ts$" src/applications/controllers --include="*.controller.ts" -l \| wc -l` vs. inspeção manual | Todos os controllers seguem `<ação>.controller.ts`; confirmar especificamente que `applications/controllers/reserva/update/update-reserve.ts` foi renomeado para `update-reserve.controller.ts` e `applications/controllers/shared/update-status.ts` para `update-reserve-and-scheduling-status.controller.ts` (AC-8) |
| CA-09 | — | `ls src/config/` e `cat src/main.ts` | Existem `env.ts`, `container.ts`, `firebase.ts` e `main.ts`; **não** há `src/applications/config/**` nem `src/infra/firebase/**` (AC-7) |
| CA-10 | — | `npm run test:coverage` | Todos os **812 testes** passam; cobertura global **99,24% stmts / 96,71% branch / 99,46% funcs / 99,39% lines** — acima do gate de 80% (AC-12), o maior dos 3 serviços |
| CA-11 | — | Inspecionar `*.usecase.spec.ts` de qualquer recurso (ex.: `close-date.usecase.spec.ts`) | Todas as portas de output injetadas são `jest.Mocked<T>` — sem `mongoose`/`axios`/`firebase-admin` reais (AC-13) |
| CA-12 | — | Verificar cobertura de `src/applications/controllers/**` e `src/infra/adapters/**` no relatório | Specs cobrindo sucesso + 4xx/5xx para os 35 controllers reais e para os 60 adapters (AC-14) |
| CA-13 | — | `grep -rn "@wip" src` | Nenhuma ocorrência — suíte inteira "real", sem skips (AC-15, trivialmente satisfeito por não haver nenhum arquivo marcado) |
| CA-14 | — | `npm run build` | Gera `dist/`; `node dist/main.js` sobe o servidor (requer `DB`/`FIREBASE_PROJECT_ID` válidos) |
| CA-15 | — | Inspecionar `src/domain/usecases/shared/event-conflict.service.ts`, `scheduling-availability.service.ts`, `event-scheduling-impact.service.ts`, `scheduling-references.validator.ts`, `reserve-scheduling.validator.ts`, `reserve-cancellation-window.ts` | Toda a lógica de decisão de negócio de agendamentos (conflito de evento, limite de 3 horários, disponibilidade, referências quadra/unidade, janela de cancelamento de 2h) vive em `domain/usecases/**`, não em `infra/adapters/**` (AC-6, específico de agendamentos) |
| CA-16 | — | Inspecionar `applications/controllers/day/close-date/close-date.controller.ts` | É o **único** controller do serviço com `instanceof` extra além do padrão `handleHttpError` (captura `CloseDateReserveConflictError` antes) — confirmar que nenhum outro controller reintroduziu esse padrão desnecessariamente |

---

### E. Checklist de regressão (fluxos vizinhos)

| ID | Fluxo | Verificação |
|---|---|---|
| R-01 | `beach-center-app` — **Gestão de quadras** (`useCreateCourt`/`useUpdateCourt`/`useDeleteCourt`/`useGetCourtById`) | CRUD de quadras pelo painel admin funciona ponta a ponta |
| R-02 | `beach-center-app` — **Gestão de reservas admin** (`Reserve.page.tsx`, `ReserveCreate`, `ReserveEdit`, `ReserveDeleteModal`, `ReserveSearchBox`, `ReserveFindByProtocolBox`, `ReserveTable`, `ReserveViewModal`, `SchedulingSelection`) | Criar/editar/cancelar reserva pelo admin; busca por protocolo; seleção de horários (limite de 3) reflete corretamente as regras do backend |
| R-03 | `beach-center-app` — **Gestão de agendamentos/horários** (`SchedulingManagementPage`, `SchedulingManagementSearchBox`, `SchedulingManagementTable`, `useSchedulings`) | Listagem e filtro de agendamentos, indicação visual de `available` |
| R-04 | `beach-center-app` — **Criação de reserva pelo cliente** (`CreateReserve.page.tsx`, `CreateReserveForm`) | Fluxo público/autenticado de reserva funciona; mensagens de erro do backend (conflito, indisponibilidade, limite de 3) aparecem corretamente na UI |
| R-05 | `beach-center-app` — **Link público de reserva** (se a UI tiver uma tela dedicada — confirmar existência) | Fluxo criar link (admin) → abrir link → reservar sem login funciona ponta a ponta |
| R-06 | `services/beach-center-bff-pagamentos` → `agendamentos` (`create-reserve.adapter.ts`) | `POST /reservas` chamado pelo checkout de pagamentos cria a reserva corretamente antes do checkout Getnet |
| R-07 | `pagamentos` → `agendamentos` (`read-reserve.adapter.ts`) | `GET /reservas/:id` com `x-api-key` (sem Bearer) funciona — usado por pagamentos para ler dados da reserva antes/durante o checkout |
| R-08 | `pagamentos` → `agendamentos` (`update-reserve-payment-metadata.adapter.ts`) | `PATCH /reservas/:id/payment` — **maior risco de regressão de contrato cross-serviço**: confirmar que o body (`payment_id`, `checkout_id`, `payment_method`, `refund_id`, `refund_status`, `refunded_at`) e o header `x-api-key` batem exatamente com o que `pagamentos` envia após um checkout/webhook Getnet |
| R-09 | `pagamentos` → `agendamentos` (`update-reserve-status.adapter.ts`) | `PATCH /reservas/:id/status` — webhook Getnet aprovando/rejeitando pagamento reflete no `status` da reserva e libera/bloqueia agendamentos |
| R-10 | `pagamentos` → `agendamentos` (`check-public-link-authorization.adapter.ts`) | `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` — pagamentos valida autorização do link antes de processar pagamento de reserva pública |
| R-11 | `agendamentos` → `pagamentos` (`refund-reserve-payment.adapter.ts`) | Ao cancelar/deletar reserva `approved`, a chamada de estorno para `pagamentos` (`PAGAMENTOS_API_URL` + `x-api-key` = `PAGAMENTOS_INTERNAL_API_KEY`) preserva o payload/endpoint exatos |
| R-12 | Consistência de chaves internas | `AGENDAMENTOS_INTERNAL_API_KEY` configurada em `agendamentos` deve ser **idêntica** à mesma variável configurada em `pagamentos` (ambos os serviços leem a mesma chave por nomes de env diferentes: aqui é a chave que o **próprio** serviço valida nas rotas `internalApiKeyMiddleware`/`adminOrInternalApiKey`) |
| R-13 | Infra local (`beach-center-server`) | Container de `agendamentos` sobe com o novo entrypoint (`src/main.ts` via ts-node **ou** `dist/main.js`); porta default **5000** (não confundir com `pagamentos`, que default é 5001); hot-reload funciona (Princípio IV) |
| R-14 | Deploy / CI | `npm ci && npm run lint && npm test` sem erro; `npm run build` gera artefato |

---

### Rastreabilidade AC → cenário

| AC | Cenários | Cobertura automatizada equivalente? |
|---|---|---|
| **AC-1** Domínio não conhece infra | CA-02, CA-03 | ✅ ESLint `no-restricted-imports` + `container.spec` |
| **AC-2** Applications não conhece infra | CA-02, CA-03 | ✅ ESLint + `*.controller.spec` (mock do container) |
| **AC-3** Usecases dependem de ports | CA-04 (inspeção) | ✅ `*.usecase.spec` (mocks `jest.Mocked<IPort>`) |
| **AC-4** Adapters implementam ports, sem regra | CA-04 | ✅ `*.adapter.spec` |
| **AC-5** Um adapter por verbo/ação | CA-05, CA-06 | ➖ estrutural (inspeção) — notável aqui pelos 3 adapters splitados (reserva ×2, public-reserve-link ×1) |
| **AC-6** Regra de negócio migrada (agendamentos) | CA-15, H-16/H-17 (event-conflict), H-28 (limite 3 + disponibilidade), H-29/EC-08 (janela 2h), H-24/X-22 (close-date) | ✅ `event-conflict.service.spec`, `scheduling-availability.service.spec`, `event-scheduling-impact.service.spec`, `scheduling-references.validator.spec`, `reserve-scheduling.validator.spec` |
| **AC-7** `config/` + `main.ts` padronizados | CA-09 | ✅ `env.spec`, `firebase.spec`, `container.spec` |
| **AC-8** Nomes de arquivo (agendamentos) | CA-07, CA-08 | ➖ estrutural (inspeção — não há teste automatizado que falhe por nome de arquivo) |
| **AC-9** Contratos de endpoint inalterados | H-01..H-40, X-01..X-50, EC-01..EC-17 | 🟡 parcial — `*.controller.spec` afirmam status+message+shape; **comparação com baseline pré-refatoração é manual**, especialmente crítico no shape de `CLOSE_DATE_RESERVE_CONFLICT` (X-22) e no bug preservado do `equipment` (EC-13) |
| **AC-10** Middlewares de auth preservados | X-04, X-05, X-21, X-29, X-41..X-44, X-48, X-49, H-33, H-35, H-36, H-40, R-07..R-10 | ✅ `routes.spec` + `auth.middleware.spec` + `internal-api-key.middleware.spec` |
| **AC-11** ESLint configurado e limpo | CA-01 | ✅ |
| **AC-12** Cobertura ≥ 80% | CA-10 | ✅ 99,24% stmts |
| **AC-13** Testes de domínio mockam portas | CA-11 | ✅ `*.usecase.spec` |
| **AC-14** Controllers e adapters cobertos | CA-12 | ✅ 35 controllers + 60 adapters com spec |
| **AC-15** `@wip` ignorado | CA-13 | ✅ `jest.config` (nenhum `@wip` presente) |
| **AC-16** Ambiente local sobe com hot-reload | CA-14, R-13 | ➖ manual (`/run-server`) |

#### Cenários SEM cobertura automatizada equivalente (prioridade de QA manual)

- **AC-9 / não-regressão real:** H-01..H-40, X-01..X-50 — specs testam controllers isolados com mock do container; **só o teste manual contra o baseline pré-Fase-3 confirma** que shape/mensagem/status do endpoint real não mudou. Prioridade máxima: `CLOSE_DATE_RESERVE_CONFLICT` (X-22), fluxo de estorno em cascata do `cancel_reserves:true` (H-26), e o bug preservado de `equipment` no PATCH parcial de reserva (EC-13).
- **Cross-serviço com `pagamentos`:** R-06..R-12 — exigem os dois serviços rodando simultaneamente com chaves internas sincronizadas; nenhum teste unitário cobre a integração real (só mocks de `axios` em cada lado isoladamente).
- **Timing/relógio real:** EC-08 (limite exato de 2h), EC-04 (agendamento "hoje" com hora já passada), EC-11 (virada de meia-noite em `America/Sao_Paulo`) — difíceis de reproduzir com fidelidade em teste unitário determinístico.
- **Concorrência:** X-46 (dois `POST` simultâneos no mesmo link público), disputa por um `scheduling_id` entre duas reservas concorrentes (variante de EC-10).
- **Integração com o `beach-center-app`:** R-01..R-05.
- **Infra/CI:** R-13, R-14, CA-14, AC-16.

