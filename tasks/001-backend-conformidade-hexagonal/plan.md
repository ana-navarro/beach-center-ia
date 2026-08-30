# Plano — 001 Conformidade dos serviços de backend com a Arquitetura Hexagonal

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.

## Contexto Técnico

### Serviços alvo (Princípio I)

| Serviço | Controllers | Usecases | Adapters | Schemas | DTO |
|---|---:|---:|---:|---:|---:|
| `services/beach-center-bff-usuarios` | 14 | 7 | 6 | 1 | 2 |
| `services/beach-center-bff-pagamentos` | 5 | 3 | 10 | 4 | 4 |
| `services/beach-center-bff-agendamentos` | 35 | 34 | 54 | 8 | 17 |

Fora do escopo (ver `context.md`): `beach-center-whatsapp` (bot não configurado),
`beach-center-bff-aulas` e `beach-center-bff-campeonatos` (repositórios vazios).

### Stack observada (idêntica nos 3 serviços)

- Node.js + TypeScript (`ts-node` em dev via `nodemon`), Express 4, Mongoose/MongoDB.
- `firebase-admin`, `axios`, `yup` (validação), `nanoid`, `mongoose-delete`.
- Entrypoint atual: **`index.js` na raiz** (CommonJS `require`), executado com `ts-node`.
  `app.use('/api/v1', routes)`.
- `tsconfig.json`: `rootDir: ./src`, `strict: true`, `module: nodenext`, `target: esnext`.
  Contém `"jsx": "react-jsx"` herdado de template de frontend (lixo).
- **Não há ESLint** em nenhum serviço de backend (existe `beach-center-app/eslint.config.js`
  no frontend, flat config, como referência).
- **Não há Jest** nem script de teste real (`"test": "test"` / placeholder).
- `serviceAccountKey.json` versionado na raiz de cada serviço (risco de segredo — fora do
  escopo desta task, registrar como pendência de segurança).

### Padrão de código atual (exemplo `usuarios/read-user`)

- O **controller** faz `new ReadUserAdapter()` e injeta no `new ReadUserUsecase(adapter)`.
  Ou seja, já existe injeção por construtor, **mas**:
  - o **usecase importa a classe concreta** do adapter (`infra/adapters/...`) como tipo;
  - o **controller importa `infra/adapters/...`** (applications → infra);
  - **não existe interface de port** entre as camadas;
  - **validação de entrada mora no controller** (`id.length !== 24`), não no usecase.

## Constitution Check

| Princípio | Situação atual | Após esta task |
|---|---|---|
| **I — Fronteiras** | CONFORME (código nos repos certos). `campeonatos` não citado na constituição → pendência, não bloqueia. | CONFORME |
| **II — Hexagonal / Ports** | **VIOLAÇÃO** (pré-existente): sem camada `ports/`; `domain/usecases` importam `infra/adapters` concretos (agendamentos ~30, usuarios 7, pagamentos 3); `applications/controllers` importam `infra/*` (quase todos); regra de negócio em `infra/adapters` (`scheduling/validators`, `events_scheduled/shared/event-conflict`); `config/` em `applications/config` (pagamentos) ou ausente; sem `main.ts` em `src/`. | **CONFORME** — o objetivo da task é eliminar essas violações. |
| **III — Test-First / Qualidade** | **VIOLAÇÃO** (pré-existente): sem ESLint, sem Jest, 0 testes em agendamentos, cobertura 0%. | **CONFORME** — Fase 0 adiciona ESLint + Jest; cada serviço sai com cobertura ≥ 80% e ESLint limpo. |
| **IV — Infra hot-reload** | Fora do código dos serviços; `beach-center-server` usa volumes. Ajuste só se `main.ts`/scripts mudarem o comando do container. | CONFORME (revisar `run-server` se o entrypoint mudar) |

**Resultado: sem violação _introduzida_ pelo plano.** Todas as violações listadas são
pré-existentes e o escopo aprovado da task é justamente corrigi-las (decisões 1–7 do
`context.md`). **Não há ERRO de bloqueio.**

## Convenções-alvo (Arquitetura Hexagonal Beach Center)

Estrutura por serviço:

```
src/
  applications/
    controllers/<recurso>/<ação>/<ação>.controller.ts   // sem import de infra/*
    routes/<recurso>.route.ts
    middlewares/
    dto/<recurso>/<ação>.dto.ts                          // schema yup + tipo de entrada/saída
  domain/
    models/<recurso>.model.ts                            // entidades de domínio
    usecases/<recurso>/<ação>/<ação>.usecase.ts          // TODA a regra de negócio
    ports/
      input/<recurso>/<ação>.port.ts                     // contrato do usecase (o que applications chama)
      output/<recurso>/<ação>.port.ts                    // contrato do que o usecase precisa da infra
  infra/
    adapters/<recurso>/<ação>/<ação>.adapter.ts          // implements domain/ports/output/...; 1 por verbo/ação; sem regra de negócio
    schemas/<recurso>.schema.ts                          // mongoose; acessado só por adapters
    ports/index.ts                                       // (opcional) reexport/barrel dos contratos de output
  config/
    env.ts                                               // leitura e validação de env vars
    container.ts                                         // composition root: factory por ação (adapter → usecase)
  main.ts                                                // bootstrap Express (substitui index.js da raiz)
```

Regras que o `/speckit-implement` deve cumprir:

1. **`domain/` nunca importa de `infra/`.** Usecases dependem de `domain/ports/output/*`
   (interfaces). A implementação concreta é injetada pelo `config/container.ts`.
2. **`applications/` nunca importa de `infra/`.** Controllers dependem de
   `domain/ports/input/*` e recebem a instância pronta do `container`.
3. **`infra/adapters/*` `implements` o port de output correspondente** e não contém `if` de
   regra de negócio — só I/O (mongoose, axios, firebase).
4. **Um adapter por verbo/ação.** Nada de adapter "genérico" com vários métodos de negócio.
5. **Regra de negócio hoje em `infra/` migra para `domain/usecases/`** (ou
   `domain/usecases/<recurso>/shared/`): `scheduling-references.validator`,
   `event-conflict` (só a parte de decisão; a query vira port de output),
   `apply-event-to-schedulings` / `release-event-from-schedulings`.
6. **Helpers de data/tempo puros** (`local-date-time`, `event-time`) vão para
   `domain/usecases/<recurso>/shared/` se forem regra, ou `src/shared/` se forem utilitário
   técnico neutro.
7. **`infra/services/*` (pagamentos)** viram `infra/adapters/<integração>/<ação>/` +
   `domain/ports/output/*`. `infra/firebase/firebase-admin.ts` é config de client → `config/`.
8. **Comportamento externo imutável**: mesmos paths, métodos, payloads, status codes,
   mensagens. Sem renomear rota, sem trocar shape de resposta.

## Mapa Arquitetural (Hexagonal)

### Fase 0 — Fundação compartilhada (aplicar nos 3 serviços)

| Camada | Criar / Alterar |
|---|---|
| lint | **criar** `.eslintrc` → adotar flat config `eslint.config.mjs` por serviço, baseado em `beach-center-app/eslint.config.js` + regra `no-restricted-imports` proibindo `domain/**` → `infra/**` e `applications/**` → `infra/**` |
| test | **criar** `jest.config.ts` (ts-jest, `collectCoverageFrom: ['src/**/*.ts', '!src/**/*.d.ts', '!**/*@wip*']`, `coverageThreshold` global 80%), instalar `jest ts-jest @types/jest` |
| scripts | **alterar** `package.json`: `"lint"`, `"lint:fix"`, `"test"`, `"test:coverage"`, `"build"`, `"start"`, `"dev"` |
| tsconfig | **alterar** remover `"jsx"`, adicionar `"types": ["node","jest"]`, garantir `noUnusedLocals`/`noImplicitReturns` |
| config | **criar** `src/config/env.ts` (valida `DB`, `PORT`, credenciais) e `src/config/container.ts` |
| entrypoint | **criar** `src/main.ts` (move a lógica de `index.js`); **alterar** `index.js` da raiz para só `require('./dist/main')` ou remover e apontar `main`/scripts para `src/main.ts` |
| infra hot-reload | **alterar** (se necessário) `beach-center-server` compose/scripts para o novo comando de start (Princípio IV) — validar via `/run-server` |

### Fase 1 — `beach-center-bff-usuarios`

| Camada | Arquivos (por família de ação) |
|---|---|
| `domain/models` | revisar `user.model.ts` (manter `IUser`); extrair tipos de auth se necessário |
| `domain/ports/output` | **criar** 1 port por adapter: `create-user`, `read-user`, `list-users`, `update-user`, `delete-user`, `user-avatar` (get/update/remove conforme adapters) — `src/domain/ports/output/user/<ação>.port.ts` |
| `domain/ports/input` | **criar** 1 port por usecase: `create-user`, `read-user`, `list-users`, `update-user`, `delete-user`, `remove-user-avatar`, `update-user-avatar` |
| `domain/usecases` | **alterar** os 7 usecases: depender do port de output, não do adapter; **mover** validações que hoje estão nos controllers (ex.: formato de `id`, regras de e-mail/senha em `auth`) para cá |
| `infra/ports` | **criar** `src/infra/ports/index.ts` (barrel dos ports de output) — opcional |
| `infra/adapters` | **alterar** os 6 adapters para `implements` o port de output; manter acesso só a `schemas` e `firebase` client |
| `infra/schemas` | revisar `user.schema.ts` (sem mudança de shape) |
| `infra/firebase` | **mover** `firebase/*` para `config/` (client) |
| `applications/dto` | **criar/alterar** dto por ação (yup) para as 5 rotas de user + 7 de auth; controller passa a validar via dto |
| `applications/controllers` | **alterar** os 14 controllers: sem `import ... infra/*`; usar `container` + port de input |
| `applications/routes` | **alterar** `auth.route.ts`, `user.route.ts`, `routes.ts` para resolver handlers do `container` |
| `config` | **criar** `container.ts` com factory de cada ação de usuário/auth |
| testes | **criar** `*.spec.ts` para os 7 usecases (100% dos ramos de regra), 6 adapters (mock mongoose/firebase), 14 controllers (mock container/usecase). Meta ≥ 80% |

### Fase 2 — `beach-center-bff-pagamentos`

| Camada | Arquivos |
|---|---|
| `domain/models` | revisar `models/*`; criar model de `transaction`/`payment` se a regra exigir |
| `domain/ports/output` | **criar** ports para: `getnet/auth`, `getnet/checkout`, `getnet/refund`, `reserva/create`, `reserva/read`, `reserva/update`, `transaction-history` (persistência), `google-drive` (upload comprovante), `manual-payment` |
| `domain/ports/input` | **criar** `create-checkout`, `create-refund`, `process-getnet-webhook`, `list-transaction-history`, `list-payment-methods` |
| `domain/usecases` | **alterar** `checkout/create-checkout`, `refund/create-refund`, `webhook/process-getnet-webhook` para usar ports; **criar** usecases para `transaction-history` e `payment-methods` (hoje a lógica está no controller/`infra/services`) |
| `infra/adapters` | **alterar** `getnet/*` e `reserva/*` para `implements` port; **converter** `infra/services/{google-drive,manual-payment,transaction-history}.service.ts` em `infra/adapters/<área>/<ação>/*.adapter.ts` |
| `infra/schemas` | revisar `schemas/*` e `schemas/getnet/*` |
| `infra/firebase` | **mover** `firebase/firebase-admin.ts` para `config/` |
| `applications/config` | **mover** `src/applications/config/*` → `src/config/*` |
| `applications/dto` | **alterar** os 4 dto; garantir validação no controller |
| `applications/controllers` | **alterar** os 5 controllers (checkout, refund, transaction-history, payment-methods, webhook): sem `infra/*` |
| `applications/routes` | **alterar** `routes.ts` para usar `container` |
| `config` | **criar** `env.ts` (chaves Getnet, Drive), `container.ts` |
| testes | `*.spec.ts` para usecases (incl. webhook: assinaturas/idempotência), adapters (mock axios/getnet, mongoose), controllers. Meta ≥ 80% |

### Fase 3 — `beach-center-bff-agendamentos`

| Camada | Arquivos (por família) |
|---|---|
| `domain/models` | revisar `court`, `events-scheduled`, `location`, `reserva`, `scheduling`, `unit`; **mover** tipos de `infra/adapters/*/types` que forem de domínio |
| `domain/ports/output` | **criar** 1 port por ação de adapter (54 adapters → agrupar por: `court/{create,read,update,delete}`, `day/{close-date,create-day,list-days,list-day-schedulings,open-date}`, `events_scheduled/{create,read,update,delete,list}`, `reserva/{create,read,update,delete,list,find-by-protocol,refund-reserve-payment}`, `scheduling/{create,read,update,delete,list}`, `unit/{create,read,update,delete,list}`, `public-reserve-link`) |
| `domain/ports/input` | **criar** 1 port por usecase (34) |
| `domain/usecases` | **alterar** os 34 usecases para depender de ports; **criar** `domain/usecases/reserva/shared/reserve-scheduling.validator.ts` sem import de infra (recebe port de leitura de scheduling + port de verificação de conflito); **mover** para `domain`: `scheduling/validators/scheduling-references.validator`, `events_scheduled/shared/{event-conflict(decisão),apply-event-to-schedulings,release-event-from-schedulings}` |
| `infra/ports` | **criar** barrel |
| `infra/adapters` | **alterar** os 54 adapters para `implements` port; **remover** dos adapters a lógica de decisão (fica só query); `event-conflict` vira um adapter de query + a decisão vai pro usecase; apagar pastas `types/`, `shared/`, `validators/` de `infra/adapters/*` após migração |
| `infra/schemas` | revisar os 8 schemas |
| `infra/firebase` | **mover** para `config/` |
| `applications/dto` | revisar os 17 dto; padronizar nome `<ação>.dto.ts` |
| `applications/controllers` | **alterar** os 35 controllers: sem `infra/*`; usar `container` + port de input; **mover** validações de formato para dto/usecase; renomear `reserva/update/update-reserve.ts` → `update-reserve.controller.ts`; `controllers/shared/update-status.ts` → local correto |
| `applications/routes` | **alterar** as 7 rotas; corrigir nomes de arquivo (`court.rotes.ts` → `court.route.ts`, `unit.routes.ts` → `unit.route.ts`) e imports em `routes.ts` |
| `config` | **criar** `env.ts`, `container.ts` (factory das ~34 ações) |
| testes | `*.spec.ts` para 34 usecases (foco: regras de reserva/evento/scheduling), 54 adapters (mock mongoose), 35 controllers. Meta ≥ 80% — suíte criada do zero |

## Checklist de Implementação

> Ordem por serviço: **usuarios → pagamentos → agendamentos**. Dentro de cada serviço:
> models → domain/ports → usecases → infra/ports → adapters → (migrar regra de negócio) →
> dto → controllers → routes → config → main.ts → testes. Um serviço por execução de
> `/speckit-implement` é recomendado.

### Fase 0 — Fundação (repetir para os 3 serviços)

> Status: **usuarios ✅** · pagamentos ⬜ · agendamentos ⬜

- [x] `services/beach-center-bff-usuarios/eslint.config.mjs` — flat config TS + `no-restricted-imports` bloqueando `**/infra/**` a partir de `**/domain/**` e `**/applications/**` (regra validada)
- [x] `services/beach-center-bff-usuarios/jest.config.ts` — ts-jest, `coverageThreshold.global` = 80, ignora `*@wip*`
- [x] `services/beach-center-bff-usuarios/package.json` — scripts `lint`, `lint:fix`, `test`, `test:coverage`, `build`, `start`, `dev`; devDeps eslint/typescript-eslint/jest/ts-jest/@types/jest/@types/node
- [x] `services/beach-center-bff-usuarios/tsconfig.json` — removido `"jsx"`, `types: ["node","jest"]`, `exclude` de specs/dist
- [x] `services/beach-center-bff-usuarios/src/config/env.ts` — leitura/validação de env (`DB` obrigatório)
- [x] `services/beach-center-bff-usuarios/src/config/container.ts` — composition root (adapter → usecase)
- [x] `services/beach-center-bff-usuarios/src/config/firebase.ts` — client Firebase Admin (movido de `infra/firebase`)
- [x] `services/beach-center-bff-usuarios/src/main.ts` — bootstrap Express migrado de `index.js`
- [x] `services/beach-center-bff-usuarios/index.js` — reduzido a `require('./dist/main')`; `package.json main` → `dist/main.js`; `.gitignore` +dist/coverage
- [ ] `beach-center-server` — conferir comando de start/volumes do container de usuarios (Princípio IV) — **pendente `/run-server`**
- [ ] Mesma fundação para `pagamentos` e `agendamentos` — **pendente**

### Fase 1 — usuarios ✅ (estrutura + código; testes na etapa `/speckit-unit-tests`)

- [x] `src/domain/models/user.model.ts` + `auth.model.ts` — tipos de domínio (sem tipos de firebase/mongoose)
- [x] `src/domain/errors.ts` — `DomainError` + subtipos com status HTTP
- [x] `src/domain/ports/output/{user-persistence,auth-provider}.port.ts` — 1 interface por verbo/ação (9 persistência + 8 auth)
- [x] `src/domain/ports/input/{user,auth}.input-port.ts` — contrato de cada usecase
- [x] `src/domain/usecases/**` — 7 CRUD refatorados + 7 de auth extraídos dos controllers (register, login, forgot-password, update-profile, update-email, update-password, authenticate-request/authorize-role); dependem só de ports; validações absorvidas dos controllers; regra de telefone movida para `domain/usecases/shared/phone-number.ts`
- [x] `src/infra/adapters/**` — cada adapter `implements` seu port de output; persistência acessa só `schemas`; auth acessa só firebase-admin/axios; sem regra de decisão (tradução de erro de provedor em `adapters/shared/firebase-error.ts`)
- [x] `src/applications/dto/**` — mantidos (yup), agora validados no controller thin
- [x] `src/applications/controllers/**` (13 + `shared/handle-http-error.ts`) — sem imports de `infra/*`; usam `container` + erros de domínio → HTTP
- [x] `src/applications/middlewares/auth.middleware.ts` — usa `container.authenticateRequest` / `authorizeRole` (sem `infra/*`)
- [x] `src/applications/routes/**` — sem alteração de contrato (handlers já resolvidos via container)
- [x] `src/config/container.ts` — factory de todas as ações de user/auth
- [x] ESLint sem erros e `tsc --noEmit` limpo
- [x] **Testes unitários (`/speckit-unit-tests`)** — 57 arquivos `*.spec.ts` (1 por arquivo de produção), 191 testes, todas as portas de infra mockadas (`firebase-admin/auth`, `axios`, `UserModel`); cobertura **99,5% stmts / 96,1% branch / 96,8% funcs / 99,5% lines** (gate `coverageThreshold` 80% ✅). Sem `@wip`.
- [x] **`/speckit-component-tests`** — N/A (task backend-only, sem alteração de UI/contrato; frontend sem Cypress).
- [x] **`/speckit-validate`** — 132 arquivos revisados 1-a-1 (produção) / bloco (testes), todos aprovados. Gates finais: ESLint 0/0, `tsc --noEmit` limpo, 191 testes verdes. Alterações staged, sem commit.
  - Decisões confirmadas na revisão: mensagem de validação unificada em "Dados inválidos"; `statusCode` em `DomainError`; `main.ts` fora da cobertura; erro de infra no middleware agora → 500 (era 401).
  - Cleanups menores anotados (não bloqueantes, para uma próxima passada): `IUpdateAuthAvatarInput` não usado; `firebase-error.ts` poderia chamar-se `provider-error.ts`; `create-user.usecase` não usa `handleUsecaseError`; `rethrowProviderError` chamado sem `return` no `update-auth-user.adapter`; casts `as IUser`; shim `applications/utils/phone-number.ts`; injetar `FIREBASE_API_KEY` em vez de ler `config` no adapter.

### Fase 2 — pagamentos

- [ ] `src/domain/models/**` — revisar; model de transação/pagamento se a regra pedir
- [ ] `src/domain/ports/output/**` — `getnet/{auth,checkout,refund}`, `reserva/{create,read,update}`, `transaction-history`, `google-drive`, `manual-payment`
- [ ] `src/domain/ports/input/**` — `create-checkout`, `create-refund`, `process-getnet-webhook`, `list-transaction-history`, `list-payment-methods`
- [ ] `src/domain/usecases/**` — refatorar checkout/refund/webhook; criar usecases de transaction-history e payment-methods
- [ ] `src/infra/adapters/**` — `getnet/*` e `reserva/*` implementam port; converter `infra/services/*` em adapters por ação
- [ ] `src/infra/schemas/**` — revisar (`schemas/getnet/*` incluso)
- [ ] `src/config/` — mover `src/applications/config/*` e `src/infra/firebase/*` para cá
- [ ] `src/applications/dto/**` (4) — validação no controller
- [ ] `src/applications/controllers/**` (5) — sem `infra/*`
- [ ] `src/applications/routes/routes.ts` — usar `container`
- [ ] `src/config/{env,container}.ts` — chaves Getnet/Drive + factories
- [ ] ESLint do serviço sem erros

### Fase 3 — agendamentos

- [ ] `src/domain/models/**` (6) — revisar; mover tipos de domínio de `infra/adapters/*/types`
- [ ] `src/domain/ports/output/**` — 1 por ação de adapter (agrupadas por recurso, ver mapa)
- [ ] `src/domain/ports/input/**` — 1 por usecase (34)
- [ ] `src/domain/usecases/**` (34) — depender de ports
- [ ] `src/domain/usecases/reserva/shared/reserve-scheduling.validator.ts` — sem import de `infra`
- [ ] `src/domain/usecases/**/shared/**` — receber `scheduling-references.validator`, `event-conflict` (decisão), `apply/release-event-to-schedulings` vindos de `infra`
- [ ] `src/infra/ports/index.ts` — barrel
- [ ] `src/infra/adapters/**` (54) — `implements` port; remover lógica de decisão; apagar `types/`,`shared/`,`validators/` sob `infra/adapters/*` após migração
- [ ] `src/infra/schemas/**` (8) — revisar
- [ ] `src/config/` — mover `src/infra/firebase/*`
- [ ] `src/applications/dto/**` (17) — padronizar `<ação>.dto.ts`
- [ ] `src/applications/controllers/**` (35) — sem `infra/*`; renomear `update-reserve.ts` → `update-reserve.controller.ts`
- [ ] `src/applications/routes/**` (7) — corrigir `court.rotes.ts`→`court.route.ts`, `unit.routes.ts`→`unit.route.ts`, ajustar `routes.ts`
- [ ] `src/config/{env,container}.ts` — factories das ~34 ações
- [ ] ESLint do serviço sem erros

### Encerramento

- [ ] Nenhum arquivo em `**/domain/**` importa de `**/infra/**` (validado por lint em todos os serviços)
- [ ] Nenhum arquivo em `**/applications/**` importa de `**/infra/**`
- [ ] Todos os adapters `implements` um `domain/ports/output/*`
- [ ] Nenhuma regra de negócio (condição de decisão) em `infra/adapters/*`
- [ ] `src/config/` e `src/main.ts` padronizados nos 3 serviços
- [ ] `beach-center-server` sobe os 3 serviços com hot-reload (`/run-server`)

## Critérios de Aceite (formais)

### Isolamento de camadas (Princípio II)

**AC-1 — Domínio não conhece infraestrutura**
- **Given** qualquer serviço alvo (`usuarios`, `pagamentos`, `agendamentos`)
- **When** o ESLint é executado (`npm run lint`) e a árvore de imports de `src/domain/**` é inspecionada
- **Then** não existe nenhum import/`require` de `src/infra/**` a partir de `src/domain/**`, e o lint passa sem erros

**AC-2 — Applications não conhece infraestrutura**
- **Given** qualquer serviço alvo
- **When** a árvore de imports de `src/applications/**` é inspecionada
- **Then** não existe import de `src/infra/**`; controllers dependem apenas de `src/domain/ports/input/**` e do `src/config/container`

**AC-3 — Usecases dependem de ports, não de adapters**
- **Given** um usecase qualquer (ex.: `read-user.usecase.ts`)
- **When** seu construtor/dependências são analisados
- **Then** o tipo injetado é uma interface de `domain/ports/output/**` e nunca a classe concreta do adapter

**AC-4 — Adapters implementam ports e não têm regra de negócio**
- **Given** um adapter qualquer em `src/infra/adapters/**`
- **When** o arquivo é inspecionado
- **Then** ele declara `implements <PortDeOutput>`, acessa apenas `infra/schemas/**` ou clients externos (axios/firebase), e não contém ramificação de regra de negócio (limites, validações de domínio, cálculo de conflito)

**AC-5 — Um adapter por verbo/ação**
- **Given** o diretório `src/infra/adapters/<recurso>/`
- **When** os arquivos são listados
- **Then** cada arquivo `*.adapter.ts` corresponde a exatamente uma ação (create/read/update/delete/list/…), sem adapter multi-ação

**AC-6 — Regra de negócio migrada para o domínio (agendamentos)**
- **Given** as regras hoje em `infra/adapters/scheduling/validators/*` e `infra/adapters/events_scheduled/shared/*`
- **When** a refatoração termina
- **Then** a lógica de decisão (conflito de evento, limite de 3 horários, horário no passado, disponibilidade, referências de scheduling) reside em `src/domain/usecases/**` e a parte de I/O virou port de output implementado por adapter

### Estrutura padrão (Princípio II)

**AC-7 — `config/` e entrypoint padronizados**
- **Given** cada serviço alvo
- **When** a estrutura de `src/` é inspecionada
- **Then** existe `src/config/env.ts`, `src/config/container.ts` e `src/main.ts`; não há mais `src/applications/config/**`; `index.js` da raiz não contém lógica de bootstrap

**AC-8 — Nomes de arquivo consistentes (agendamentos)**
- **Given** `src/applications/routes/` de agendamentos
- **When** os arquivos são listados
- **Then** todos seguem `<recurso>.route.ts` (sem `court.rotes.ts` nem `unit.routes.ts`) e todos os controllers seguem `<ação>.controller.ts`

### Preservação de comportamento (decisão 6)

**AC-9 — Contratos de endpoint inalterados**
- **Given** a coleção de endpoints de cada serviço antes da refatoração
- **When** as mesmas requisições são feitas após a refatoração (mesmo método, path sob `/api/v1`, corpo e headers)
- **Then** o status code, o shape do JSON de resposta e as mensagens são idênticos aos anteriores

**AC-10 — Middlewares de auth preservados**
- **Given** rotas hoje protegidas por `authMiddleware` / `requireRole(...)`
- **When** a rota é reconstruída no wiring novo
- **Then** os mesmos middlewares, na mesma ordem, continuam aplicados

### Qualidade e testes (Princípio III)

**AC-11 — ESLint configurado e limpo**
- **Given** cada serviço alvo
- **When** `npm run lint` é executado
- **Then** o comando existe, usa uma config própria do serviço e termina sem erros (0 warnings não justificados)

**AC-12 — Suíte Jest com cobertura ≥ 80%**
- **Given** cada serviço alvo
- **When** `npm run test:coverage` é executado
- **Then** todos os testes passam e a cobertura global (statements/lines) é ≥ 80%, com `coverageThreshold` falhando o build abaixo disso

**AC-13 — Testes de domínio mockam as portas de infra**
- **Given** os testes de `domain/usecases/**`
- **When** um teste de usecase roda
- **Then** todas as portas de output são test doubles (mocks/stubs), sem acesso real a mongoose/axios/firebase

**AC-14 — Controllers e adapters cobertos**
- **Given** a suíte de cada serviço
- **When** o relatório de cobertura é gerado
- **Then** há testes para controllers (sucesso + erro 4xx/5xx) e para adapters (mock do driver), além dos usecases

**AC-15 — Arquivos `@wip` ignorados**
- **Given** um arquivo marcado com o comentário `@wip`
- **When** a suíte roda
- **Then** seus testes são `skipped` e ele fica fora da métrica de cobertura

### Infra (Princípio IV)

**AC-16 — Ambiente local sobe com hot-reload**
- **Given** o novo `src/main.ts` e scripts
- **When** `/run-server` (ou `docker compose up`) sobe os 3 serviços
- **Then** cada serviço inicia, conecta no MongoDB e recarrega ao salvar um arquivo `.ts` (volume montado, sem rebuild)

## Riscos e observações

- **Tamanho**: agendamentos sozinho tem ~123 arquivos de código-fonte + suíte nova. Considerar
  fechar a task por serviço se `/speckit-validate` ficar longo demais.
- **`event-conflict.ts`** é usado por múltiplos usecases e hoje mistura query + decisão — a
  quebra em port + regra precisa de atenção para não alterar resultado.
- **Sem testes de referência** em agendamentos: a garantia de "sem regressão" (AC-9) depende
  de `/speckit-test` (exploratório) e dos testes de controller criados agora.
- **`serviceAccountKey.json` versionado**: risco de segredo exposto — abrir task de segurança
  separada (`git rm --cached` + rotação + `.gitignore`).
- **`campeonatos` fora da constituição**: rodar `/speckit-constitution` depois para registrar
  ou remover o repositório.

## Próximo passo

`/speckit-implement` — começar pela **Fase 0 + Fase 1 (usuarios)**.
