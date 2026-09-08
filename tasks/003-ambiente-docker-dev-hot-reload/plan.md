# Plano — 003 Ambiente Docker de desenvolvimento com hot-reload

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`,
> `/speckit-unit-tests`, `/speckit-component-tests` e `/speckit-test`.

## Contexto Técnico

### Serviço(s) alvo (Princípio I)

| Repositório | Situação | Mudança nesta task |
|---|---|---|
| **`beach-center-server`** | infra (nginx, PM2, scripts de deploy) | **dono da task** — novo `docker-compose.dev.yml`, `Dockerfile.dev`, entrypoint, `nginx/dev.conf`, config do Firebase Emulator, `.env.dev.example`, `.dockerignore`, `scripts/dev-up.sh`, README |
| `services/beach-center-bff-usuarios` | produção (hexagonal) | **edição mínima guardada por env** — `config/firebase.ts` e 2 adapters de auth passam a honrar `FIREBASE_AUTH_EMULATOR_HOST`. Sem `FIREBASE_AUTH_EMULATOR_HOST` o comportamento é **idêntico ao atual** (produção intocada) |
| `beach-center-app` | produção (React/Vite) | **edição mínima guardada por env** — `config/firebase.config.ts` chama `connectAuthEmulator` quando `VITE_FIREBASE_AUTH_EMULATOR_HOST` está definido |
| `services/beach-center-bff-{agendamentos,pagamentos,aulas}` | produção (hexagonal) | **nenhuma** — o `ensureFirebaseApp()` já faz `initializeApp({ projectId })` puro, compatível com o emulador; `PORT`/`DB`/URLs inter-serviço são 100% sobrescritas via `environment:` no compose |
| `services/beach-center-whatsapp` | produção (Express/CJS) | **nenhuma** — já tem `/health`, `dev: nodemon index.js`, sobe com placeholders de credencial Meta |
| `services/beach-center-bff-campeonatos` | **repositório vazio** (só `package.json`, `"test":"teste"`, sem `src/`/`index.js`) | **nenhuma** — declarado no compose sob um *profile* que **não** sobe por padrão (não há o que executar) |

**Justificativa da fronteira**: o compose de dev é infraestrutura (Princípio IV) → pertence a
`beach-center-server`. As 3 edições fora dele (2 arquivos em `usuarios`, 1 no `app`) são o **mínimo
indispensável** para tornar o código *emulator-aware* e foram **explicitamente autorizadas pelo
usuário** ao escolher "manter o Firebase Auth Emulator". Todas ficam **atrás de uma variável de
ambiente**: quando a env não existe (produção, CI, `npm run dev` local sem Docker), o caminho de
código é exatamente o de hoje.

### Stack e ferramentas

- **Docker Engine / Docker Desktop (Windows 11 + WSL2)** + **Docker Compose v2** (`docker compose`).
- Imagem base dos containers Node: **`node:20-alpine`** (todos os serviços são Node ≥ 18; aulas/
  usuarios usam `ts-node`/`nodemon`; whatsapp usa `nodemon index.js`; app usa `vite`).
- **`mongo:7`** — banco único compartilhado, volume nomeado.
- **Firebase Emulator Suite** via imagem `node:20` + `firebase-tools` (emulador `auth`), ou imagem
  comunitária equivalente (a fixar no implement) — porta `9099` (auth) e `4000` (UI).
- **`nginx:1.27-alpine`** — gateway reverse-proxy de dev.
- Hot-reload: **bind-mount** do código de cada repo + `node_modules` em **volume nomeado por
  serviço** + comando em modo watch. Polling habilitado (`CHOKIDAR_USEPOLLING`, `--legacy-watch`)
  para o inotify funcionar através do bind-mount no Docker Desktop/Windows.

### Decisões tomadas nesta etapa (ambiguidades do `context.md`)

1. **Mapa de portas canônico** (resolve a colisão `agendamentos`×`usuarios`, ambos default `5000`).
   O compose **força** cada porta via `environment: PORT`, então vale dentro e fora do Docker:

   | Serviço | Porta interna | Exposta no host (dev) |
   |---|---|---|
   | agendamentos | 5000 | 5000 |
   | pagamentos | 5001 | 5001 |
   | usuarios | 5002 | 5002 |
   | aulas | 5003 | 5003 |
   | whatsapp | 5004 | 5004 |
   | campeonatos | 5005 | *(reservada, não sobe)* |
   | mongo | 27017 | 27017 |
   | firebase-emulator | 9099 / 4000 | 9099 / 4000 |
   | **nginx (gateway)** | 80 | **8080** |
   | beach-center-app (Vite) | 5173 | 5173 |

   > Alinhar o *default* de `PORT` no `env.ts` de cada repo (usuarios `5000`→`5002`, aulas
   > `5002`→`5003`, whatsapp `5003`→`5004`) fica como **recomendação de follow-up**, fora do
   > escopo desta task (o compose já cobre o dev).

2. **Gateway na porta `8080`** (não `80`) — evita privilégio de admin no Windows e conflito com
   outros serviços locais.

3. **`nginx/dev.conf` novo**, separado do `beach-center.conf` de produção (que fica como está — sua
   desatualização é dívida conhecida, **fora do escopo**). Upstreams por nome de serviço do compose.
   Esquema de rotas **espelha a produção** e acrescenta `aulas`:
   - `/api/agendamentos/api/v1/` → `http://agendamentos:5000/api/v1/`
   - `/api/pagamentos/api/v1/` → `http://pagamentos:5001/api/v1/`
   - `/api/usuarios/api/v1/` → `http://usuarios:5002/api/v1/`
   - `/api/aulas/api/v1/` → `http://aulas:5003/api/v1/`
   - `/api/whatsapp/api/v1/` → `http://whatsapp:5004/api/v1/`
   - `= /webhooks/getnet` → `http://pagamentos:5001/api/v1/webhooks/getnet`
   - `/` → `http://app:5173/` **com upgrade de WebSocket** (HMR do Vite)
   - `campeonatos` fica **comentado** no dev.conf com um TODO.

4. **`campeonatos`**: bloco de serviço no compose sob `profiles: ["campeonatos"]` — `docker compose
   up` **não** o inicia (o repo não tem `src/main.ts`). Quando existir código, sobe com
   `docker compose --profile campeonatos up`.

5. **Um único `docker/Dockerfile.dev` genérico** em `beach-center-server` (decisão #4 do usuário).
   `FROM node:20-alpine`, sem `COPY` de código (vem por bind-mount). Um `docker/entrypoint.dev.sh`
   roda `npm ci` quando o `node_modules` do volume está vazio/desatualizado e então `exec "$@"`.
   O comando watch de cada serviço é definido no **compose** (`command:`), não no repo:
   - BFFs TS: `npx nodemon --legacy-watch --exec "ts-node" src/main.ts`
   - whatsapp: `npx nodemon --legacy-watch index.js`
   - app: `npm run dev -- --host 0.0.0.0`

6. **Firebase Auth Emulator** (`auth` apenas nesta rodada; `firestore` fica de fora — nenhum
   serviço usa Firestore). Projeto fake **`demo-beach-center`** (o prefixo `demo-` faz o
   `firebase-admin` e o SDK cliente entrarem em modo offline sem credencial). Config em
   `beach-center-server/docker/firebase/{firebase.json,.firebaserc}`. Os serviços recebem
   `FIREBASE_AUTH_EMULATOR_HOST=firebase-emulator:9099` e `FIREBASE_PROJECT_ID=demo-beach-center`;
   o app recebe `VITE_FIREBASE_AUTH_EMULATOR_HOST=localhost:9099` e
   `VITE_FIREBASE_PROJECT_ID=demo-beach-center`.

7. **Variáveis de ambiente**: `beach-center-server/.env.dev.example` (versionado) → `cp` para
   `.env.dev` (gitignored). O compose usa `env_file: .env.dev` + `environment:` por serviço para o
   que é específico (PORT, URLs inter-serviço via DNS do compose, hosts do emulador). Chaves
   internas (`AGENDAMENTOS_INTERNAL_API_KEY`, `PAGAMENTOS_INTERNAL_API_KEY`) recebem valores fixos
   de **dev** no `.env.dev.example`.

8. **URLs inter-serviço** dentro da rede do compose:
   `PAGAMENTOS_API_URL=http://pagamentos:5001/api/v1`,
   `AGENDAMENTOS_API_URL=http://agendamentos:5000/api/v1` (whatsapp). Já são lidas de env.

9. **Healthchecks**: mongo (`mongosh --eval "db.adminCommand('ping')"`), firebase-emulator (HTTP
   em `:4000`), whatsapp (`GET /health`). Os 4 BFFs TS **não têm rota `/health`** → healthcheck por
   **resposta HTTP qualquer** em `/api/v1` via one-liner Node
   (`node -e "require('http').get('http://localhost:PORT/api/v1',r=>process.exit(0)).on('error',()=>process.exit(1))"`).
   Adicionar `GET /health` aos BFFs fica como **follow-up opcional** (cruzaria 4 repos).

10. **`node_modules`**: volume nomeado por serviço (`agendamentos_node_modules`, …). `npm ci` roda
    no entrypoint na primeira subida (volume vazio). Mudança de `package.json` → `docker compose
    run --rm <svc> npm ci` ou `docker compose build --no-cache` (documentado no README).

11. **Hot-reload no Windows**: `environment` global `CHOKIDAR_USEPOLLING=true`,
    `WATCHPACK_POLLING=true`; nodemon com `--legacy-watch`; Vite com `server.watch.usePolling`
    (via env `CHOKIDAR_USEPOLLING`, que o Vite respeita).

12. **Rodar isolado continua válido** — nenhuma mudança nos scripts `dev`/`start` dos repos; o
    `.env` local de cada serviço não é tocado.

## Constitution Check

| Princípio | Situação | Após esta task |
|---|---|---|
| **I — Fronteiras** | Task de infra → `beach-center-server`. 3 arquivos fora dele (`usuarios/src/config/firebase.ts`, `usuarios/src/infra/adapters/auth/{sign-in-with-password,send-password-reset-email}.adapter.ts`, `beach-center-app/src/config/firebase.config.ts`). | **CONFORME com exceção justificada e autorizada.** As edições são *emulator-awareness* atrás de env (sem a env = comportamento atual). Autorizadas explicitamente pelo usuário. Nenhuma regra de negócio muda. |
| **II — Hexagonal / Ports** | As edições em `usuarios` ficam em `config/` e `infra/adapters/auth/` — camadas corretas para um detalhe de infraestrutura (endpoint do provedor de auth). Nenhum `domain/`/`applications/` tocado. | **CONFORME.** |
| **III — Test-First / Qualidade** | Os specs de `usuarios` afetados: `config/firebase.spec.ts`, `infra/adapters/auth/sign-in-with-password.adapter.spec.ts`, `send-password-reset-email.adapter.spec.ts`. | **Exige atualização dos specs** (novos casos: com/sem `FIREBASE_AUTH_EMULATOR_HOST`) mantendo ESLint limpo e cobertura ≥ 80% — cobrado em `/speckit-unit-tests` e `/speckit-complete`. Compose/nginx/Dockerfile são validados por `docker compose config` + `nginx -t` (não por ESLint). |
| **IV — Infra hot-reload** | `beach-center-server` não tinha compose; hot-reload só via `npm run dev` por serviço. | **CONFORME — esta task cumpre o Princípio IV**: volumes Docker para modificação em tempo real de todos os repositórios. É a razão de existir da task. |

**Resultado: sem violação. A exceção ao Princípio I está justificada e autorizada. Nenhum ERRO de bloqueio.**

## Mapa de Artefatos

> **Arquitetura Hexagonal (Princípio II) não se aplica** — task de infraestrutura. Não há
> `controllers/usecases/ports/adapters` a criar. O mapa abaixo lista os artefatos por função.

```
beach-center-server/
  docker-compose.dev.yml            # orquestra tudo
  .env.dev.example                  # template versionado das envs de dev
  .dockerignore                     # contexto de build enxuto
  .gitignore                        # + .env.dev, dados locais
  docker/
    Dockerfile.dev                  # imagem Node genérica (BFFs + app + whatsapp)
    entrypoint.dev.sh               # npm ci condicional -> exec "$@"
    healthcheck-http.sh   (ou inline no compose)
    firebase/
      firebase.json                 # emuladores: { auth: { port: 9099 }, ui: { port: 4000 } }
      .firebaserc                   # default project: demo-beach-center
      Dockerfile.emulator           # node:20 + firebase-tools (se não usar imagem pronta)
  nginx/
    dev.conf                        # gateway de dev (novo; NÃO substitui beach-center.conf)
  scripts/
    dev-up.sh                       # docker compose -f docker-compose.dev.yml up -d --build + ps
    dev-down.sh                     # down (-v opcional)
    dev-logs.sh                     # logs -f
  README.md                         # + seção "Ambiente local via Docker"

services/beach-center-bff-usuarios/         (edição mínima, guardada por env)
  src/config/env.ts                          # + FIREBASE_AUTH_EMULATOR_HOST no tipo Env e no load
  src/config/firebase.ts                     # se EMULATOR_HOST setado -> initializeApp({ projectId }) sem cert()
  src/config/firebase.spec.ts                # novos casos
  src/infra/adapters/auth/sign-in-with-password.adapter.ts        # base URL condicional ao emulador
  src/infra/adapters/auth/sign-in-with-password.adapter.spec.ts
  src/infra/adapters/auth/send-password-reset-email.adapter.ts    # idem
  src/infra/adapters/auth/send-password-reset-email.adapter.spec.ts

beach-center-app/                            (edição mínima, guardada por env)
  src/config/firebase.config.ts              # connectAuthEmulator quando VITE_FIREBASE_AUTH_EMULATOR_HOST setado
  .env.example                               # criar/atualizar com as VITE_* de dev (incl. emulator host)
```

## Checklist de Implementação

> Ordem: fundação Docker → serviços no compose → gateway → env → emulador → edições cross-cut
> (+ specs) → scripts/README → validação.

### Fase 0 — Fundação Docker (`beach-center-server`)

- [x] `beach-center-server/docker/Dockerfile.dev` — `FROM node:20-alpine` + `tini`; ENTRYPOINT
      `/sbin/tini -- /usr/local/bin/entrypoint.dev.sh` (entrypoint **copiado** para a imagem;
      contexto de build = `./docker`)
- [x] `beach-center-server/docker/entrypoint.dev.sh` — `npm ci` condicional (stamp
      `node_modules/.dev-install-stamp` vs `package-lock.json -nt`) → `exec "$@"`
- [x] `beach-center-server/docker/.dockerignore` — contexto enxuto (só Dockerfile + entrypoint)
- [x] `beach-center-server/.gitignore` — `.env.dev`, `*.log`, logs do Firebase Emulator

### Fase 1 — `docker-compose.dev.yml` (`beach-center-server`)

- [x] Volumes nomeados: `mongo_data` + `<svc>_node_modules` (×6 BFFs, incl. campeonatos) + `app_node_modules`. Rede default do compose (`beach-center-dev_default`)
- [x] Serviço **mongo** (`mongo:7`) — `mongo_data:/data/db`, healthcheck `mongosh ping`, `27017:27017`
- [x] Serviço **firebase-emulator** — `docker/firebase/Dockerfile.emulator`, `emulators:start --project demo-beach-center`, portas `9099`/`4000`, healthcheck `wget http://localhost:9099/`
- [x] Serviço **agendamentos** — build `./docker`, bind-mount + `agendamentos_node_modules`, `command` nodemon `--legacy-watch`, env (PORT 5000, DB, FIREBASE_PROJECT_ID, FIREBASE_AUTH_EMULATOR_HOST, PAGAMENTOS_API_URL, `*_INTERNAL_API_KEY`), `depends_on` mongo+emulator `service_healthy`, `5000:5000`, healthcheck HTTP via `node -e`
- [x] Serviço **pagamentos** — idem (PORT 5001, + AGENDAMENTOS_API_URL, `5001:5001`)
- [x] Serviço **usuarios** — idem (PORT 5002, + `FIREBASE_API_KEY=demo-key`, `5002:5002`)
- [x] Serviço **aulas** — idem (PORT 5003, `5003:5003`)
- [x] Serviço **whatsapp** — `command` nodemon `index.js`, PORT 5004, DB, AGENDAMENTOS_API_URL, placeholders `WHATSAPP_*`, healthcheck `GET /health`, `5004:5004`
- [x] Serviço **campeonatos** — `profiles: ["campeonatos"]` (não sobe por padrão), PORT 5005, comentário "repo vazio"
- [x] Serviço **app** — `command: npm run dev -- --host 0.0.0.0 --port 5173`, bind-mount + `app_node_modules`, env `VITE_*` (URLs no gateway `http://localhost:8080/api/...`, `VITE_FIREBASE_*` do emulador, `VITE_FIREBASE_AUTH_EMULATOR_HOST`), `5173:5173`, `CHOKIDAR_USEPOLLING`
- [x] Serviço **nginx** (`nginx:1.27-alpine`) — monta `nginx/dev.conf` em `/etc/nginx/conf.d/default.conf:ro`, `${GATEWAY_PORT:-8080}:80`, `depends_on` dos BFFs + app
- [x] Polling global (`CHOKIDAR_USEPOLLING`, `WATCHPACK_POLLING`) via anchor `x-bff-env`

### Fase 2 — Gateway (`beach-center-server/nginx/dev.conf`)

- [x] `upstream bc_*` por serviço (nome DNS do compose + porta interna)
- [x] `location` para agendamentos, pagamentos, usuarios, **aulas**, whatsapp (esquema `/api/<svc>/api/v1/`)
- [x] `location = /webhooks/getnet` → pagamentos
- [x] `location /` → `bc_frontend` com `proxy_http_version 1.1` + `Upgrade`/`Connection` + `proxy_read_timeout 86400s` (HMR)
- [x] `campeonatos` comentado com TODO
- [x] `client_max_body_size 20m`; `proxy_set_header` padrão (Host, X-Real-IP, X-Forwarded-*)

### Fase 3 — Env e emulador

- [x] `beach-center-server/.env.dev.example` — knobs opcionais (`GATEWAY_PORT`, credenciais
      Firebase reais, chaves internas, `WHATSAPP_*`). **Decisão:** os defaults vivem inline no
      compose (valores não-secretos de dev), então `.env.dev` é **opcional** — `docker compose up`
      funciona sem ele. `cp .env.dev.example .env.dev` só para sobrescrever.
- [x] `beach-center-server/docker/firebase/firebase.json` — emuladores `auth` (9099), `ui` (4000), `hub` (4400), `logging` (4500), `singleProjectMode`
- [x] `beach-center-server/docker/firebase/.firebaserc` — `default: demo-beach-center`
- [x] `beach-center-server/docker/firebase/Dockerfile.emulator` — `node:20-alpine` + `openjdk17-jre-headless` + `firebase-tools@13`

### Fase 4 — Edições cross-cut guardadas por env

- [x] `usuarios/src/config/env.ts` — `FIREBASE_AUTH_EMULATOR_HOST?: string` no tipo `Env` e no `loadEnv`
- [x] `usuarios/src/config/firebase.ts` — com `FIREBASE_AUTH_EMULATOR_HOST` → `initializeApp({ projectId })` sem `cert()`; sem a env → fluxo atual (cert exigido)
- [x] `usuarios/src/infra/adapters/auth/identity-toolkit.ts` — **novo** helper `identityToolkitBaseUrl()` (emulador vs Google)
- [x] `usuarios/src/infra/adapters/auth/sign-in-with-password.adapter.ts` — usa `identityToolkitBaseUrl()`
- [x] `usuarios/src/infra/adapters/auth/send-password-reset-email.adapter.ts` — usa `identityToolkitBaseUrl()`
- [x] `usuarios/src/config/firebase.spec.ts` — + casos: modo emulador (só projectId, sem cert), emulador exige projectId, produção com creds parciais → `firebase.ts` 100%
- [x] `usuarios/src/infra/adapters/auth/identity-toolkit.spec.ts` — **novo** (Google vs emulador)
- [x] `usuarios/src/infra/adapters/auth/sign-in-with-password.adapter.spec.ts` — + caso URL do emulador
- [x] `usuarios/src/infra/adapters/auth/send-password-reset-email.adapter.spec.ts` — + caso URL do emulador
- [x] `usuarios/src/config/env.spec.ts` — + passthrough / omissão de `FIREBASE_AUTH_EMULATOR_HOST`
- [x] `beach-center-app/src/config/firebase.config.ts` — `connectAuthEmulator` quando `VITE_FIREBASE_AUTH_EMULATOR_HOST` setado
- [x] `beach-center-app/.env.example` — **novo**, todas as `VITE_*` + `VITE_FIREBASE_AUTH_EMULATOR_HOST`, dev vs prod

### Fase 5 — Conveniência e docs

- [x] `beach-center-server/scripts/dev-up.sh` — `up -d --build` (+ `--env-file .env.dev` se existir) + `ps` + resumo de URLs
- [x] `beach-center-server/scripts/dev-down.sh` — `down` (`-v` repassado)
- [x] `beach-center-server/scripts/dev-logs.sh` — `logs -f --tail=100 [serviço]`
- [x] `beach-center-server/README.md` — seção "Ambiente local via Docker" (pré-requisitos, layout
      de pastas, subir, tabela de portas, URLs, comandos, reinstalar deps, Firebase Emulator,
      dívidas conhecidas) + reorganização "Produção" abaixo

### Fase 6 — Validação

> **Docker não está instalado nesta máquina** — os itens que exigem `docker`/`docker compose`
> ficam para o desenvolvedor rodar (ou para o `/speckit-complete`).

- [x] `docker-compose.dev.yml` — YAML válido, anchors resolvidos (parse via `js-yaml`): 10
      serviços, 8 volumes, comandos/env corretos por serviço
- [x] `bash -n` limpo nos 3 scripts + `entrypoint.dev.sh`
- [ ] `docker compose -f docker-compose.dev.yml config` — **pendente (sem Docker local)**
- [ ] `docker compose ... up -d --build` — todos `healthy` — **pendente (sem Docker local)**
- [ ] `nginx -t` no container — **pendente (sem Docker local)**
- [ ] `curl http://localhost:8080/api/usuarios/api/v1/...` pelo gateway — **pendente**
- [ ] hot-reload de um BFF (nodemon reinicia) — **pendente**
- [ ] HMR do app via `http://localhost:8080` — **pendente**
- [ ] `down && up` → Mongo persiste — **pendente**
- [x] ESLint (0) + `tsc --noEmit` (0) + testes (**198/198**) verdes em `beach-center-bff-usuarios`
- [x] `tsc -b` (0) + ESLint no arquivo alterado (0) em `beach-center-app` — **nota:** o repo tem
      **36 erros de ESLint pré-existentes** (em `SidebarItems.ts`, `Register.page.tsx` etc.), não
      introduzidos por esta task; `firebase.config.ts` (alterado) está limpo

## Critérios de Aceite (formais)

### Ambiente unificado

**AC-1 — Subida completa com um comando**
- **Given** um checkout limpo com Docker Desktop rodando e `.env.dev` criado a partir de `.env.dev.example`
- **When** o desenvolvedor executa `./scripts/dev-up.sh` (ou `docker compose -f beach-center-server/docker-compose.dev.yml up -d --build`)
- **Then** sobem, e ficam `healthy`, os containers: `mongo`, `firebase-emulator`, `agendamentos`,
  `pagamentos`, `usuarios`, `aulas`, `whatsapp`, `app` e `nginx` — sem `campeonatos` (profile desativado)

**AC-2 — Gateway espelha o roteamento de produção**
- **Given** o ambiente no ar
- **When** o desenvolvedor chama `http://localhost:8080/api/<serviço>/api/v1/<rota>` para
  `agendamentos`, `pagamentos`, `usuarios`, `aulas` e `whatsapp`
- **Then** a requisição é encaminhada ao container correto na porta interna correspondente e a
  resposta do serviço volta pelo gateway (mesmo esquema de path do `nginx/beach-center.conf` de produção, + `aulas`)

**AC-3 — Frontend servido pelo gateway com HMR**
- **Given** o ambiente no ar
- **When** o desenvolvedor abre `http://localhost:8080/` e edita um componente em `beach-center-app/src`
- **Then** a página é servida (proxy para o Vite) e a alteração aparece via Hot Module Replacement
  sem reload manual e sem rebuild de container

### Hot-reload

**AC-4 — Hot-reload de um microsserviço**
- **Given** o container `aulas` (ou qualquer BFF) rodando
- **When** o desenvolvedor salva uma alteração em `services/beach-center-bff-aulas/src/**/*.ts`
- **Then** o `nodemon` dentro do container detecta a mudança (polling) e reinicia o processo em
  segundos, **sem** `docker build` nem `docker restart`

**AC-5 — `node_modules` isolado do host**
- **Given** o host é Windows e os containers são Linux
- **When** o ambiente sobe
- **Then** cada serviço usa o `node_modules` do seu volume nomeado (instalado no container), sem
  conflito com um eventual `node_modules` do host, e o código-fonte é o do bind-mount

### Persistência e isolamento

**AC-6 — Dados do Mongo sobrevivem a `down`/`up`**
- **Given** dados gravados no Mongo pelo ambiente
- **When** o desenvolvedor roda `docker compose ... down` (sem `-v`) e sobe de novo
- **Then** os dados continuam lá (volume `mongo_data` preservado); `down -v` limpa

**AC-7 — Banco único compartilhado**
- **Given** os BFFs no ar
- **When** `usuarios` grava um usuário e `aulas`/`agendamentos` leem a coleção `usuarios` (mirror read-only)
- **Then** todos enxergam o mesmo banco `beach-center` no mesmo container Mongo

### Autenticação via emulador

**AC-8 — Verificação de ID token contra o emulador**
- **Given** o Firebase Auth Emulator no ar e `FIREBASE_AUTH_EMULATOR_HOST` setado nos BFFs
- **When** um ID token emitido pelo emulador é enviado a uma rota autenticada de qualquer BFF
- **Then** o `authMiddleware` valida o token contra o emulador (não contra o Google) e a
  requisição prossegue; **sem** a env, o código valida contra o Google exatamente como hoje

**AC-9 — Login e reset de senha do `usuarios` usam o emulador em dev**
- **Given** `FIREBASE_AUTH_EMULATOR_HOST` definido
- **When** `POST /api/v1/auth/login` (e o fluxo de reset de senha) é chamado
- **Then** o adapter direciona a chamada REST ao host do emulador
  (`http://<host>/identitytoolkit.googleapis.com/...`) em vez de `https://identitytoolkit.googleapis.com/...`;
  **sem** a env, a URL é a de produção (comportamento e testes atuais inalterados)

**AC-10 — Login pelo frontend no ambiente Docker**
- **Given** `VITE_FIREBASE_AUTH_EMULATOR_HOST` definido no build de dev do app
- **When** o app inicializa o Firebase
- **Then** chama `connectAuthEmulator` e as operações de auth do browser vão para o emulador;
  **sem** a env, `connectAuthEmulator` **não** é chamado (produção intocada)

### Coexistência com o fluxo atual

**AC-11 — Rodar um serviço isolado continua funcionando**
- **Given** o repositório após esta task
- **When** o desenvolvedor roda `npm run dev` dentro de `services/beach-center-bff-pagamentos` (fora do Docker), como antes
- **Then** o serviço sobe normalmente — nenhum script `dev`/`start`/`build` de repo foi alterado,
  nenhum `.env` local foi tocado

**AC-12 — `campeonatos` não quebra a subida**
- **Given** o repo `beach-center-bff-campeonatos` vazio
- **When** `docker compose ... up` roda sem `--profile campeonatos`
- **Then** o ambiente sobe completo e o `campeonatos` simplesmente não é iniciado (nenhum erro de build/exec)

### Qualidade (Princípio III — cobrado em `/speckit-unit-tests` e `/speckit-complete`)

**AC-13 — Repos tocados permanecem verdes**
- **Given** as edições cross-cut em `beach-center-bff-usuarios` e `beach-center-app`
- **When** rodam `npx eslint .`, `npx tsc --noEmit` e a suíte de testes de cada um
- **Then** ESLint 0 erros, `tsc` limpo, todos os testes passando e cobertura de `usuarios` ≥ 80%
  (specs de `firebase` e dos 2 adapters de auth atualizados com os casos com/sem emulador)

**AC-14 — Compose e nginx válidos**
- **Given** os artefatos de infra
- **When** rodam `docker compose -f beach-center-server/docker-compose.dev.yml config` e `nginx -t` (no container)
- **Then** ambos retornam sucesso, sem referência a serviço/porta/volume inexistente

## Riscos e observações

- **Firebase Auth Emulator exige JRE** na imagem — se a imagem do emulador não tiver Java, o
  `emulators:start` falha. Mitigado por `openjdk17-jre` no `Dockerfile.emulator`.
- **Polling de arquivos** aumenta uso de CPU no Windows; aceitável para dev. Documentar no README
  que em macOS/Linux nativo o polling pode ser desligado.
- **`npm ci` na primeira subida** deixa o boot inicial lento (6 containers instalando deps). Aceitável;
  o README explica. Builds seguintes reusam o volume.
- **Divergência de porta entre o `env.ts` de cada repo e o mapa canônico** — o compose mascara isso
  no dev, mas alguém rodando 2 serviços isolados (ex.: `usuarios` + `aulas`) ainda tromba se não
  passar `PORT`. Follow-up recomendado (fora do escopo): alinhar os defaults.
- **`beach-center.conf` de produção continua desatualizado** (sem `aulas`, portas inconsistentes).
  Corrigir é dívida conhecida, deliberadamente **fora do escopo** desta task de dev.
- **`beach-center-server` está em `main`** (branch default) — o `/speckit-complete` criará um branch
  de feature antes de commitar.
- Testes de componente Cypress/Cucumber **não se aplicam** (task de infra, sem UI nova) — mesma
  situação das tasks 001 e 002.

## Próximo passo

`/speckit-implement` — sugerido começar pela Fase 0/1 (Dockerfile + compose com mongo + emulador +
1 BFF), validar o hot-reload de um serviço, e então replicar para os demais, gateway, e por último
as edições cross-cut com seus specs.
