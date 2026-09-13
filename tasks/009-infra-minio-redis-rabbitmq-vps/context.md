# Task 009 — Infraestrutura de produção: MinIO + Redis/RabbitMQ no VPS

> Gerado por `/speckit-task`. Não implementa nada. Base para `/speckit-plan`.
> **Perguntas bloqueadoras em aberto** — ver seção "Perguntas em aberto". Depende, em espírito,
> das tasks [[task-007-constitution-injection-llm-engine]] e
> [[task-008-desacoplamento-upload-comprovante]] (que já provisionaram o equivalente em
> **desenvolvimento**).

## Título

Provisionar, via Docker, os serviços de infraestrutura de apoio para a nova arquitetura de
validação de comprovantes **no ambiente de produção (VPS)**: MinIO (storage S3-compatible),
Redis (cache) e/ou RabbitMQ (mensageria), com persistência, rede isolada e credenciais
externalizadas em `.env`.

## Contexto de negócio e técnico

- Objetivo de fundo (task 007/008): abandonar o Google Drive (rate limit) e permitir validação
  assíncrona de comprovante por IA sem travar o usuário.
- **Diferença crucial em relação à 008**: a 008 já subiu `minio` + `minio-init` + `rabbitmq` +
  `injection` no **`beach-center-server/docker-compose.dev.yml`** — isso é o ambiente de
  **desenvolvimento local** (simula o servidor real, mas roda na máquina do dev). **Esta task
  (009) é sobre o ambiente de PRODUÇÃO**, que hoje **não usa Docker** — ver achado abaixo.

### Achado importante sobre a produção atual (`beach-center-server/README.md`)

A produção do Beach Center **não é conteinerizada**. É bare-metal: **NGINX + Node/PM2** direto no
VPS, atualizado por um comando SSH (`ssh beach-center beach-center-update` → `git pull` + `npm ci`
+ build do front + reload do PM2/NGINX). Estrutura no servidor:
`/var/www/beach-center/{beach-center-app,beach-center-bff-agendamentos,beach-center-bff-pagamentos,
beach-center-bff-usuarios,beach-center-server,beach-center-whatsapp}`. **Não há nenhum
`docker-compose` de produção hoje** — só o de desenvolvimento (`docker-compose.dev.yml`). O
próprio `README.md` já lista como dívida conhecida que `nginx/beach-center.conf` (produção) está
desatualizado.

Isso significa que **esta task introduziria a primeira peça de Docker em produção** — e cria uma
pergunta arquitetural real: os microsserviços Node (`agendamentos`, `pagamentos`, `injection`,
`llm-engine`...) continuam rodando via PM2 **fora** do Docker, então uma rede Docker isolada
(`beach-center-network`) **não** os alcança automaticamente — processos PM2 não estão nela. Para
o `injection`/`llm-engine` (rodando via PM2, fora de containers) falarem com o MinIO/Redis/
RabbitMQ (dentro de containers), a comunicação teria que ser via **porta publicada no host**
(`127.0.0.1:9000`, `127.0.0.1:6379`, `127.0.0.1:5672`), não via rede Docker interna — a menos que
se decida também dockerizar os serviços Node (fora do escopo desta task, que é só infra de apoio).

### O que a US pede, literalmente

1. **MinIO**: imagem `minio/minio` mais recente, `server /data --console-address ":9001"`,
   `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`, volume persistente em `/data`, portas 9000 (API) e
   9001 (console), bucket **`comprovantes-pagamento`** com política de **leitura pública**.
2. **Redis**: `redis:alpine`, "padrão obrigatório para Cache", volume opcional, porta 6379.
3. **RabbitMQ** (rotulado "opcional, se definido pelo time"): `rabbitmq:3-management-alpine`,
   `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS`, portas 5672/15672, volume em
   `/var/lib/rabbitmq`.
4. Rede Docker dedicada `beach-center-network`.
5. `.env`/`.env.example` com as credenciais (real não commitado).

### Tensão com decisões já tomadas (task 008 / Princípio VI)

- **Bucket**: task 008 já fixou `comprovantes` (dev) como o nome do bucket, usado como default no
  `injection` (`MINIO_BUCKET`) e no `pagamentos` (`MINIO_BUCKET`). Esta US pede
  **`comprovantes-pagamento`** — nome diferente. Como os dois lêem o nome via env (`MINIO_BUCKET`),
  tecnicamente não há conflito de código — mas um nome diferente por ambiente é um risco
  operacional (esquecer de configurar a env em prod usaria o default errado).
- **Mensageria**: a task 007 (Constitution, Princípio VI) e a 008 (`/speckit-plan`, decisão DP9)
  **já escolheram RabbitMQ** (`amqplib`) como a fila entre `injection` e `llm-engine`, com fila
  `comprovante.validar`. O `llm-engine` (US03, ainda não implementado) presumivelmente consumirá
  dessa fila. Esta US pede Redis como "padrão obrigatório para Cache" — um uso **diferente** do
  Redis (cache de consultas, não fila) — mas também insinua "RabbitMQ **ou** Redis/BullMQ" como se
  a escolha de mensageria ainda estivesse em aberto, o que **não é mais o caso**.
- **Leitura pública do bucket**: a 008 já fez isso no `minio-init` de dev
  (`mc anonymous set download`) — mesma abordagem que esta US pede para prod.

### O que já existe (reaproveitável) — de dev, para não recriar do zero

- `beach-center-server/docker-compose.dev.yml`: serviços `minio` (`minio/minio:RELEASE.2025-04-22T22-12-26Z`),
  `minio-init` (`minio/mc`, cria bucket + política pública), `rabbitmq` (`rabbitmq:3.13-management-alpine`).
- `beach-center-server/.env.dev.example`: já documenta `MINIO_*`/`RABBITMQ_*`/`INJECTION_*`.

## Serviço(s) alvo (Princípio I)

| Repositório | Papel | Justificativa |
|---|---|---|
| **`beach-center-server`** | Único alvo — é o repositório de infraestrutura do ecossistema ("Configurações de gateway, orquestração de containers e scripts de infraestrutura" — Princípio I). Novo arquivo `docker-compose.prod.yml` (nome a confirmar) + `.env.prod.example` (ou equivalente) + doc no `README.md` + eventual script de bootstrap do bucket. | — |
| ~~Demais serviços~~ | **Não tocados** — task é só infraestrutura de apoio (MinIO/Redis/RabbitMQ), não código de aplicação. | — |

## Regras de negócio conhecidas

1. Volumes persistentes: dados do MinIO e da fila **não podem** ser perdidos em
   `docker compose down && docker compose up -d`.
2. MinIO: bucket `comprovantes-pagamento` com leitura pública (só leitura — sem escrita/list
   anônimos).
3. Console do MinIO acessível via porta 9001.
4. Redis obrigatório (cache); RabbitMQ conforme a escolha já feita nas tasks 007/008.
5. Rede Docker dedicada para os serviços que estiverem em containers se comunicarem sem expor
   portas desnecessárias à internet.
6. Credenciais só em `.env` (não commitado); `.env.example` versionado.

## Impacto arquitetural previsto

Não há camadas hexagonais aqui (task de infraestrutura pura, sem código de aplicação). Arquivos
previstos, todos em `beach-center-server/`:

- `docker-compose.prod.yml` (C1) — serviços `minio`, `minio-init` (bucket `comprovantes-pagamento`
  com leitura pública), `redis`, `rabbitmq`, rede `beach-center-network`, volumes nomeados,
  portas publicadas só em `127.0.0.1` (C2).
- `.env.prod.example` — todas as credenciais/portas (nome a confirmar no plano: `.env.prod.example`
  vs. `.env.example` dedicado a este compose).
- `.gitignore` — garantir que o `.env` real de produção não seja commitado (hoje só ignora
  `.env.dev`).
- `README.md` — nova seção "Infraestrutura de apoio (MinIO/Redis/RabbitMQ) em produção": como
  subir, como criar o bucket, como as portas ficam expostas (loopback vs. rede Docker), como o
  PM2 (fora do Docker) se conecta.
- Possível script `scripts/minio-init.sh` ou reaproveito do container `minio/mc` (como já existe
  no dev) para o bootstrap do bucket.

## Decisões confirmadas (rodada 1 — `/speckit-task`)

| # | Pergunta | Decisão |
|---|---|---|
| **C1** | Onde vive o compose | **Novo `docker-compose.prod.yml`** dentro de `beach-center-server`, arquivo irmão do `docker-compose.dev.yml`. Contém **só** MinIO/Redis/RabbitMQ (+ init do bucket) — os microsserviços Node continuam via PM2, fora dele. |
| **C2** | Rede PM2 ↔ Docker | **Portas publicadas só em `127.0.0.1`** (`127.0.0.1:9000`/`9001`/`6379`/`5672`/`15672`). Os processos PM2 no mesmo host acessam via `localhost`; a internet nunca alcança essas portas. A `beach-center-network` (Docker) isola os containers **entre si**; o acesso PM2→container é via host (loopback), não via rede Docker. |
| **C3** | Nome do bucket | **Nomes diferentes por ambiente, via env** — dev continua `comprovantes` (já em uso desde a task 008); produção usa **`comprovantes-pagamento`** (conforme a US). Nenhum código muda (`MINIO_BUCKET` já é configurável); documentar com destaque para não esquecer de setar em prod. |
| **C4-escopo** | O que a entrega cobre | **Só IaC + documentação + roteiro manual.** Este ambiente não tem VPS real nem Docker local (mesma limitação das tasks 007/008). Entrego `docker-compose.prod.yml` + `.env.prod.example` + atualização do `README.md` + roteiro de verificação manual (`/speckit-test`) para rodar no VPS de verdade. Nenhum container é executado/validado por mim. |

## Perguntas menores (default proposto — não bloqueiam)

- **Q4 — Redis sem consumidor ainda.** O DoD é 100% infra (containers no ar, persistência,
  bucket acessível) — **nenhum item pede alteração de código**. Default: Redis sobe **disponível
  para uso futuro** (cache de consultas — sem consumidor nesta task). A fila `comprovante.validar`
  **continua em RabbitMQ** (decisão já tomada nas tasks 007/008) — Redis **não** a substitui.

## Notas

- Esta task **não** duplica a 008: reaproveita os *padrões* já validados lá (imagens, comando do
  `mc`, política pública) mas para um arquivo/ambiente **novo** (produção), com nomes/escopo
  próprios.
- `beach-center-bff-injection` e `beach-center-bff-llm-engine` ainda **não têm processo de deploy
  em produção** (não estão na lista de pastas do `/var/www/beach-center/` nem no script
  `update-all.sh`) — isso é gap conhecido, mas **fora do escopo** desta task (só infra de apoio).

## Próximo passo

Perguntas bloqueadoras (C1–C4) **fechadas**. Rodar **`/speckit-plan`**.
