# Plano — 002 Sistema de Gestão de Aulas

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Serviço | Situação | Mudança |
|---|---|---|
| `services/beach-center-bff-aulas` | vazio (só `package.json`) | **bootstrap completo do zero** — dono do domínio `aula`/`aluno` |
| `services/beach-center-bff-usuarios` | em produção (hexagonal, task 001) | adicionar `"PROFESSOR"` a `UserType` |
| `services/beach-center-bff-agendamentos` | em produção (hexagonal, task 001) | **nenhuma mudança** — integração de bloqueio de quadra fica fora de escopo (ver `context.md`, o usuário está separando `events_scheduled` em modelos por domínio) |

Justificativa da fronteira: `aula`/`aluno`/`vaga` são regras de negócio de gestão de turma, não de
agendamento de quadra nem de identidade/autenticação — pertencem a um microsserviço próprio
(`aulas`), já prevґisto na Constituição (Princípio I). `usuarios` é o único dono legítimo de
`UserType`, então a adição de `"PROFESSOR"` cruza para lá em vez de `aulas` reimplementar
autenticação/perfil de usuário.

### Stack (idêntica aos demais serviços, ver `usuarios`/`pagamentos`/`agendamentos` pós-task-001)

- Node.js + TypeScript (`ts-node`/`nodemon` em dev), Express, Mongoose/MongoDB.
- `firebase-admin` (verificação de ID token), `yup` (validação de DTO), `mongoose-delete`
  (soft-delete, mesmo padrão de `court`/`unit`/etc. em `agendamentos`).
- ESLint flat config (`eslint.config.mjs`, cópia do padrão validado em usuarios/pagamentos/agendamentos,
  incl. `no-restricted-imports` bloqueando `domain/**`→`infra/**` e `applications/**`→`infra/**`).
- Jest + ts-jest, `coverageThreshold` global 80% (Princípio III) — testes ficam para
  `/speckit-unit-tests`, fora desta task.
- `config/env.ts`, `config/container.ts`, `config/firebase.ts`, `src/main.ts` — mesmo padrão de
  bootstrap (Fase 0) usado em `agendamentos` (task 001).
- **Padrão de auth entre serviços**: `agendamentos` e `pagamentos` não chamam `usuarios` via HTTP
  para autenticar — cada serviço mantém sua própria cópia read-only do schema `user` (Mongoose,
  mesmo banco compartilhado) e verifica o ID token do Firebase localmente. `aulas` segue o mesmo
  padrão: `src/infra/schemas/user.schema.ts` (mirror read-only) + `src/infra/adapters/auth/*`.

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras** | `aulas` vazio, dentro do escopo já previsto pela Constituição | CONFORME — código só em `aulas` (novo) e um campo de enum em `usuarios` |
| **II — Hexagonal / Ports** | N/A (sem código ainda) | CONFORME desde o início — `aulas` nasce já na estrutura validada (ports input/output, container, sem `domain`/`applications` importando `infra`) |
| **III — Test-First / Qualidade** | N/A | Fase 0 (ESLint + Jest configurados) entregue nesta task; testes em si ficam para `/speckit-unit-tests` — **não** é violação, é a ordem definida no Princípio V |
| **IV — Infra hot-reload** | `beach-center-server` não tem upstream/`location` nginx para `aulas` ainda (só agendamentos/pagamentos/usuarios) | Pendência registrada — requer novo serviço no `docker-compose`/`nginx/beach-center.conf` e execução de `/run-server`. Não bloqueia esta task (fora do escopo "backend e microsserviços" pedido pelo usuário), mas fica registrado para não ser esquecido |

**Resultado: sem violação. Nenhum ERRO de bloqueio.**

## Mapa Arquitetural (Hexagonal)

Estrutura (idêntica à convenção já validada em `usuarios`/`pagamentos`/`agendamentos`):

```
services/beach-center-bff-aulas/src/
  applications/
    controllers/
      aula/{create,read,update,delete,list}/*.controller.ts
      aluno/{create,read,update,delete,list}/*.controller.ts
    routes/
      aula.route.ts            // /aulas
      aluno.route.ts           // /aulas/:aula_id/alunos  (recurso aninhado)
      routes.ts
    middlewares/
      auth.middleware.ts       // authMiddleware, requireRole, requireOwnerOrAdmin
    dto/
      aula.dto.ts              // create/update (yup)
      aluno.dto.ts
  domain/
    errors.ts                  // DomainError + 5 subclasses, copiado literal de usuarios
    models/
      aula.model.ts            // IAula
      aluno.model.ts           // IAluno
    ports/
      input/
        aula.input-port.ts
        aluno.input-port.ts
        auth.input-port.ts
      output/
        aula-persistence.port.ts
        aluno-persistence.port.ts
        auth-provider.port.ts  // IVerifyIdTokenPort, IFindUserByFirestoreIdPort
    usecases/
      aula/{create,read,update,delete,list}/*.usecase.ts
      aluno/{create,read,update,delete,list}/*.usecase.ts   // list = recálculo lazy de `ativo`
      auth/authenticate-request.usecase.ts
      shared/{handle-usecase-error.ts, object-id.ts}
  infra/
    adapters/
      aula/{create,read,update,delete,list}/*.adapter.ts
      aluno/{create,read,update,delete,list}/*.adapter.ts
      auth/{verify-id-token,find-user-by-firestore-id}.adapter.ts
    schemas/
      aula.schema.ts
      aluno.schema.ts
      user.schema.ts            // mirror read-only, mesmo padrão de agendamentos
  config/
    env.ts                      // DB, PORT, FIREBASE_PROJECT_ID
    firebase.ts                 // ensureFirebaseApp (lazy)
    container.ts                // composition root
  main.ts
index.js                        // shim -> require('./dist/main')
```

### Decisões de design tomadas nesta etapa (ambiguidades não cobertas pelo `context.md`)

1. **Contagem de vagas**: `capacidade_maxima` é validada contra o total de alunos **matriculados
   não deletados** na aula (soft-delete via `mongoose-delete`), **independente do flag `ativo`**.
   Um aluno inadimplente (`ativo=false`) continua ocupando a vaga até ser explicitamente removido
   — inadimplência não libera vaga automaticamente. *(Assunção — confirmar/ajustar se divergir da
   expectativa do usuário antes do `/speckit-validate`.)*
2. **Visibilidade de leitura**: `GET /aulas` e `GET /aulas/:id` exigem apenas usuário autenticado
   (`authMiddleware`, qualquer `user_type`) — não há requisito de leitura pública nesta rodada.
   Escrita (`POST`/`PUT` em aula e em aluno) exige `ADMIN` ou o professor dono da aula
   (`requireOwnerOrAdmin`). `DELETE /aulas/:id` (apagar a turma inteira) fica restrito a `ADMIN`
   apenas — decisão de segurança, o professor gerencia alunos/vagas mas não desfaz a turma.
3. **Rota aninhada**: `aluno` é sempre acessado via `/aulas/:aula_id/alunos/...` (não existe rota
   solta `/alunos`), reforçando que aluno não existe sem uma aula.
4. **Soft-delete**: `aula` e `aluno` usam `mongoose-delete` (`deleted: {$ne: true}` nas queries),
   mesmo padrão de `court`/`unit`/`scheduling` em `agendamentos`.

## Checklist de Implementação

> Ordem: Fase 0 (fundação) → `usuarios` (adicionar PROFESSOR) → `aulas` (models → ports → usecases
> → adapters → dto → controllers → routes → auth → container → main.ts).

### Fase 0 — Fundação (`beach-center-bff-aulas`)

- [x] `eslint.config.mjs` — cópia do flat config validado em usuarios/pagamentos/agendamentos
- [x] `jest.config.ts` — ts-jest, `coverageThreshold.global` = 80, ignora `*@wip*`
- [x] `package.json` — `main: dist/main.js`, scripts (`dev`, `build`, `start`, `lint`, `lint:fix`,
      `test`, `test:coverage`), dependencies (`express`, `mongoose`, `mongoose-delete`,
      `firebase-admin`, `yup`, `cors`, `dotenv`), devDependencies completas
- [x] `tsconfig.json` — `rootDir: ./src`, `module: nodenext`, `types: ["node","jest"]`, `strict`,
      `exclude` de specs/dist/coverage
- [x] `nodemon.json` — `exec ts-node src/main.ts`
- [x] `.gitignore` — `node_modules/`, `.env`, `dist/`, `coverage/`
- [x] `src/config/env.ts` — `DB`, `PORT` (default **5002**, livre — usuarios/agendamentos=5000, pagamentos=5001), `FIREBASE_PROJECT_ID`
- [x] `src/config/firebase.ts` — `ensureFirebaseApp()`, mesmo padrão de agendamentos/pagamentos
- [x] `src/main.ts` — bootstrap Express (cors, json, `/api/v1`, mongoose.connect)
- [x] `index.js` — shim `require('./dist/main')`

### `usuarios` — adicionar `PROFESSOR`

- [x] `services/beach-center-bff-usuarios/src/domain/models/user.model.ts` — `UserType = "CLIENTE" | "ADMIN" | "PROFESSOR"`
- [x] `services/beach-center-bff-usuarios/src/infra/schemas/user.schema.ts` — `enum: ["CLIENTE", "ADMIN", "PROFESSOR"]`
- [x] `src/applications/dto/user.dto.ts` — `oneOf(['CLIENTE','ADMIN','PROFESSOR'])` no `updateUserDTO`
- [x] Revisado `register-auth.usecase.ts`: cadastro público (`POST /auth/register`) já hardcoda
      `user_type: "CLIENTE"` — autoatribuir `PROFESSOR` não é possível por lá, nenhuma mudança
      necessária. Só `PATCH /usuarios/:id` (ADMIN-only, `requireRole('ADMIN')`) pode setar
      `PROFESSOR`, o que já satisfaz "só ADMIN pode criar um PROFESSOR" sem lógica extra.
- [x] Verificado: `tsc --noEmit` limpo, `eslint .` 0 erros, suíte completa 57/191 testes verdes
      (nenhuma quebra pela adição do enum)

### `aulas` — domínio `aula`

- [x] `src/domain/errors.ts` — `DomainError` + `InvalidInputError`/`NotFoundError`/`ConflictError`/`UnauthorizedError`/`ForbiddenError` (mesmo padrão de `usuarios`)
- [x] `src/domain/models/aula.model.ts` — `IAula` + `ICreateAulaData` + `IUpdateAulaData`
- [x] `src/domain/ports/output/aula-persistence.port.ts` — `ICreateAulaPort`, `IReadAulaPort`, `IUpdateAulaPort`, `IDeleteAulaPort`, `IListAulasPort`
- [x] `src/domain/ports/input/aula.input-port.ts` — um por usecase
- [x] `src/domain/usecases/aula/{create,read,update,delete,list}/*.usecase.ts` — `assertValidObjectId` em read/update/delete; `NotFoundError` quando adapter retorna null; `create`/`update` validam que `professor` existe e tem `user_type="PROFESSOR"` via `IFindUserByIdPort` — `NotFoundError` 404 / `InvalidInputError` 400
- [x] `src/infra/schemas/aula.schema.ts` — Mongoose. **Desvio:** soft-delete manual (`deleted: Boolean` + filtro `{ deleted: { $ne: true } }`) em vez de `mongoose-delete` — o plugin não tem `@types` e quebraria `tsc --noEmit` sob `strict`; mesmo padrão real de `court`/`unit` em `agendamentos` (rationale documentada no topo do arquivo)
- [x] `src/infra/adapters/aula/{create,read,update,delete,list}/*.adapter.ts` — `implements` os ports acima
- [x] `src/applications/dto/aula.dto.ts` — yup: `createAulaDTO` (todos obrigatórios, `professor` valida ObjectId de 24 chars), `updateAulaDTO` (parcial)
- [x] `src/applications/controllers/aula/{create,read,update,delete,list}/*.controller.ts` — thin, `container.<usecase>.execute(...)`, `handleHttpError`
- [x] `src/applications/routes/aula.route.ts` — `POST/GET/PUT/DELETE /aulas`, `GET /aulas/:id`; `authMiddleware` em todas, `requireRole('ADMIN')` em delete. **Desvio:** `POST /aulas` usa `requireRole('ADMIN')` (não `requireOwnerOrAdmin`) — não há "dono" antes da aula existir, e AC-1/AC-2 descrevem o ADMIN criando; `PUT /aulas/:id` usa `requireOwnerOrAdmin`

### `aulas` — domínio `aluno`

- [x] `src/domain/models/aluno.model.ts` — `IAluno` + `ICreateAlunoData` + `IUpdateAlunoData` (`ativo` derivado no usecase/adapter, fora do payload de criação)
- [x] `src/domain/ports/output/aluno-persistence.port.ts` — `ICreateAlunoPort`, `IReadAlunoPort`, `IUpdateAlunoPort`, `IDeleteAlunoPort`, `IListAlunosPort` (por `aula_id`), `ICountAlunosByAulaPort`, `IRecalculateAlunosStatusPort`
- [x] `src/domain/ports/input/aluno.input-port.ts`
- [x] `src/domain/usecases/aluno/create/create-aluno.usecase.ts` — valida `aula` existe (via `IReadAulaPort`), valida `count >= capacidade_maxima` → `ConflictError` 409; deriva `ativo` de `vencimento_fatura >= hoje`
- [x] `src/domain/usecases/aluno/{read,update,delete}/*.usecase.ts` — thin + `assertValidObjectId`
- [x] `src/domain/usecases/aluno/list/list-alunos.usecase.ts` — **recálculo lazy**: `recalculateAlunosStatusPort.execute(aulaId)` ANTES de listar (dois `updateMany` escopados por `aula_id`: `vencimento < hoje & ativo` → `false`, `vencimento >= hoje & !ativo` → `true`) — mesmo padrão de `markPastSchedulingsUnavailable`
- [x] `src/infra/schemas/aluno.schema.ts` — Mongoose, soft-delete manual (mesmo desvio de `aula.schema.ts`), `index({ aula_id: 1 })`
- [x] `src/infra/adapters/aluno/{create,read,update,delete,list}/*.adapter.ts` + `count-by-aula` + `recalculate-status` adapters
- [x] `src/applications/dto/aluno.dto.ts` — yup (`aula_id` vem da URL, nunca do body)
- [x] `src/applications/controllers/aluno/{create,read,update,delete,list}/*.controller.ts` — thin
- [x] `src/applications/routes/aluno.route.ts` — `POST/GET /aulas/:aula_id/alunos`, `GET/PUT/DELETE /aulas/:aula_id/alunos/:id`; `Router({ mergeParams: true })`, `authMiddleware` + `requireOwnerOrAdmin` em todas (ownership resolvido pela `aula_id` da URL)

### `aulas` — auth e integração final

- [x] `src/domain/ports/output/auth.port.ts` — `IVerifyIdTokenPort`, `IFindUserByFirestoreIdPort`, `IFindUserByIdPort`
- [x] `src/infra/adapters/auth/firebase-verify-id-token.adapter.ts`; `src/infra/adapters/user/find-user-by-firestore-id.adapter.ts`, `find-user-by-id.adapter.ts`
- [x] `src/domain/usecases/auth/authenticate-request.usecase.ts` — mesmas mensagens de erro do padrão de `agendamentos`
- [x] `src/applications/middlewares/auth.middleware.ts` — `authMiddleware`, `requireRole`, **novo** `requireOwnerOrAdmin` (lê `req.params.aula_id ?? req.params.id`, carrega a aula via `container.readAula`, compara `professor` com `req.databaseUser.id`)
- [x] `src/config/container.ts` — composition root (11 usecases: 5 aula + 5 aluno + auth)
- [x] `src/applications/routes/routes.ts` — monta `aluno.route.ts` (`/aulas/:aula_id/alunos`) + `aula.route.ts` (`/aulas`) sob `/api/v1`
- [x] ESLint sem erros e `tsc --noEmit` limpo

### Encerramento

- [x] Nenhum arquivo em `domain/**` importa de `infra/**` (grep limpo)
- [x] Nenhum arquivo em `applications/**` importa de `infra/**` (grep limpo)
- [x] Todos os adapters `implements` um `domain/ports/output/*` (15/15)
- [x] Um adapter por verbo/ação (sem adapter multi-método)
- [ ] `beach-center-server` — pendência registrada (novo upstream/`location` nginx + entrada no
      compose para `aulas`), resolver via `/run-server` numa passada futura (fora desta task)

## Critérios de Aceite (formais)

### Aula — CRUD

**AC-1 — Criar aula com professor válido**
- **Given** um `ADMIN` autenticado e um usuário com `user_type="PROFESSOR"` existente
- **When** `POST /api/v1/aulas` é chamado com `classe`, `modalidade`, `dias`, `hora_inicio`,
  `hora_fim`, `professor` (id do usuário professor), `quadra`, `capacidade_maxima`
- **Then** a aula é criada (201) e retornada com todos os campos

**AC-2 — Criar aula com professor inexistente ou que não é PROFESSOR**
- **Given** um `ADMIN` autenticado
- **When** `POST /api/v1/aulas` é chamado com um `professor` que não existe, ou que existe mas
  tem `user_type` diferente de `"PROFESSOR"`
- **Then** a requisição é rejeitada (400/404, mensagem indicando professor inválido) e nenhuma
  aula é criada

**AC-3 — Ler, listar, atualizar e deletar aula**
- **Given** uma aula existente
- **When** `GET /api/v1/aulas/:id`, `GET /api/v1/aulas`, `PUT /api/v1/aulas/:id` (por `ADMIN` ou
  pelo professor dono) e `DELETE /api/v1/aulas/:id` (só `ADMIN`) são chamados
- **Then** cada operação retorna o resultado esperado (200/204) e reflete o estado atual/soft-deleted

**AC-4 — Professor não pode gerenciar aula de outro professor**
- **Given** um usuário `PROFESSOR` autenticado que não é o `professor` responsável pela aula `X`
- **When** ele chama `PUT /api/v1/aulas/X` ou qualquer rota de gestão de aluno de `X`
- **Then** a requisição é rejeitada com 403 (`Acesso negado`)

**AC-5 — Apenas ADMIN deleta a aula**
- **Given** um usuário `PROFESSOR` autenticado, dono da aula `X`
- **When** ele chama `DELETE /api/v1/aulas/X`
- **Then** a requisição é rejeitada com 403, mesmo sendo o professor responsável

### Aluno — matrícula e vagas

**AC-6 — Matricular aluno dentro da capacidade**
- **Given** uma aula com `capacidade_maxima=10` e 9 alunos matriculados (não deletados)
- **When** `POST /api/v1/aulas/:aula_id/alunos` é chamado com dados válidos de um novo aluno
- **Then** o aluno é criado (201) associado à `aula_id`

**AC-7 — Matricular aluno além da capacidade**
- **Given** uma aula com `capacidade_maxima=10` e 10 alunos matriculados (não deletados,
  independente do campo `ativo`)
- **When** `POST /api/v1/aulas/:aula_id/alunos` é chamado com um novo aluno
- **Then** a requisição é rejeitada com 409 (conflito de vagas) e nenhum aluno é criado

**AC-8 — Recalculo lazy de `ativo` na listagem**
- **Given** um aluno com `vencimento_fatura` no passado e `ativo=true` no banco
- **When** `GET /api/v1/aulas/:aula_id/alunos` é chamado
- **Then** o aluno retornado tem `ativo=false`, e o banco é atualizado para refletir isso (mesmo
  padrão de `markPastSchedulingsUnavailable` em `agendamentos`)

**AC-9 — Reativação após atualização de `vencimento_fatura`**
- **Given** um aluno com `ativo=false` cujo `vencimento_fatura` é atualizado para uma data futura
  (via `PUT` do aluno, registrando o pagamento manualmente)
- **When** `GET /api/v1/aulas/:aula_id/alunos` é chamado em seguida
- **Then** o aluno retornado tem `ativo=true`

**AC-10 — CRUD básico de aluno**
- **Given** um aluno existente numa aula
- **When** `GET`, `PUT` e `DELETE` em `/api/v1/aulas/:aula_id/alunos/:id` são chamados por `ADMIN`
  ou pelo professor dono da aula
- **Then** cada operação retorna o resultado esperado e reflete o estado atual/soft-deleted; um
  aluno removido não conta mais para `capacidade_maxima` (AC-7)

### Autenticação e autorização

**AC-11 — Rota exige autenticação**
- **Given** uma requisição sem header `Authorization: Bearer <token>` ou com token inválido
- **When** qualquer rota de `aulas`/`alunos` é chamada
- **Then** a resposta é 401 (`'Token nao fornecido'` ou `'Token invalido ou expirado'`, conforme o
  caso), mesma semântica de mensagens já usada em `agendamentos`/`usuarios`

**AC-12 — Usuário autenticado sem registro local**
- **Given** um token Firebase válido cujo `uid` não corresponde a nenhum usuário na cópia local
  de `user.schema.ts`
- **When** qualquer rota autenticada é chamada
- **Then** a resposta é 403 (`'Usuario nao encontrado no sistema'`)

### Qualidade (Princípio III — cobrado em `/speckit-unit-tests`, não nesta task)

**AC-13 — Isolamento de camadas**
- **Given** o serviço `beach-center-bff-aulas` completo
- **When** o ESLint é executado
- **Then** nenhum import de `infra/**` a partir de `domain/**` ou `applications/**`, lint sem erros

**AC-14 — Um adapter por ação**
- **Given** `src/infra/adapters/**`
- **When** os arquivos são listados
- **Then** cada arquivo corresponde a exatamente uma ação (create/read/update/delete/list/count/recalculate), sem adapter multi-ação

## Riscos e observações

- **Nova porta/serviço em `beach-center-server`**: pendência de infraestrutura (Princípio IV),
  não implementada nesta task — precisa de `/run-server` numa passada futura para expor `aulas`
  no gateway nginx.
- **Cruzamento em `usuarios`**: adicionar `"PROFESSOR"` ao enum é uma mudança pequena mas em outro
  microsserviço já hexagonal — deve ser feita com o mesmo rigor de ESLint/tsc das demais fases.
- **Sem integração com `agendamentos`**: por decisão explícita do usuário (ver `context.md`), o
  campo `quadra` na aula é só referência — não há validação cruzada de que a quadra existe em
  `agendamentos`, nem bloqueio de horário. Isso é dívida técnica conhecida e deliberada, a ser
  resolvida numa task futura após a separação de `events_scheduled` por domínio.
- **`vencimento_fatura` sem integração com `pagamentos`**: atualização é 100% manual via `PUT` do
  aluno — sem isso, o fluxo de "registrar pagamento" depende de alguém (ADMIN/professor) lembrar
  de atualizar a data.

## Próximo passo

`/speckit-implement` — sugerido começar pela Fase 0 + `usuarios` (PROFESSOR) + domínio `aula`, e
numa segunda passada o domínio `aluno` + auth + `container.ts`.
