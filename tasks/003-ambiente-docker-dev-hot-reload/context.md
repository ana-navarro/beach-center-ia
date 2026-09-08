# Task 003 — Ambiente Docker de desenvolvimento com hot-reload

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.

## Título

Configuração, dentro de `beach-center-server`, de um `docker-compose` de **desenvolvimento** que
sobe **todo o ecossistema de uma vez** (todos os microsserviços + frontend + gateway nginx + banco
+ emulador de auth), com **hot-reload** do código-fonte de cada repositório via volumes Docker,
simulando um servidor real. Rodar cada serviço isoladamente (`npm run dev`) continua possível — o
compose é uma alternativa de conveniência, não uma substituição.

## Descrição (a partir do input do usuário)

- Um único `docker compose up` sobe **todos os repositórios em paralelo**.
- Cada serviço com **hot-reload**: editar o código no host reflete no container sem rebuild.
- O ambiente **simula um servidor real completo**: as requisições passam pelo gateway nginx
  (mesmo esquema de rotas da produção), não direto nas portas dos serviços.
- Não remove a possibilidade de rodar um microsserviço por vez fora do Docker.

## Serviço(s) alvo (Princípio I)

**Repositório dono da mudança: `beach-center-server`** (infraestrutura — Princípio IV). É onde
ficam nginx, PM2 e scripts de deploy hoje; é o lugar natural para o `docker-compose` de dev.

Repositórios **orquestrados** pelo compose (bind-mount de código, sem lógica nova):

| Repositório | Papel no compose | Estado atual relevante |
|---|---|---|
| `services/beach-center-bff-agendamentos` | container Node (`npm run dev`) | `src/` hexagonal; `PORT ?? 5000`; consome `pagamentos` via HTTP (`PAGAMENTOS_API_URL`, `*_INTERNAL_API_KEY`) |
| `services/beach-center-bff-pagamentos` | container Node | `src/` hexagonal; `PORT ?? 5001` |
| `services/beach-center-bff-usuarios` | container Node | `src/` hexagonal; **`PORT ?? 5000` (colide com agendamentos)** |
| `services/beach-center-bff-aulas` | container Node | `src/` hexagonal (task 002); `PORT ?? 5002` |
| `services/beach-center-bff-campeonatos` | container Node | **vazio** — sem `src/`, sem `main.ts`, `package.json` com `"test":"teste"` apenas |
| `services/beach-center-whatsapp` | container Node | `src/`; `PORT 5003`; `.env.example` com credenciais Meta Cloud API + `AGENDAMENTOS_API_URL` + auto-reply |
| `beach-center-app` | container Vite (HMR) | `scripts.dev = "vite"`; usa `VITE_API_URL`, `VITE_AGENDAMENTOS_API_URL`, `VITE_USUARIOS_API_URL`, `VITE_PAGAMENTOS_API_URL`, `VITE_WHATSAPP_API_URL`, `VITE_FIREBASE_*` |
| `beach-center-server` (nginx) | gateway reverse-proxy | `nginx/beach-center.conf` **desatualizado**: só agendamentos/pagamentos/usuarios/whatsapp, portas inconsistentes com o código, sem `aulas`/`campeonatos`, `root` aponta para `/var/www/...` (produção) |

Serviços de infraestrutura **novos** (containers de imagem pública, sem repo próprio):
- **MongoDB** — container no compose, **banco único compartilhado** (mantém o padrão atual: cada
  serviço tem cópia read-only do schema `user`, mesmo DB), dados em volume nomeado.
- **Firebase Auth Emulator** — container no compose; serviços (`firebase-admin`) e frontend
  apontam para ele. Dev 100% offline.

### Tensão com o Princípio I (a confirmar no `/speckit-plan`)

O ideal é que **tudo** viva em `beach-center-server`. Porém, alguns pontos **podem** exigir
edições mínimas nos repos de serviço:
1. **Colisão de porta** `agendamentos` × `usuarios` (ambos default `5000`). O compose consegue
   forçar `PORT` por `environment:` sem tocar código — **mas** o default do `env.ts` de `usuarios`
   fica enganoso. Decidir: só override no compose, ou corrigir o default no repo.
2. **Firebase Emulator** — verificar se `ensureFirebaseApp()` de cada serviço já respeita
   `FIREBASE_AUTH_EMULATOR_HOST` (o `initializeApp` com `credential.cert(...)` real **não**
   funciona com emulador; `initializeApp({ projectId })` funciona). Se não respeitar, é uma
   correção no repo do serviço.
3. **`.dockerignore`** por repo (evitar copiar `node_modules`/`dist` do host no bind-mount) — ou
   resolver via `.dockerignore` único no contexto de build central.
4. **`campeonatos`** não tem `src/main.ts` — não há o que subir. Decidir se entra como stub.

Qualquer edição fora de `beach-center-server` deve ser **mínima, listada e justificada** no plano.

## Regras / decisões conhecidas (respondidas pelo usuário)

1. **Escopo: todos os repositórios, sem exceção** — agendamentos, aulas, pagamentos, usuarios,
   campeonatos, whatsapp, `beach-center-app` e o gateway nginx. Simular completamente um servidor real.
2. **MongoDB: container no próprio compose, banco único compartilhado**, volume nomeado para persistência.
3. **Firebase: Firebase Auth Emulator no compose** — nada de credenciais reais no dev.
4. **Dockerfiles centralizados em `beach-center-server`** — um `Dockerfile.dev` genérico (Node +
   watch) reutilizado pelos serviços via bind-mount + `npm run dev`. **Nenhum `Dockerfile` novo
   nos repos de serviço** (mantém o pedido "dentro do beach-center-server" e minimiza o
   cruzamento do Princípio I). O plano avalia se o frontend (Vite) e o whatsapp precisam de
   tratamento à parte.
5. **Hot-reload via volumes Docker** (Princípio IV): bind-mount `./repo:/app` + `node_modules` em
   volume anônimo/nomeado do container; comando em modo watch (`nodemon`/`ts-node`/`vite`).
6. **Rodar isolado continua válido**: `npm run dev` em cada serviço não é afetado.
7. **Alinhamento com a skill `/run-server`**: ela já espera encontrar em `beach-center-server` um
   `docker-compose*.yml` com volumes de código + comando watch + `.env.example`. Esta task
   materializa exatamente esses artefatos.

## Perguntas em aberto (para o `/speckit-plan` ou confirmação do usuário)

1. **Mapa de portas canônico.** Proposta a validar (resolve a colisão atual):
   `agendamentos 5000 · pagamentos 5001 · usuarios 5002 · aulas 5003 · campeonatos 5004 ·
   whatsapp 5005 · app 5173 (Vite) · nginx 8080 · mongo 27017 · firebase-emulator 9099/4000`.
   Manter whatsapp em `5003` (como no `.env.example`) e deslocar os demais? Ou renumerar tudo?
2. **Porta pública do gateway.** `80` (pode exigir privilégio no Windows / conflitar com outros
   serviços) ou `8080`?
3. **Esquema de rotas do nginx de dev.** Reproduzir o de produção
   (`/api/agendamentos/api/v1/`, `/api/usuarios/api/v1/`, …) e acrescentar `aulas` e `campeonatos`?
   O frontend é servido pelo nginx (proxy pro Vite, para ter HMR) ou o Vite é acessado direto?
4. **Arquivo nginx.** Criar `nginx/dev.conf` separado (deixando `beach-center.conf` como o de
   produção) ou atualizar/–substituir o existente? O `beach-center.conf` atual está claramente
   desatualizado — corrigir também é parte da task?
5. **`campeonatos` (repo vazio).** Opções: (a) omitir do compose até ter `src/`; (b) stub mínimo
   que sobe e responde healthcheck; (c) container declarado mas com `profiles:` para não subir por
   padrão. Qual?
6. **Configuração do Firebase Emulator.** Projeto fake (`demo-beach-center`?), quais emuladores
   (só `auth`, ou `auth` + `firestore`?), e como os serviços recebem `FIREBASE_AUTH_EMULATOR_HOST`
   / `GCLOUD_PROJECT` / `FIREBASE_PROJECT_ID`. O frontend (`firebase` client) usa
   `connectAuthEmulator` — precisa de flag/env? Verificar o código do `beach-center-app`.
7. **`firebase-admin` × emulador.** Confirmar (lendo o código) se `ensureFirebaseApp()` de
   agendamentos/pagamentos/usuarios/aulas funciona apontando para o emulador **sem** service
   account. Se algum usar `credential.cert(...)`, precisa de ajuste no repo.
8. **Origem das variáveis de ambiente.** Um `beach-center-server/.env.dev.example` versionado +
   `.env.dev` local (gitignored), consumido via `env_file:` no compose? Ou `environment:` inline
   por serviço? Como ficam segredos de dev (internal API keys entre agendamentos↔pagamentos —
   valores fixos de dev?).
9. **URLs inter-serviço dentro da rede do compose.** `AGENDAMENTOS_API_URL=http://agendamentos:5000/api/v1`,
   `PAGAMENTOS_API_URL=http://pagamentos:5001/api/v1`, etc. Confirmar que todos os serviços leem
   essas envs (task 001 padronizou `x-api-key` + `internalApiKeyMiddleware`).
10. **`node_modules` no container.** `npm ci` no build da imagem, ou num entrypoint na primeira
    subida? Volume nomeado por serviço (evita conflito de binários nativos host Windows ↔ container
    Linux). Como lidar quando o `package.json` de um repo muda (rebuild manual?).
11. **Hot-reload no Windows/Docker Desktop.** Bind-mount + inotify não propaga bem no Docker
    Desktop for Windows — provável necessidade de `CHOKIDAR_USEPOLLING=true` / `nodemon --legacy-watch`
    / `usePolling` no Vite. Confirmar Docker Desktop + WSL2 no host.
12. **Healthchecks e ordem de subida.** `mongo` e `firebase-emulator` `healthy` antes dos serviços
    (`depends_on: condition: service_healthy`). Os serviços Node **não têm rota `/health`** hoje —
    usar healthcheck TCP na porta, ou adicionar `/health` (cruzaria para os repos de serviço)?
13. **whatsapp sem credenciais Meta.** O `.env.example` tem `WHATSAPP_ACCESS_TOKEN=change-me`.
    Confirmar que o serviço **sobe** com valores placeholder (só o webhook/API HTTP, sem enviar
    mensagens reais) e não usa `whatsapp-web.js`/puppeteer.
14. **`beach-center-app` build em dev.** `vite` com `--host 0.0.0.0` e `server.hmr` configurado
    para funcionar atrás do proxy nginx. As `VITE_*_API_URL` apontam para o gateway
    (`http://localhost:8080/api/...`) ou para `localhost:<porta>` direto?
15. **Comando de conveniência.** Um `scripts/dev-up.sh` / `Makefile` / npm script em
    `beach-center-server`? Nome do arquivo compose (`docker-compose.yml` vs `docker-compose.dev.yml`
    — a skill `run-server` procura `docker-compose*.yml`).
16. **Seed inicial.** Rodar `beach-center-whatsapp` `npm run seed:messages` e/ou criar um usuário
    `ADMIN`/`PROFESSOR` de dev automaticamente na primeira subida? Ou fora de escopo?
17. **`.gitignore`** de `beach-center-server` para `.env.dev`, volumes locais, etc.

## Impacto arquitetural previsto

**Não se aplica a Arquitetura Hexagonal (Princípio II)** — esta é uma task de **infraestrutura
(Princípio IV)**. Não há `controllers/usecases/ports/adapters`.

Artefatos previstos em **`beach-center-server/`**:
- `docker-compose.dev.yml` (ou `docker-compose.yml`) — todos os serviços, mongo, firebase-emulator,
  nginx; redes; volumes nomeados; `depends_on` + healthchecks; bind-mounts de código.
- `docker/Dockerfile.dev` — imagem Node genérica para os BFFs (e possivelmente variações para
  `app` / `whatsapp`).
- `docker/` — scripts de entrypoint (`npm ci` condicional, wait-for-mongo), config do firebase
  emulator (`firebase.json`, `.firebaserc`).
- `nginx/dev.conf` — gateway de dev (upstreams por nome de serviço do compose, rotas incl. `aulas`
  e `campeonatos`, proxy do frontend/HMR).
- `.env.dev.example` + entrada no `.gitignore` para `.env.dev`.
- `.dockerignore` (contexto de build).
- Atualização do `README.md` de `beach-center-server` com a seção "Ambiente local via Docker".

Possíveis edições **mínimas** fora de `beach-center-server` (a decidir no plano, ver "Tensão com o
Princípio I"): default de `PORT` em `usuarios/src/config/env.ts`; suporte a
`FIREBASE_AUTH_EMULATOR_HOST` em `ensureFirebaseApp()` de um ou mais serviços; `.dockerignore` por
repo. Nada de regra de negócio.

## Fora de escopo

- Deploy/produção (PM2, `update-all.sh`, nginx de produção) — permanecem como estão; no máximo o
  `beach-center.conf` de produção é **corrigido** de passagem se o plano decidir, mas o foco é dev.
- Implementar o serviço `campeonatos` (só orquestração/stub).
- CI/CD, imagens de produção multi-stage, publicação em registry.
- Qualquer regra de negócio nos microsserviços.
