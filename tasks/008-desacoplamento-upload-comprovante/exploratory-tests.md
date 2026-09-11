# Testes exploratórios — Task 008 (US02): Desacoplamento do upload de comprovante

> Gerado por `/speckit-test`. Roteiro **manual**, para rodar com o ecossistema no ar
> (`docker compose -f beach-center-server/docker-compose.dev.yml up -d --build`, que agora inclui
> `minio`, `minio-init` e `rabbitmq` — task 008). Cobre os 12 Critérios de Aceite do `plan.md`
> (AC-1…AC-12). Nenhum cenário aqui tem cobertura automatizada por definição (é o objetivo do
> `/speckit-test`); a coluna "Automação" marca o equivalente já coberto em `/speckit-unit-tests`.

## Escopo e pré-condições

| Item | Valor |
|---|---|
| **Serviços** | `beach-center-bff-injection` (novo, porta 5006), `beach-center-bff-pagamentos` (5001), `beach-center-bff-agendamentos` (5000), `minio` (9000 API / 9001 console), `rabbitmq` (5672 AMQP / 15672 management), `mongo`, `firebase-emulator` |
| **Front** | `beach-center-app` — **não alterado**; usar a tela de criação de reserva (`CreateReserveForm`) e o painel `TransactionHistory` como estão hoje |
| **Seed** | 1 reserva `pending` com 1–3 `scheduling_id` válidos (via `POST /reservas` autenticado ou fluxo do front); 1 usuário `ADMIN` (Firebase Emulator) para o painel de revisão |
| **Console MinIO** | `http://localhost:9001` (usuário/senha `minioadmin`) — conferir o bucket `comprovantes` e os objetos subidos |
| **Management RabbitMQ** | `http://localhost:15672` (`guest`/`guest`) — conferir a fila `comprovante.validar` e as mensagens publicadas (sem consumer ainda — `llm-engine` é US03+, as mensagens ficam na fila) |
| **Chaves internas (dev)** | `dev-injection-key` (`pagamentos`→`injection`), `dev-agendamentos-key` (`injection`/`pagamentos`→`agendamentos`) — já no `docker-compose.dev.yml` |
| **curl direto no injection** | `curl -X POST http://localhost:5006/api/v1/agendamentos/comprovantes -H "x-api-key: dev-injection-key" -H "Content-Type: application/json" -d '{...}'` — útil para os cenários de exceção sem passar pelo front |

## 1. Caminhos felizes — por Critério de Aceite

| ID | Pré-condição | Passos | Resultado esperado | Automação |
|---|---|---|---|---|
| **H-1 (AC-1)** | Repo `beach-center-bff-pagamentos` no editor | Abrir `src/` e procurar por `multer`, `busboy`, `googleapis`, `google-drive`, `IFileStoragePort`, coleção `manual_payments` | Nada encontrado fora de comentários históricos; `POST /transaction-history` é um proxy (sem lógica de arquivo) | — |
| **H-2 (AC-2)** | `injection` no ar | `curl` no `POST /agendamentos/comprovantes` com `x-api-key` válida e `proof_file` em PDF de 2 MB | `202` (segue para as validações de negócio — não é rejeitado por formato/tamanho) | `static-file-validation.middleware.spec.ts` |
| **H-3 (AC-3)** | Reserva `pending` criada; comprovante válido enviado (front ou curl) | Após o `202`, abrir o console do MinIO → bucket `comprovantes` | Existe um objeto novo `‹uuid›.pdf` (ou `.png`/`.jpg`), tamanho e conteúdo batendo com o enviado | `put-object.adapter.spec.ts` |
| **H-4 (AC-4)** | idem | `GET /reservas/:id` no `agendamentos` (ou painel admin) logo após o envio | `status = "waiting_approve"`, `number` preenchido (protocolo), `proof_key = ‹uuid›.‹ext›` | `inject-comprovante.usecase.spec.ts` + `update-payment-metadata.controller.spec.ts` |
| **H-5 (AC-5)** | idem | Abrir o RabbitMQ management → fila `comprovante.validar` → "Get messages" | 1 mensagem nova, JSON com `agendamento_id`, `usuario_id`, `email`, `file_url`, `proof_key`, `bucket`, `mime_type`, `timestamp` — `file_url` resolve (abre o objeto no navegador) | `publish-validation-event.adapter.spec.ts` |
| **H-6 (AC-5)** | idem | Medir o tempo de resposta do `POST /transaction-history` no front (Network tab) | Resposta rápida (não espera nenhum processamento de IA — não existe IA ainda, mas a latência é a do upload+2 PATCH+publish, não de uma "análise") | usecase (ordem de chamadas) |
| **H-7 (AC-6)** | Nenhuma | `git status` em `beach-center-app` antes/depois da task | Sem diferenças; o formulário de criação de reserva continua enviando `proof_file.base64` do mesmo jeito | verificado no implement/validate |
| **H-8 (AC-7)** | Reserva pública (via link) `pending` | Cliente abre o link público, preenche o formulário e envia o comprovante sem estar logado | `202`; reserva vai para `waiting_approve` do mesmo jeito (autorização por `public_reserve_token`) | usecase (`byPublicLink`) |
| **H-9 (AC-9)** | Reserva em `waiting_approve` com `proof_key` | Login como ADMIN → `TransactionHistoryPage` | A reserva aparece na lista; "Abrir comprovante" redireciona para a URL do MinIO e exibe o arquivo | `query-manual-payments.usecase.spec.ts`, `transaction-history.controller.spec.ts` |
| **H-10 (AC-9)** | idem | ADMIN clica "Aprovar" | Reserva vira `approved`, `payment_method = PIX`; comprovante some da lista de pendentes | `review-manual-payment.usecase.spec.ts` |
| **H-11 (AC-9)** | Reserva `waiting_approve` | ADMIN clica "Rejeitar" com uma nota | Reserva vira `rejected`; `review_note` gravado na reserva (`GET /reservas/:id` mostra o campo) | idem |
| **H-12 (AC-11)** | `docker compose up` | Subir a stack do zero | `minio`, `minio-init` (roda e sai com sucesso), `rabbitmq` e `injection` sobem saudáveis; `injection` não aparece em nenhuma rota do `nginx`/gateway público | inspeção do `docker-compose.dev.yml` (YAML) |

## 2. Fluxos de exceção

| ID | Pré-condição | Passos | Resultado esperado | Automação |
|---|---|---|---|---|
| **E-1 (AC-2)** | `injection` no ar | `proof_file.mime_type = "image/gif"` | `400` — "Formato de comprovante invalido. Aceitos: PDF, JPG ou PNG"; nada sobe no MinIO | `static-file-validation.middleware.spec.ts` |
| **E-2 (AC-2)** | idem | `base64` de um arquivo de 6 MB | `400` — "O comprovante deve ter no maximo 5 MB" | idem |
| **E-3 (AC-2)** | idem | Requisição sem `x-api-key` (direto no `injection`, não pelo proxy) | `401 Unauthorized` | `internal-api-key.middleware.spec.ts` |
| **E-4 (AC-7)** | Reserva já `waiting_approve` (comprovante já enviado antes) | Reenviar comprovante para a mesma reserva | `409` — "A reserva nao esta pendente de pagamento"; nenhum novo objeto no MinIO, nenhuma mensagem nova na fila | usecase (`ConflictError`) |
| **E-5 (AC-7)** | Reserva de outro cliente | Enviar comprovante autenticado com e-mail diferente do dono da reserva, sem `public_reserve_token` | `403` — "Acesso negado para esta reserva" | usecase (`ForbiddenError`) |
| **E-6 (AC-7)** | Link público expirado/inválido | Enviar com `public_reserve_token` inválido | `403` | `check-public-link.adapter` + usecase |
| **E-7 (AC-7)** | Reserva de 1 slot (R$ 80) | Enviar `amount: 120` (valor de 2 slots) | `400` — "Os dados do pagamento nao correspondem a reserva" | usecase (reconciliação) |
| **E-8 (AC-7)** | `reserve_id` inexistente | `curl` com um ObjectId que não existe | `404` — "Reserva nao encontrada" | usecase |
| **E-9 (AC-8)** | `agendamentos` fora do ar (parar o container) | Enviar um comprovante válido | Upload chega a subir no MinIO, mas o `PATCH /reservas/:id/status` falha → `injection` **remove o objeto do MinIO** (best-effort) e responde `502`; conferir no console do MinIO que o objeto não ficou órfão | `inject-comprovante.usecase.spec.ts` (compensação) |
| **E-10 (AC-8)** | `rabbitmq` fora do ar (parar o container) | Enviar um comprovante válido | Reserva chega a ir para `waiting_approve` e depois é **revertida para `pending`** (mesmo `number`); objeto removido do MinIO; resposta `502` | idem |
| **E-11 (AC-9)** | Reserva `pending` (sem comprovante) | Tentar acessar `GET /transaction-history/:id/proof` para essa reserva | `404` — "Arquivo do comprovante nao encontrado" (sem `proof_key`) | `query-manual-payments.usecase.spec.ts` |
| **E-12 (AC-9)** | Reserva já `approved` | ADMIN tenta aprovar de novo | `409` — "Este comprovante ja foi revisado" | `review-manual-payment.usecase.spec.ts` |
| **E-13** | `minio` fora do ar | Enviar um comprovante válido | `PutObjectCommand` falha → `502` antes de qualquer chamada ao `agendamentos` (reserva continua `pending`) | `put-object.adapter.spec.ts` |
| **E-14** | Corpo malformado (`proof_file` ausente) | `POST` sem o campo `proof_file` | `400` do middleware de validação estática (antes mesmo do DTO) | `static-file-validation.middleware.spec.ts` |
| **E-15** | Concorrência: dois envios simultâneos para a mesma reserva `pending` | Disparar 2 requisições quase ao mesmo tempo | Um dos dois vence a corrida (`waiting_approve`); o outro recebe `409` na leitura seguinte (não há lock distribuído — janela de corrida pequena, mas real; **não coberto por teste automatizado**, registrar como risco conhecido) | **sem automação** |

## 3. Edge cases

| ID | Pré-condição | Passos | Resultado esperado | Automação |
|---|---|---|---|---|
| **X-1** | Comprovante com exatamente 5.000.000 bytes decodificados | Enviar | Aceito (limite é `<=` 5 MiB) | `static-file-validation.middleware.spec.ts` ("aceita exatamente 5 MB") |
| **X-2** | Comprovante com 5.000.001 bytes | Enviar | Rejeitado (`400`) | idem |
| **X-3** | `mime_type` correto mas `base64` corrompido (não decodifica) | Enviar | `400` — "Conteudo do comprovante (base64) invalido" | idem |
| **X-4** | Reserva de 3 slots (R$ 160) | Enviar comprovante com os 3 `slots` na ordem invertida em relação à reserva | Aceito — a reconciliação compara os ids **ordenados**, não a ordem de envio | `reconcile-reserve.spec.ts` |
| **X-5** | `usuario_id` ausente (fluxo público) | Conferir a mensagem na fila | `usuario_id: null` (não quebra o schema da mensagem) | `inject-comprovante.usecase.spec.ts` |
| **X-6** | Nome de arquivo com acentos/espaços (`"comprovante ção.pdf"`) | Enviar | Aceito — o nome do arquivo não é usado como key (a key é sempre `‹uuid›.‹ext›`); `file_name` original só é metadado não persistido pelo `injection` | leitura do código (`buildProofKey`) |
| **X-7** | `INJECTION_INTERNAL_API_KEY` divergente entre `pagamentos` e `injection` (erro de configuração) | Configurar valores diferentes nos dois `.env` e enviar pelo front | `pagamentos` repassa a chamada, `injection` responde `401`, `pagamentos` repassa o `401` ao front (proxy transparente) | `forward-comprovante.adapter.spec.ts` + `transaction-history.controller.spec.ts` |
| **X-8** | Reserva de duração "1" mas com 2 `slots` no corpo | Enviar | `400` (inconsistência `duration_hours` vs. `slots.length`) detectada na reconciliação | `reconcile-reserve.spec.ts` |
| **X-9** | Comprovante PDF vs. JPG vs. PNG na mesma sessão de testes | Enviar um de cada | Cada um gera `proof_key` com a extensão correta (`.pdf`/`.jpg`/`.png`) | usecase ("extensão correta por mime") |
| **X-10** | Painel de revisão com uma reserva sem `review_note` anterior | Abrir o modal de revisão | Campo de observação vem vazio; salvar sem preencher não quebra (`review_note` fica ausente, não `""`) | `review-manual-payment.usecase.spec.ts` |

## 4. Checklist de regressão

| ID | Pré-condição | Passos | Resultado esperado |
|---|---|---|---|
| **R-1** | Fluxo de checkout Getnet/PIX (`POST /checkout`) | Rodar o fluxo de pagamento automático (gateway), sem comprovante manual | Continua funcionando — não foi tocado por esta task |
| **R-2** | Webhook Getnet | Disparar um webhook de teste | `process-getnet-webhook.usecase` intacto, sem relação com `manual_payments` |
| **R-3** | `GET /payment-methods` | Chamar a rota | Resposta igual à de antes |
| **R-4** | Reagendamento/cancelamento por protocolo (task 006b) | Cancelar uma reserva `approved` via `/reservas/protocol/:number/cancel` | Continua funcionando; `refund_status: manual`; **notificação WhatsApp** de cancelamento não foi afetada |
| **R-5** | Expiração lazy de reservas `pending` (task 006a) | Deixar uma reserva `pending` > 24 h sem comprovante, chamar `GET /reservas` | Ainda expira para `expired` (o `injection` não interfere nessa varredura, que é do `agendamentos`) |
| **R-6** | Comprovante de `ranking_agendamento` (fora de escopo da 008) | Enviar comprovante via `PATCH /ranking-agendamentos/protocolo/:np/comprovante` | Continua no fluxo antigo (Google Drive/base64) — **não foi migrado** nesta task |
| **R-7** | `beach-center-whatsapp` | Cancelar uma reserva `approved` | Mensagem `cancelamento-reembolso` ainda dispara (não depende de `manual_payments`) |
| **R-8** | Frontend `CreateReserveForm` | Criar uma reserva completa do zero (sem comprovante) | Reserva nasce `pending` normalmente — este passo não muda |
| **R-9** | `docker-compose.dev.yml` demais serviços | Subir a stack completa | `agendamentos`, `pagamentos`, `usuarios`, `aulas`, `whatsapp`, `campeonatos` (profile), `app`, `nginx` sobem normalmente; nenhuma porta/host conflita com `minio`(9000/9001)/`rabbitmq`(5672/15672)/`injection`(5006) |
| **R-10** | `beach-center-bff-pagamentos` demais rotas admin | `GET /transaction-history?status=approved` | Lista reservas `approved` (agora via `agendamentos`, mas o contrato de resposta para o front é preservado) |

## Rastreabilidade AC → cenário

| Critério de Aceite | Cenário(s) |
|---|---|
| AC-1 — pagamentos livre de I/O de arquivo | H-1 |
| AC-2 — validação estática (formato/tamanho) + guard | H-2, E-1, E-2, E-3, E-14, X-1, X-2, X-3 |
| AC-3 — upload no MinIO | H-3, E-13, X-6, X-9 |
| AC-4 — reserva "Em Análise" + proof_key | H-4 |
| AC-5 — evento na fila após o DB, resposta imediata | H-5, H-6 |
| AC-6 — front-end intacto | H-7 |
| AC-7 — pending/autorização/reconciliação | H-8, E-4, E-5, E-6, E-7, E-8, X-4, X-8 |
| AC-8 — compensação em falha | E-9, E-10 |
| AC-9 — painel de revisão sem `manual_payments` | H-9, H-10, H-11, E-11, E-12, X-7, X-10 |
| AC-10 — Hexagonal/ESLint | verificado no `/speckit-validate` (não é cenário manual) |
| AC-11 — infra reproduzível (docker-compose) | H-12, R-9 |
| AC-12 — cobertura ≥ 80% | verificado no `/speckit-unit-tests` (não é cenário manual) |

**Todos os 12 ACs cobertos** — AC-10 e AC-12 são verificados por ferramenta (lint/coverage), não por
roteiro manual, mas estão documentados para rastreabilidade completa.

## Contagem

| Categoria | Qtd |
|---|---|
| Caminhos felizes (H-*) | 12 |
| Fluxos de exceção (E-*) | 15 |
| Edge cases (X-*) | 10 |
| Regressão (R-*) | 10 |
| **Total** | **47** |

## Observações

- **Nenhum cenário foi executado de fato** — o ambiente de trabalho não tem Docker disponível
  (registrado desde o `/speckit-implement`/`/speckit-validate`). Este roteiro é para quem sobe a
  stack (`docker compose -f beach-center-server/docker-compose.dev.yml up -d --build`) localmente.
- **E-15** (concorrência sem lock distribuído) é um risco conhecido, sem automação — mesma
  categoria de risco que já existia no fluxo antigo do `pagamentos` (a condição de corrida não é
  nova desta task).
- **R-6** confirma que o comprovante de `ranking_agendamento` **não foi migrado** — está fora do
  escopo da US02 (Q6 do `context.md`), fica para task futura.

## Próximo passo

`/speckit-complete` — quality gate local (lint + testes + cobertura ≥ 80% nos 3 serviços TS já
confirmados no `/speckit-unit-tests`/`/speckit-validate`), commit e push dos repositórios
afetados.
