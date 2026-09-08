# Testes Exploratórios — 003 Ambiente Docker de desenvolvimento com hot-reload

> Gerado por `/speckit-test`. Roteiro de teste **manual** (o projeto não tem QA dedicado).
> A maioria dos Critérios de Aceite desta task só é verificável com Docker rodando — foram
> **impossíveis de automatizar** nesta máquina (Docker não instalado). Este roteiro é o
> caminho principal de verificação da task 003.
>
> Base: `context.md` + `plan.md` (Critérios de Aceite `AC-1`..`AC-14`).

---

## 1. Escopo e pré-condições

### O que a task entrega

Um `docker-compose.dev.yml` em `beach-center-server` que sobe **todo o ecossistema** (4 BFFs +
whatsapp + frontend + gateway nginx + MongoDB + Firebase Auth Emulator) com **hot-reload** do
código de cada repositório. Rodar um serviço isolado com `npm run dev` continua funcionando.

### Pré-condições de ambiente

| Item | Requisito |
|---|---|
| Docker | Docker Desktop com backend **WSL2** (Windows 11) ou Docker Engine + Compose v2 (`docker compose`, não `docker-compose`) |
| Recursos | ~6 GB RAM livres para o Docker; ~5 GB disco (imagens + `node_modules` em volumes) |
| Portas livres no host | `8080` (gateway), `5000`–`5004`, `5173`, `27017`, `9099`, `4000` |
| Layout de pastas | `beach-center-app/`, `beach-center-server/` e `services/beach-center-bff-*` como pastas irmãs sob `beach-center-ia/` |
| Repositórios | Todos clonados. `beach-center-bff-campeonatos` pode estar vazio (esperado). |

### Arquivos-chave desta task

- `beach-center-server/docker-compose.dev.yml`, `docker/Dockerfile.dev`, `docker/entrypoint.dev.sh`
- `beach-center-server/docker/firebase/{firebase.json,.firebaserc,Dockerfile.emulator}`
- `beach-center-server/nginx/dev.conf`
- `beach-center-server/scripts/dev-{up,down,logs}.sh`
- `beach-center-server/.env.dev.example`
- `services/beach-center-bff-usuarios/src/config/{env.ts,firebase.ts}` + `src/infra/adapters/auth/{identity-toolkit.ts,sign-in-with-password.adapter.ts,send-password-reset-email.adapter.ts}`
- `beach-center-app/src/config/firebase.config.ts` + `beach-center-app/.env.example`

### Primeira subida (executar antes dos cenários)

```bash
cd beach-center-server
cp .env.dev.example .env.dev          # opcional
./scripts/dev-up.sh                    # 1ª vez: instala deps de cada serviço (alguns minutos)
docker compose -f docker-compose.dev.yml ps
```

### Convenção das tabelas

`ID | Pré-condição | Passos | Resultado esperado`. Coluna **Auto?**: `sim` = há teste unitário
equivalente (`/speckit-unit-tests`); `manual` = só aqui (exige Docker/observação).

---

## 2. Caminhos felizes

| ID | Pré-condição | Passos | Resultado esperado | Auto? | AC |
|---|---|---|---|---|---|
| H-01 | `.env.dev` criado (ou ausente — defaults), Docker rodando | `./scripts/dev-up.sh` | Build sem erro; `docker compose ps` mostra `mongo`, `firebase-emulator`, `agendamentos`, `pagamentos`, `usuarios`, `aulas`, `whatsapp`, `app`, `nginx` — todos `Up` e, após ~1–2 min, `healthy`. `campeonatos` **não** aparece. | manual | AC-1, AC-12 |
| H-02 | Ambiente no ar | `curl -i http://localhost:8080/api/usuarios/api/v1/` | Resposta HTTP do container `usuarios` (não erro de conexão do nginx). Status pode ser 404 (rota raiz) — o que importa é que **passou pelo gateway**. | manual | AC-2 |
| H-03 | Ambiente no ar | `curl -s http://localhost:8080/api/whatsapp/api/v1/` e `curl -s http://localhost:8080/api/aulas/api/v1/aulas -H "Authorization: Bearer <token>"` | Cada rota chega ao serviço certo na porta interna certa (5004 / 5003) | manual | AC-2 |
| H-04 | Ambiente no ar | Abrir `http://localhost:8080/` no browser | O frontend (Vite) é servido através do nginx; a página carrega | manual | AC-3 |
| H-05 | H-04, DevTools aberto | Editar um texto em `beach-center-app/src/**/*.tsx` e salvar | HMR atualiza a tela em segundos, **sem** reload manual e **sem** rebuild de container; console do Vite mostra `hmr update` | manual | AC-3 |
| H-06 | Ambiente no ar, `./scripts/dev-logs.sh aulas` aberto | Editar `services/beach-center-bff-aulas/src/main.ts` (ex.: mudar uma string de log) e salvar | `nodemon` no container detecta (`[nodemon] restarting due to changes...`) e reinicia o processo em poucos segundos | manual | AC-4 |
| H-07 | Idem para outro BFF | Editar `services/beach-center-bff-usuarios/src/**/*.ts` | nodemon reinicia o `usuarios` | manual | AC-4 |
| H-08 | Ambiente no ar | `docker compose exec agendamentos ls -la node_modules \| head` e `ls services/beach-center-bff-agendamentos/node_modules` no host | Container tem `node_modules` populado (volume nomeado); host pode estar **sem** `node_modules` ou com um diferente — não há conflito | manual | AC-5 |
| H-09 | Mongo com dados (criar um registro via qualquer BFF) | `./scripts/dev-down.sh` (sem `-v`) e depois `./scripts/dev-up.sh` | Os dados continuam no Mongo (volume `mongo_data` preservado) | manual | AC-6 |
| H-10 | Ambiente no ar | `docker compose exec mongo mongosh --quiet --eval "db.getMongo().getDBNames()"` após `usuarios` e `aulas` gravarem | Existe **um** database `beach-center` no mesmo container, compartilhado | manual | AC-7 |
| H-11 | Emulator UI em `http://localhost:4000` | Criar um usuário no emulador (via UI ou app), fazer login e chamar uma rota autenticada de um BFF com o ID token | O `authMiddleware` do BFF valida o token **contra o emulador** (`FIREBASE_AUTH_EMULATOR_HOST`) e a requisição prossegue (200/403 conforme o `user_type`, não 401 de token inválido) | parcial | AC-8 |
| H-12 | `usuarios` no ar com `FIREBASE_AUTH_EMULATOR_HOST` | `POST http://localhost:8080/api/usuarios/api/v1/auth/login` com email/senha de um usuário criado no emulador | Login bem-sucedido; o adapter chamou `http://firebase-emulator:9099/identitytoolkit.googleapis.com/...` (ver log/UI do emulador), não o Google | parcial | AC-9 |
| H-13 | App no ar via Docker | No Devtools> Network, disparar um login | As chamadas de auth do browser vão para `localhost:9099` (emulador), não `identitytoolkit.googleapis.com`; sem warning ruidoso (`disableWarnings: true`) | manual | AC-10 |
| H-14 | Repo após a task | `cd services/beach-center-bff-pagamentos && npm run dev` (fora do Docker, com o ambiente Docker **parado**) | O serviço sobe normalmente — nenhum script `dev/start/build` foi alterado; `.env` local intacto | manual | AC-11 |
| H-15 | Repo `beach-center-bff-campeonatos` vazio | `docker compose -f docker-compose.dev.yml up` (sem `--profile campeonatos`) | Ambiente sobe completo; `campeonatos` simplesmente não inicia (sem erro de build/exec) | manual | AC-12 |
| H-16 | Repo `beach-center-bff-usuarios` | `npx eslint . && npx tsc --noEmit && npx jest` | ESLint 0, tsc 0, **206/206** testes, cobertura ≥ 80% | **sim** | AC-13 |
| H-17 | Artefatos de infra | `docker compose -f beach-center-server/docker-compose.dev.yml config` e `docker compose exec nginx nginx -t` | Ambos retornam sucesso, sem serviço/porta/volume/upstream inexistente | manual | AC-14 |

---

## 3. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado | Auto? |
|---|---|---|---|---|
| E-01 | Algo já ouvindo na porta `8080` (ou `5000`, `27017`...) | `./scripts/dev-up.sh` | Docker falha ao publicar a porta com mensagem clara (`address already in use`). Mitigação: `GATEWAY_PORT=8081` no `.env.dev` para o gateway; para as demais, liberar a porta ou ajustar o mapeamento no compose | manual |
| E-02 | Docker Desktop **parado** | `./scripts/dev-up.sh` | `docker` retorna `Cannot connect to the Docker daemon` — o script para com erro, sem estado parcial | manual |
| E-03 | `.env.dev` **não existe** | `./scripts/dev-up.sh` | Script imprime "usando os defaults do compose" e sobe normalmente (o `.env.dev` é opcional) | manual |
| E-04 | `.env.dev` com sintaxe inválida (linha sem `=`) | `./scripts/dev-up.sh` | `docker compose --env-file` reclama da linha inválida; corrigir o arquivo | manual |
| E-05 | Sem rede / registry inacessível na 1ª subida | `./scripts/dev-up.sh` | `docker build` / `npm ci` falham ao baixar. O `entrypoint.dev.sh` não cria o stamp, então uma nova subida com rede tenta de novo | manual |
| E-06 | Imagem do emulador sem JRE (regressão do `Dockerfile.emulator`) | subir | `firebase emulators:start` falha com erro de Java; o container fica `unhealthy` e os BFFs não passam do `depends_on: service_healthy`. Confirma que `openjdk17-jre-headless` é obrigatório | manual |
| E-07 | Um BFF com erro de compilação (TS quebrado no fonte) | salvar o arquivo quebrado | nodemon tenta reiniciar, `ts-node` falha e loga o erro; o container continua `Up` mas `unhealthy`. Corrigir o fonte → nodemon reinicia e volta a `healthy` | manual |
| E-08 | `package.json` de um serviço alterado (nova dependência) | salvar e observar | O container **não** reinstala sozinho (o fonte mudou, não o `node_modules` do volume). É preciso `docker compose run --rm <svc> npm ci` ou `dev-down.sh -v && dev-up.sh`. Documentado no README | manual |
| E-09 | `nginx/dev.conf` com erro de sintaxe | `docker compose restart nginx` | nginx falha ao recarregar; `docker compose logs nginx` mostra a linha. O compose monta o arquivo `:ro`, então a correção é no host + restart | manual |
| E-10 | Um BFF referenciado no nginx está `Down` (ex.: `docker compose stop aulas`) | `curl http://localhost:8080/api/aulas/api/v1/` | `502 Bad Gateway` do nginx (upstream indisponível). Subir o serviço resolve; nginx (`restart: unless-stopped`) reconecta | manual |
| E-11 | `FIREBASE_AUTH_EMULATOR_HOST` **removido** do serviço `usuarios` no compose, sem credenciais reais | subir `usuarios` | `ensureFirebaseApp()` lança `"Credenciais do Firebase Admin nao configuradas..."` na 1ª rota autenticada — comportamento **correto** (produção exige service account) | **sim** |
| E-12 | Token do emulador enviado a um BFF **sem** `FIREBASE_AUTH_EMULATOR_HOST` | chamar rota autenticada | `firebase-admin` tenta validar contra o Google e falha → `401`. Confirma que a variável é o que liga os dois lados | manual |
| E-13 | whatsapp sem credenciais Meta (placeholders `dev-*`) | subir e `curl http://localhost:8080/api/whatsapp/api/v1/` | O bot **sobe** e responde no `/health`; só o envio real de mensagem falha (esperado — sem token válido) | manual |
| E-14 | `beach-center-app` com os 36 erros de ESLint pré-existentes | `cd beach-center-app && npx eslint .` | 36 erros (não introduzidos pela task 003). Só relevante para o `/speckit-complete` — decidir corrigir à parte ou escopar o gate ao diff | parcial |

---

## 4. Edge cases

| ID | Pré-condição | Passos | Resultado esperado | Auto? |
|---|---|---|---|---|
| B-01 | Host Windows, Docker Desktop/WSL2 | editar um `.ts` e cronometrar o restart do nodemon | O restart ocorre (polling via `--legacy-watch` + `CHOKIDAR_USEPOLLING`), mas com **latência de 1–3 s** e CPU mais alta — aceitável para dev. Em macOS/Linux nativo o polling pode ser desligado | manual |
| B-02 | 1ª subida (volumes `*_node_modules` vazios) | `./scripts/dev-up.sh` e observar os logs | Cada serviço roda `npm ci` no `entrypoint.dev.sh` antes de iniciar; a subida inicial leva **vários minutos** (6 serviços). Subidas seguintes reusam o volume e sobem rápido | manual |
| B-03 | Volume `node_modules` já populado, sem mudança de lock | `dev-down.sh` (sem `-v`) + `dev-up.sh` | `entrypoint.dev.sh` detecta o stamp `.dev-install-stamp` e **pula** o `npm ci` | manual |
| B-04 | `package-lock.json` de um serviço mais novo que o stamp | subir | `entrypoint.dev.sh` reexecuta `npm ci` (regra `package-lock.json -nt stamp`) | manual |
| B-05 | `beach-center-bff-campeonatos` recebeu `src/main.ts` no futuro | `docker compose -f docker-compose.dev.yml --profile campeonatos up -d` | O serviço `campeonatos` (PORT 5005) sobe; descomentar o `location` correspondente em `nginx/dev.conf` habilita a rota no gateway | manual |
| B-06 | Ambiente Docker no ar **e** `npm run dev` de um serviço rodando no host ao mesmo tempo | tentar as duas coisas | Colisão de porta no host (ex.: `5002` publicado pelo container `usuarios` e o `npm run dev` local também querendo `5002`). Escolher um dos dois modos por vez, ou mudar a porta local | manual |
| B-07 | Dois BFFs rodando **isolados** (fora do Docker) sem passar `PORT` | `npm run dev` em `usuarios` e depois em `aulas` | `usuarios` default `5000`... `aulas` default `5002` (task 002 assumiu usuarios=5000). Divergência do mapa canônico. Follow-up conhecido: alinhar os defaults do `env.ts`. Passar `PORT=5002 npm run dev` contorna | manual |
| B-08 | HMR atrás do proxy nginx | editar um `.tsx`; observar a aba Network > WS | O WebSocket do HMR conecta em `ws://localhost:8080/` e o nginx faz `Upgrade` para `app:5173` (`proxy_read_timeout 86400s` evita a queda). Se o HMR não conectar, acessar `http://localhost:5173` direto ainda funciona | manual |
| B-09 | Emulador reiniciado (`docker compose restart firebase-emulator`) | após o restart, tentar autenticar | Os usuários criados no emulador **somem** (o emulador de auth não persiste por padrão). Recriar. Não é bug da task — é o comportamento do emulador | manual |
| B-10 | `dev-down.sh -v` | rodar | Apaga `mongo_data` **e** todos os `*_node_modules` — a próxima subida reinstala tudo (lento). Usar só quando quiser ambiente limpo de verdade | manual |
| B-11 | `mongo:7` — versão | subir | Usa MongoDB 7. Se algum serviço espera um recurso de outra versão, ajustar a tag no compose. `mongosh` vem embutido na imagem (healthcheck depende disso) | manual |
| B-12 | Alterar `docker-compose.dev.yml` (ex.: adicionar env) | `./scripts/dev-up.sh` de novo | `--build` reconstrói só o que mudou; serviços com env novo são recriados. Não precisa `down` | manual |
| B-13 | Nome do projeto compose | `docker compose ls` | Aparece como `beach-center-dev` (definido em `name:` no compose), isolando volumes/rede de outros projetos | manual |
| B-14 | `curl` no webhook do Getnet | `curl -X POST http://localhost:8080/webhooks/getnet` | Encaminhado para `pagamentos:5001/api/v1/webhooks/getnet` (rota exata, `location = /webhooks/getnet`) | manual |
| B-15 | `VITE_*` do compose vs `.env` local do `beach-center-app` | subir o app via Docker | O compose injeta `VITE_*` por `environment:` — tem precedência sobre um `.env` do repo? Confirmar que o app no container usa as URLs do gateway (`http://localhost:8080/api/...`) e o emulador | manual |

---

## 5. Checklist de regressão (produção / fluxos vizinhos)

| ID | Área | O que verificar | Motivo do risco |
|---|---|---|---|
| R-01 | `beach-center-bff-usuarios` — produção | **Sem** `FIREBASE_AUTH_EMULATOR_HOST`, `ensureFirebaseApp()` continua exigindo e usando `cert()` com service account, idêntico a antes | `firebase.ts` foi ramificado |
| R-02 | `usuarios` — login / reset de senha em produção | Sem a env de emulador, os adapters chamam `https://identitytoolkit.googleapis.com/v1/...` (URL inalterada) | `identity-toolkit.ts` novo |
| R-03 | `usuarios` — suíte completa | `npx jest` verde (**206/206**, era 198 antes; +8 casos da task) e cobertura ≥ 80% | mudança em `config/` + `infra/adapters/auth/` de serviço em produção |
| R-04 | `usuarios` — ESLint / tsc | `npx eslint .` = 0, `npx tsc --noEmit` = 0 | Princípio III |
| R-05 | `beach-center-app` — produção | Sem `VITE_FIREBASE_AUTH_EMULATOR_HOST`, `connectAuthEmulator` **não** é chamado; `getAuth` aponta para o Firebase real | `firebase.config.ts` ganhou o bloco condicional |
| R-06 | `beach-center-app` — build | `npm run build` (`tsc -b && vite build`) continua passando | import novo (`connectAuthEmulator`) |
| R-07 | `beach-center-server` — produção | `nginx/beach-center.conf`, `process/pm2/ecosystem.config.cjs` e `scripts/update-all.sh` **intactos** — a task só **adicionou** artefatos de dev | Princípio I / IV |
| R-08 | `beach-center-server` — `/run-server` | A skill `/run-server` (que procura `docker-compose*.yml` + volumes + `.env.example`) agora encontra o `docker-compose.dev.yml` e opera sobre ele | a task materializa o que a skill espera |
| R-09 | Rodar isolado (todos os serviços) | `npm run dev` em cada BFF, um a um, fora do Docker | Nenhum `package.json` / `.env` / `nodemon` de repo foi alterado (exceto os 5 arquivos guardados por env em `usuarios` e 1 no `app`) |
| R-10 | Deploy de produção | O comando `ssh beach-center beach-center-update` não é afetado (não referencia nada de dev) | isolamento dev × prod |

---

## 6. Rastreabilidade AC → cenário

| AC | Descrição resumida | Cenários | Cobertura automatizada |
|---|---|---|---|
| **AC-1** | Subida completa com um comando | H-01, E-02, E-03 | não (exige Docker) |
| **AC-2** | Gateway espelha o roteamento de produção | H-02, H-03, B-14 | não |
| **AC-3** | Frontend servido pelo gateway com HMR | H-04, H-05, B-08 | não |
| **AC-4** | Hot-reload de um microsserviço | H-06, H-07, E-07, B-01 | não |
| **AC-5** | `node_modules` isolado do host | H-08, B-02, B-03 | não |
| **AC-6** | Dados do Mongo sobrevivem a `down`/`up` | H-09, B-10 | não |
| **AC-7** | Banco único compartilhado | H-10 | não |
| **AC-8** | Verificação de ID token contra o emulador | H-11, E-11, E-12 | **parcial** (`firebase.spec.ts` cobre o `initializeApp` sem cert; a validação real do token exige Docker) |
| **AC-9** | Login e reset de senha usam o emulador em dev | H-12, R-02 | **parcial** (`identity-toolkit.spec.ts` + 2 adapter specs cobrem a URL; o fluxo real exige o emulador) |
| **AC-10** | Login pelo frontend no ambiente Docker | H-13, R-05 | não (app sem runner de teste) |
| **AC-11** | Rodar um serviço isolado continua funcionando | H-14, R-09 | não |
| **AC-12** | `campeonatos` não quebra a subida | H-15, B-05 | não |
| **AC-13** | Repos tocados permanecem verdes | H-16, R-03, R-04, R-06 | **sim** (`/speckit-unit-tests`: 206/206, 99,5%) |
| **AC-14** | Compose e nginx válidos | H-17 | **parcial** (YAML + anchors validados no `/speckit-implement`; `docker compose config` e `nginx -t` exigem Docker) |

**Cenários sem equivalente automatizado (só manual):** todos os `H-*` exceto H-16; `E-01`..`E-10`,
`E-12`, `E-13`; todos os `B-*` exceto (parcialmente) nenhum; `R-01`, `R-02`, `R-05`..`R-10`.

---

## 7. Observações / dívidas conhecidas confirmadas pelos testes

1. **AC-1..AC-7, AC-10..AC-12, AC-14 não puderam ser verificados** nesta rodada — Docker não está
   instalado na máquina de desenvolvimento atual. A verificação real depende de subir o ambiente.
2. **Divergência de porta ao rodar 2+ BFFs isolados** sem passar `PORT` (B-07). Follow-up: alinhar
   os defaults de `env.ts` ao mapa canônico (agendamentos 5000 · pagamentos 5001 · usuarios 5002 ·
   aulas 5003 · whatsapp 5004).
3. **`beach-center-app` tem 36 erros de ESLint pré-existentes** (E-14) — bloqueiam o Quality Gate
   do `/speckit-complete` se aplicado ao repo inteiro. Decidir antes: corrigir à parte ou escopar.
4. **Latência de hot-reload no Windows** (B-01) por causa do polling — inerente ao Docker Desktop.
5. **Usuários do emulador não persistem** entre restarts do container do emulador (B-09).
6. **`nginx/beach-center.conf` de produção segue desatualizado** (sem `aulas`, portas divergentes)
   — deliberadamente fora do escopo; o gateway de dev usa `nginx/dev.conf`.
7. **Mudança de `package.json`** exige reinstalação manual das deps no volume (E-08) — documentado
   no README, mas é um passo fácil de esquecer.
