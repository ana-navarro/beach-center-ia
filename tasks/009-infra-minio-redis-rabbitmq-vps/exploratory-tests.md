# Roteiro de Testes Exploratórios — Task 009: Infraestrutura de produção (MinIO + Redis + RabbitMQ)

> Gerado por `/speckit-test`. Roteiro **manual** — este projeto não tem QA dedicado e esta task
> não produz código de aplicação (sem Jest/Cypress equivalente — ver `/speckit-unit-tests`,
> execução interrompida por decisão do usuário, e a nota de N/A no Constitution Check do
> `plan.md`, Princípio III). **Todos** os cenários abaixo exigem execução manual no VPS real —
> nenhum ambiente local com Docker está disponível nesta sessão (mesma limitação das tasks
> 007/008, decisão C4-escopo do `context.md`).

## Escopo e pré-condições

**Escopo:** apenas os arquivos de infraestrutura desta task, em `beach-center-server/`:
`docker-compose.prod.yml`, `.env.prod.example`, `.gitignore`, `scripts/prod-infra-up.sh`,
`scripts/prod-infra-down.sh`, `README.md`. Nenhum microsserviço Node é exercitado diretamente
(`injection`/`llm-engine` ainda não têm processo de deploy em produção — gap conhecido, fora do
escopo).

**Pré-condições gerais:**
- VPS Linux com **Docker Engine + Docker Compose v2** instalados (hoje ainda **não instalados** —
  primeiro pré-requisito a satisfazer antes de qualquer cenário).
- Repositório `beach-center-server` atualizado no VPS (`git pull` na branch que contém esta task).
- Acesso SSH ao VPS com permissão para rodar `docker`/`docker compose` (grupo `docker` ou `sudo`).
- Nenhum outro processo ocupando as portas `127.0.0.1:9000/9001/6379/5672/15672` no host.
- `.env.prod` **ainda não criado** no início do roteiro (para exercitar também o cenário de
  ausência — Exceção E1).
- Cliente `curl` disponível no VPS (ou na máquina local, via túnel SSH) para os testes de bucket
  e management UIs.
- Opcional: cliente `mc` (MinIO Client) local para os testes de política de bucket (E4).

## Caminhos felizes (rastreabilidade `AC-*`)

| ID | Pré-condição | Passos | Resultado esperado | AC |
|---|---|---|---|---|
| **HP-1** | Docker/Compose instalados; `.env.prod` ainda não existe. | 1. `cp .env.prod.example .env.prod`. 2. Preencher `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD`, `MINIO_BUCKET=comprovantes-pagamento`, `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS` com valores fortes (deixar `REDIS_PASSWORD` vazio). 3. Rodar `./scripts/prod-infra-up.sh`. | Os 4 serviços sobem; `minio-init` termina com `exit 0` (`docker compose ps` mostra `Exited (0)`); `minio`, `redis`, `rabbitmq` ficam `healthy` em até ~30s. Script imprime o bloco de URLs ao final. | **AC-1** |
| **HP-2** | Ambiente do HP-1 no ar. | 1. Fazer upload manual de um arquivo de teste ao bucket (via console MinIO em `http://127.0.0.1:9001` através de túnel SSH, ou `mc cp`). 2. `curl -i http://127.0.0.1:9000/comprovantes-pagamento/<arquivo>` sem credenciais. 3. Tentar `curl -X PUT http://127.0.0.1:9000/comprovantes-pagamento/outro-arquivo.txt -d "teste"` sem credenciais. | Passo 2 retorna **HTTP 200** com o conteúdo do arquivo (leitura pública confirmada). Passo 3 retorna **HTTP 403/AccessDenied** (sem escrita anônima). | **AC-2** |
| **HP-3** | Ambiente do HP-1 no ar; ao menos 1 objeto no bucket e 1 fila declarada no RabbitMQ (`rabbitmqadmin declare queue name=teste` ou via management UI). | 1. `docker compose -f docker-compose.prod.yml down` (sem `-v`). 2. `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d`. 3. Repetir o `curl` do HP-2 e conferir a fila `teste` no RabbitMQ management UI. | Objeto no MinIO e fila no RabbitMQ continuam presentes após o ciclo down/up — nenhum dado perdido. | **AC-3** |
| **HP-4** | Ambiente do HP-1 no ar; túnel SSH aberto para 9001 (`ssh -L 9001:127.0.0.1:9001 beach-center`). | 1. Acessar `http://127.0.0.1:9001` no navegador local. 2. Logar com `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`. | Console web carrega e autentica com sucesso; bucket `comprovantes-pagamento` visível na lista. | **AC-4** |
| **HP-5** | Ambiente do HP-1 no ar. | A partir de uma máquina **fora** do VPS (não localhost, não túnel): 1. `curl -m 5 http://<IP-publico-do-VPS>:9000`. 2. Repetir para as portas 9001, 6379, 5672, 15672. | Todas as tentativas dão **timeout/connection refused** — nenhuma porta responde externamente. | **AC-5** |
| **HP-6** | Ambiente do HP-1 no ar. | No VPS: `docker network inspect beach-center-network` (ou nome com prefixo do projeto, ex. `beach-center-prod-infra_beach-center-network`). | Os 4 containers (`minio`, `minio-init`, `redis`, `rabbitmq`) aparecem listados na mesma rede; nenhum está na rede `bridge`/`default` do host. | **AC-6** |
| **HP-7** | Ambiente do HP-1 no ar. | 1. `git status` / `git check-ignore -v .env.prod` no `beach-center-server`. 2. Conferir `git ls-files | grep .env.prod`. | `.env.prod` é ignorado pelo git (`check-ignore` aponta a regra no `.gitignore`); `.env.prod.example` aparece rastreado; `.env.prod` real **não** aparece em `git ls-files`. | **AC-7** |
| **HP-8** | Repositório com `docker-compose.dev.yml` (task 008) e `docker-compose.prod.yml` (esta task) lado a lado. | `grep MINIO_BUCKET docker-compose.dev.yml .env.dev.example docker-compose.prod.yml .env.prod.example`. | Dev mostra `comprovantes` (inalterado); prod mostra `comprovantes-pagamento`; nenhuma linha de dev foi alterada por esta task (`git diff` vazio nesses arquivos). | **AC-8** |
| **HP-9** | Ambiente do HP-1 no ar; túnel SSH para 15672. | 1. Acessar `http://127.0.0.1:15672`. 2. Logar com `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS`. | Management UI carrega e autentica com sucesso; dashboard exibe o node RabbitMQ ativo. | **AC-9** |

## Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **E1** | `.env.prod` **não existe** no diretório `beach-center-server`. | Rodar `./scripts/prod-infra-up.sh`. | Script **falha rapidamente** (exit 1) com mensagem clara pedindo `cp .env.prod.example .env.prod` — nenhum container é criado. |
| **E2** | `.env.prod` criado, mas `MINIO_ROOT_PASSWORD` alterado **depois** que `minio-init` já criou o alias com a senha antiga (ex.: editar `.env.prod` e reiniciar só o `minio-init`). | 1. Subir normalmente (HP-1). 2. Editar `MINIO_ROOT_PASSWORD` no `.env.prod`. 3. `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d minio-init` (sem recriar `minio`). | `minio-init` falha (`mc alias set` é rejeitado pelo MinIO, que ainda usa a senha antiga) — container termina com exit code != 0. Confirma que trocar credenciais exige recriar o `minio` também, não só o init (documentar como nota operacional, não é bug). |
| **E3** | Uma porta do host já ocupada (ex.: `sudo python3 -m http.server 9000` rodando antes). | Rodar `./scripts/prod-infra-up.sh`. | `docker compose up` falha ao publicar a porta 9000 com erro `port is already allocated`; os demais serviços (que não colidem) sobem normalmente ou o compose aborta a subida conforme comportamento padrão do Compose — nenhum crash silencioso. |
| **E4** | Ambiente do HP-1 no ar. | Tentar `mc ls` (list) anônimo no bucket: `curl -i "http://127.0.0.1:9000/comprovantes-pagamento?list-type=2"` sem credenciais. | Retorna **HTTP 403/AccessDenied** — a política `download` (leitura de objeto) não inclui `list`/`ListBucket` anônimo. |
| **E5** | Ambiente do HP-1 no ar, com `REDIS_PASSWORD` preenchido (não vazio) no `.env.prod`. | 1. Reconfigurar `.env.prod` com `REDIS_PASSWORD=uma-senha`. 2. Recriar o serviço `redis`: `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --force-recreate redis`. 3. `docker exec -it <container-redis> redis-cli ping` (sem `-a`). | Passo 3 retorna `NOAUTH Authentication required` (senha aplicada corretamente); com `redis-cli -a uma-senha --no-auth-warning ping` retorna `PONG`. Confirma que o `--requirepass` via `$REDIS_PASSWORD` funciona quando preenchido. |
| **E6** | Ambiente do HP-1 no ar. | Matar o processo do container MinIO abruptamente: `docker kill <container-minio>` (simula crash). | Por causa de `restart: unless-stopped`, o Docker recria o container automaticamente; após alguns segundos `docker compose ps` mostra `minio` novamente `healthy`, sem intervenção manual. |

## Edge cases

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **EC-1** | `.env.prod` com `REDIS_PASSWORD=` (vazio, valor default do `.env.prod.example`). | Subir o ambiente e rodar o healthcheck do Redis manualmente: `docker exec <container-redis> redis-cli ping`. | Retorna `PONG` sem necessidade de `-a` — confirma que `--requirepass ""` desabilita a exigência de senha corretamente quando a variável está vazia (comportamento esperado do Redis, não um bug). |
| **EC-2** | Ambiente do HP-1 no ar. | Rodar `docker compose -f docker-compose.prod.yml --env-file .env.prod up -d` **uma segunda vez** sem alterar nada (idempotência). | Nenhum erro; `minio-init` roda de novo (`mc mb --ignore-existing` e `mc anonymous set download` são idempotentes) e termina com sucesso; os demais serviços permanecem `Up`/`healthy` sem recriação desnecessária. |
| **EC-3** | Ambiente do HP-1 no ar, dados de teste gravados (objeto no MinIO, fila no RabbitMQ). | Rodar `./scripts/prod-infra-down.sh -v`. | Todos os volumes nomeados (`minio_prod_data`, `redis_prod_data`, `rabbitmq_prod_data`) são removidos junto com os containers; ao subir de novo, o bucket e a fila **não** existem mais (perda de dados esperada e documentada no `README.md` — este é o único fluxo destrutivo do roteiro). |
| **EC-4** | VPS reiniciado (reboot) com Docker configurado para iniciar no boot (`systemctl enable docker`), containers no ar antes do reboot. | Reiniciar o VPS (`sudo reboot`) e aguardar a volta. | Os 3 serviços de longa duração (`minio`, `redis`, `rabbitmq`) voltam automaticamente (`restart: unless-stopped` + Docker daemon habilitado no boot); `minio-init` **não** re-executa automaticamente no boot (política `restart: "no"`) — comportamento esperado, pois o bucket já foi criado antes. |
| **EC-5** | `MINIO_BUCKET` no `.env.prod` deixado em branco por engano. | Subir o ambiente com essa configuração. | `minio-init` falha ao rodar `mc mb --ignore-existing local/""` (nome de bucket vazio é inválido) — falha visível nos logs, não um bucket silenciosamente errado. Confirma a importância de preencher `MINIO_BUCKET` explicitamente (nota já destacada no `README.md`). |

## Checklist de regressão

| ID | Área afetada | Passos | Resultado esperado |
|---|---|---|---|
| **R-1** | Ambiente de desenvolvimento (`docker-compose.dev.yml`) | Rodar `./scripts/dev-up.sh` normalmente (ambiente local de dev, tasks 007/008). | Sobe exatamente como antes desta task — nenhum serviço, variável ou porta de dev foi alterado; `git diff` em `docker-compose.dev.yml` e `.env.dev.example` está vazio. |
| **R-2** | Bucket de dev (`comprovantes`) | Com o ambiente de dev no ar, verificar que o bucket `comprovantes` (dev) continua distinto e não é afetado por operações no bucket `comprovantes-pagamento` (prod). | Os dois buckets vivem em instâncias MinIO **separadas** (containers/volumes diferentes, dev local vs. prod VPS) — nenhuma interferência possível; nomes diferentes reduzem risco de confusão operacional. |
| **R-3** | Scripts de dev (`dev-up.sh`/`dev-down.sh`/`dev-logs.sh`) | Executar os três normalmente. | Comportamento inalterado — nenhum desses arquivos foi tocado por esta task. |
| **R-4** | `.gitignore` | Conferir que `.env.dev` continua sendo ignorado (regra pré-existente) além da nova regra `.env.prod`. | Ambas as regras coexistem; `git check-ignore -v .env.dev` e `.env.prod` apontam para linhas distintas do `.gitignore`. |
| **R-5** | `README.md` — seção "Produção (NGINX + PM2 + SSH)" | Revisar visualmente o `README.md` completo após a inserção da nova seção. | A seção pré-existente de deploy via PM2/SSH permanece íntegra, na ordem original, sem conteúdo cortado ou duplicado pela inserção da nova seção "Infraestrutura de apoio". |

## Cobertura `AC-* → cenário`

| AC | Cenário(s) | Cobertura automatizada equivalente |
|---|---|---|
| AC-1 | HP-1 | Nenhuma — task de infraestrutura pura, sem Jest/Cypress (Princípio III N/A, ver `plan.md`) |
| AC-2 | HP-2, E4 | Nenhuma |
| AC-3 | HP-3, EC-3 (variante destrutiva) | Nenhuma |
| AC-4 | HP-4 | Nenhuma |
| AC-5 | HP-5 | Nenhuma |
| AC-6 | HP-6 | Nenhuma |
| AC-7 | HP-7 | Nenhuma |
| AC-8 | HP-8, R-2 | Nenhuma |
| AC-9 | HP-9 | Nenhuma |

Todos os 9 Critérios de Aceite têm ao menos um cenário de caminho feliz correspondente. Como esta
task não produz código de aplicação, **não há suíte automatizada** (Jest/Cypress) a comparar — a
validação da task depende inteiramente da execução manual deste roteiro no VPS real.
