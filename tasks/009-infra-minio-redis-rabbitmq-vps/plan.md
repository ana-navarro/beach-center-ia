# Plan — Task 009: Infraestrutura de produção (MinIO + Redis + RabbitMQ no VPS)

> Gerado por `/speckit-plan`. Não implementa nada. Base para `/speckit-implement`.
> Task de infraestrutura pura (IaC + documentação) — **sem código de aplicação**, portanto
> **sem Arquitetura Hexagonal** (Princípio II não se aplica; ver Constitution Check).

## Contexto Técnico

**Serviço(s) alvo (Princípio I):** único alvo é `beach-center-server` — repositório definido pela
constituição como dono de "configurações de gateway, orquestração de containers e scripts de
infraestrutura". Nenhum outro repositório é tocado: esta task não altera código de aplicação dos
BFFs (`injection`, `llm-engine`, `agendamentos`, `pagamentos`), apenas a infraestrutura de apoio
que eles consumirão via variáveis de ambiente (fora do escopo desta task apontar essas envs em
produção real, já que os serviços Node ainda não têm processo de deploy — ver `context.md`, seção
"Notas").

**Diferença em relação ao `docker-compose.dev.yml` (task 008):** aquele arquivo sobe o ecossistema
inteiro (Mongo, Firebase Emulator, todos os BFFs, frontend, nginx) para simular o servidor num
único host de desenvolvimento. Este `docker-compose.prod.yml` sobe **apenas** a infraestrutura de
apoio (MinIO, Redis, RabbitMQ) que vai rodar ao lado dos processos PM2 já existentes no VPS — os
microsserviços Node continuam fora do Docker (decisão C1/C2, `context.md`).

**Stack e bibliotecas:**
- Docker Compose (schema `docker-compose.yml`, sem `version:` — mesmo padrão do `docker-compose.dev.yml`).
- `minio/minio` (mesma tag pinada do dev, `RELEASE.2025-04-22T22-12-26Z`, para evitar drift de
  comportamento entre ambientes) + `minio/mc` (`RELEASE.2025-04-16T18-13-26Z`) para o bootstrap do
  bucket via `minio-init` (reaproveita o padrão do dev — `context.md`, "O que já existe").
- `redis:alpine` (US pede explicitamente esta tag, não uma versão pinada).
- `rabbitmq:3-management-alpine` (US pede esta tag; dev usa `3.13-management-alpine` — manter a
  tag pedida pela US aqui, já que produção pode receber patches de segurança automáticos da faixa
  `3-management-alpine`, enquanto dev fixa uma versão exata para reprodutibilidade local — não é
  uma inconsistência a corrigir, é uma escolha válida por ambiente).
- `.env` para credenciais (não commitado); `.env.prod.example` versionado com placeholders.

## Constitution Check

| Princípio | Status | Justificativa |
|---|---|---|
| **I — Fronteiras** | CONFORME | Único arquivo tocado pertence a `beach-center-server`, dono declarado da infraestrutura. Nenhum código de `injection`/`llm-engine`/`agendamentos`/`pagamentos` é criado ou alterado. |
| **II — Hexagonal** | N/A (não se aplica) | Task de infraestrutura pura (Docker Compose + docs), sem código de aplicação TypeScript/Node. O Princípio II rege apenas os micro-serviços Node — nenhum é criado ou modificado aqui. |
| **III — Test-First/Qualidade** | N/A (não se aplica) | Sem código de aplicação, não há ESLint/Jest/cobertura a medir. A verificação desta task é o roteiro manual gerado por `/speckit-test` (rodar no VPS real), não testes automatizados. |
| **IV — Infra Reproduzível com Hot-Reload** | CONFORME (com nota) | O Princípio IV fala de hot-reload em `beach-center-server` para *desenvolvimento local* — já satisfeito pelo `docker-compose.dev.yml` existente. `docker-compose.prod.yml` é infraestrutura de produção (sem bind-mount de código, sem hot-reload — não se aplica a containers de storage/mensageria de terceiros). Persistência via volumes nomeados é a garantia equivalente exigida para produção (regra de negócio 1 do `context.md`). |
| **VI — Validação assíncrona de comprovantes** | CONFORME | Esta task não implementa o fluxo do Princípio VI, mas provisiona duas das suas dependências de infraestrutura em produção (MinIO como storage, RabbitMQ como fila `comprovante.validar`), sem alterar o desenho do fluxo já normatizado. Bucket com leitura pública replica a mesma política já usada em dev (Princípio VI, item 4). |

**Nenhuma violação sem justificativa.** Prosseguir.

## Mapa Arquitetural (Hexagonal)

Não se aplica — task sem código de aplicação (ver Constitution Check, Princípio II). Não há
`controllers/`, `routes/`, `domain/`, `infra/adapters/` etc. a criar. Mapa de arquivos abaixo é
estrutural/infra, não hexagonal.

## Checklist de Implementação

Ordem de dependência: exemplo de env → compose → gitignore → script de verificação (opcional) →
documentação.

- [x] `beach-center-server/.env.prod.example` — placeholders para todas as variáveis usadas pelo
      `docker-compose.prod.yml`: `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `MINIO_BUCKET`
      (default documentado como `comprovantes-pagamento`), `REDIS_PASSWORD` (opcional, vazio por
      padrão), `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`. Comentários indicando que o
      arquivo real (`.env.prod`) nunca é commitado.
- [x] `beach-center-server/docker-compose.prod.yml` — serviços:
  - `minio` (imagem `minio/minio:RELEASE.2025-04-22T22-12-26Z`, comando
    `server /data --console-address ":9001"`, env `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` via
    `${...}` do `.env.prod`, portas `127.0.0.1:9000:9000` e `127.0.0.1:9001:9001`, volume nomeado
    `minio_prod_data:/data`, healthcheck `mc ready local`, `restart: unless-stopped`).
  - `minio-init` (imagem `minio/mc:RELEASE.2025-04-16T18-13-26Z`, `depends_on: minio` com
    `condition: service_healthy`, entrypoint que faz `mc alias set` + `mc mb --ignore-existing
    local/comprovantes-pagamento` + `mc anonymous set download local/comprovantes-pagamento`
    (leitura pública, sem escrita/list anônimos — regra de negócio 2), `restart: "no"`).
  - `redis` (imagem `redis:alpine`, porta `127.0.0.1:6379:6379`, volume nomeado
    `redis_prod_data:/data` (persistência opcional via `--save`/AOF — default de imagem já
    persiste snapshots em `/data`), `restart: unless-stopped`, healthcheck
    `redis-cli ping`).
  - `rabbitmq` (imagem `rabbitmq:3-management-alpine`, env
    `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS` via `.env.prod`, portas
    `127.0.0.1:5672:5672` e `127.0.0.1:15672:15672`, volume nomeado
    `rabbitmq_prod_data:/var/lib/rabbitmq`, `restart: unless-stopped`, healthcheck
    `rabbitmq-diagnostics -q ping`).
  - Rede dedicada `beach-center-network` (bridge, atribuída a todos os serviços acima) — isola os
    containers entre si; não é o canal de acesso do PM2 (regra de negócio 5 / decisão C2).
  - `volumes:` nomeados: `minio_prod_data`, `redis_prod_data`, `rabbitmq_prod_data`.
- [x] `beach-center-server/.gitignore` — adicionar `.env.prod` (hoje só ignora `.env.dev`) para
      garantir que credenciais reais de produção nunca sejam commitadas.
- [x] `beach-center-server/scripts/prod-infra-up.sh` — wrapper fino, espelhando
      `scripts/dev-up.sh`: `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d`.
- [x] `beach-center-server/scripts/prod-infra-down.sh` — wrapper fino, espelhando
      `scripts/dev-down.sh` (com suporte a `-v` para apagar volumes, uso deliberado/raro em
      produção — documentar o risco no `README.md`).
- [x] `beach-center-server/README.md` — nova seção "Infraestrutura de apoio (MinIO/Redis/RabbitMQ)
      em produção", cobrindo: pré-requisito de Docker no VPS (ainda não instalado — achado do
      `context.md`), como copiar `.env.prod.example` → `.env.prod` e preencher credenciais fortes,
      como subir (`./scripts/prod-infra-up.sh`), como as portas ficam expostas só em
      `127.0.0.1` e como os processos PM2 (fora do Docker) as acessam via `localhost` (decisão
      C2), nome do bucket em produção (`comprovantes-pagamento`, diferente do dev — decisão C3,
      com destaque para não usar o default de dev por engano), e nota explícita de que
      `injection`/`llm-engine` ainda não têm processo de deploy em produção (gap conhecido, fora
      do escopo desta task).

## Critérios de Aceite (formais)

- **AC-1 (subida limpa)**
  Given o VPS com Docker/Docker Compose instalados e `.env.prod` preenchido a partir do
  `.env.prod.example`,
  When o operador executa `./scripts/prod-infra-up.sh` (ou
  `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d`),
  Then os quatro serviços (`minio`, `minio-init`, `redis`, `rabbitmq`) sobem, `minio-init`
  termina com sucesso (`service_completed_successfully`) e `minio`/`redis`/`rabbitmq` reportam
  `healthy`.

- **AC-2 (bucket criado com leitura pública)**
  Given os containers `minio` e `minio-init` no ar,
  When o operador acessa `http://127.0.0.1:9000/comprovantes-pagamento/<algum-objeto-de-teste>`
  sem credenciais (após um upload manual de teste),
  Then o arquivo é servido (HTTP 200) sem autenticação — confirmando leitura pública — e uma
  tentativa de `PUT`/`mc ls` anônimo no bucket é rejeitada (sem escrita/list anônimos).

- **AC-3 (persistência de dados)**
  Given o MinIO com ao menos um objeto gravado no bucket `comprovantes-pagamento` e o RabbitMQ com
  ao menos uma fila declarada,
  When o operador executa `docker compose -f docker-compose.prod.yml down` seguido de
  `docker compose -f docker-compose.prod.yml up -d` (sem `-v`),
  Then o objeto no MinIO e a fila no RabbitMQ continuam presentes após a subida (volumes nomeados
  preservados).

- **AC-4 (console do MinIO acessível)**
  Given o serviço `minio` saudável,
  When o operador acessa `http://127.0.0.1:9001` a partir do próprio VPS (ou via túnel SSH),
  Then o console web do MinIO responde e aceita login com `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`.

- **AC-5 (portas não expostas à internet)**
  Given o `docker-compose.prod.yml` aplicado no VPS,
  When o operador inspeciona as portas publicadas (`docker compose -f docker-compose.prod.yml ps`
  ou `ss -tlnp`) a partir de **fora** do VPS,
  Then nenhuma das portas 9000/9001/6379/5672/15672 responde externamente — todas publicadas
  apenas em `127.0.0.1` (decisão C2).

- **AC-6 (rede Docker isolada entre containers)**
  Given os quatro serviços no ar,
  When o operador inspeciona a rede Docker (`docker network inspect beach-center-network`),
  Then `minio`, `minio-init`, `redis` e `rabbitmq` estão todos na mesma rede dedicada
  `beach-center-network` (não na rede `default` do host).

- **AC-7 (credenciais não commitadas)**
  Given o repositório `beach-center-server` após a implementação desta task,
  When o operador executa `git status` / revisa o `.gitignore`,
  Then `.env.prod` está listado no `.gitignore` e não aparece como arquivo rastreado, enquanto
  `.env.prod.example` (só placeholders) está versionado.

- **AC-8 (nome de bucket correto por ambiente — regressão)**
  Given o `docker-compose.dev.yml` (task 008) e o `docker-compose.prod.yml` (esta task) no mesmo
  repositório,
  When o operador compara os valores de bucket em cada um,
  Then o dev continua usando `comprovantes` (inalterado) e o prod usa `comprovantes-pagamento` —
  nenhum arquivo de dev foi modificado por esta task.

- **AC-9 (RabbitMQ com plugin de management ativo)**
  Given o serviço `rabbitmq` no ar,
  When o operador acessa `http://127.0.0.1:15672` a partir do VPS (ou túnel SSH) com
  `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS`,
  Then a UI de management do RabbitMQ carrega e autentica com sucesso.

## Não avançar

Este plano não implementa nada. Próximo passo do Ciclo Speckt: `/speckit-implement`.
