# Comprovante (ingestão de comprovante de pagamento)

## Visão geral

`beach-center-bff-injection` (task 008, US02) é o novo microsserviço **porta de entrada para o
upload de comprovantes de pagamento**. Nasceu do desacoplamento do `beach-center-bff-pagamentos`
(Princípio de Responsabilidade Única): o `pagamentos` fica só com dados transacionais (valores,
status lógico do pagamento, gateway Getnet), e todo o I/O pesado de arquivo — recebimento,
validação estática, upload para o storage, marcação da reserva e enfileiramento para a IA — vira
responsabilidade exclusiva deste serviço.

**Arquitetura orientada a eventos (Princípio VI da Constituição, task 007):**

```
Front-end → (via proxy do beach-center-bff-pagamentos) → beach-center-bff-injection
  → S3 SDK (putObject) → MinIO
  → REST interna (x-api-key) → beach-center-bff-agendamentos  (status: "waiting_approve" + proof_key)
  → publish { agendamento_id, usuario_id, email, file_url, proof_key, bucket, mime_type, timestamp }
  → RabbitMQ (fila "comprovante.validar")
```

O serviço é **stateless** — não tem banco de dados próprio. Toda gravação de estado passa pelo
`beach-center-bff-agendamentos` (dono da coleção de reservas), via REST interna. A resposta ao
chamador é **imediata** (`202 Accepted`): a validação por IA (`beach-center-bff-llm-engine`,
consumidor da fila — US03+) roda depois, em background.

O **front-end não muda**: o contrato de entrada é o mesmo JSON com `proof_file.base64` que o
`pagamentos` já aceitava — só a URL muda (e nem isso, porque o `pagamentos` mantém as rotas
antigas como proxy; ver [`../beach-center-bff-pagamentos/comprovante-pagamento.md`](../beach-center-bff-pagamentos/comprovante-pagamento.md)).

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /agendamentos/comprovantes` | `internalApiKeyMiddleware` — header `x-api-key` = `INJECTION_INTERNAL_API_KEY`. Sem/errada → `401 {"message": "Unauthorized"}`. **Só o `beach-center-bff-pagamentos` chama esta rota** (proxy) — não é exposta ao público pelo gateway |

A autorização **sobre a reserva** (o cliente é dono dela ou tem o link público) é feita pelo
próprio usecase, não por middleware — ver abaixo.

## Endpoints

### POST /api/v1/agendamentos/comprovantes

Recebe o comprovante, valida, sobe para o MinIO, marca a reserva como **"Em Análise"** e publica
o evento de validação na fila. Aciona `InjectComprovanteUsecase`.

**Validação estática** (middleware `staticFileValidationMiddleware`, roda **antes** do usecase —
nenhum I/O acontece se falhar):
- `proof_file.mime_type` deve ser `image/jpeg`, `image/png` ou `application/pdf`.
- `proof_file.base64` deve decodificar para no máximo **5 MiB** (`5 * 1024 * 1024` bytes).
- Base64 ausente, vazio ou não-decodificável também é rejeitado.

**Regras de negócio** (`InjectComprovanteUsecase`):
1. Lê a reserva em `beach-center-bff-agendamentos` (`GET /reservas/:id`, `x-api-key`) — `404` se
   não existir.
2. A reserva precisa estar `status: "pending"` — senão `409`.
3. Autorização: e-mail do `requester_email` (repassado pelo proxy do `pagamentos` quando
   autenticado) igual ao e-mail da reserva, **ou** `public_reserve_token` válido (checado contra
   `beach-center-bff-agendamentos`) — senão `403`.
4. Reconciliação: `amount` (∈ `{80,120,160}`), `duration_hours` (∈ `{1,2,3}`) e os `slots`
   enviados precisam bater com a reserva (`total`, quantidade e ids de `scheduling_id`) — senão
   `400`.
5. Upload do arquivo no MinIO: `PutObjectCommand` no bucket `MINIO_BUCKET`, `Key` = `<uuid>.<ext>`
   (extensão derivada do `mime_type`), `ContentType` = o mime enviado.
6. `PATCH /reservas/:id/status` (`agendamentos`, `x-api-key`) com `status: "waiting_approve"` —
   é essa transição que **gera o protocolo** (`number`, task 006a).
7. `PATCH /reservas/:id/payment` (`agendamentos`, `x-api-key`) com `{ proof_key }`.
8. Publica na fila **`comprovante.validar`** (RabbitMQ, durável, mensagem persistente) o payload
   descrito abaixo.
9. Responde **`202`** imediatamente — não aguarda nenhuma inferência de IA.

**Compensação em falha** (depois do upload):
- Falha no passo 6 (marcar `waiting_approve`) → remove o objeto do MinIO (best-effort) → `502`.
- Falha no passo 7 (gravar `proof_key`) → reverte a reserva para `pending` + remove o objeto → `502`.
- Falha no passo 8 (publish na fila) → reverte a reserva para `pending` + remove o objeto → `502`.
  (O `number`/protocolo já gerado é preservado — a próxima tentativa reaproveita.)

**Corpo da requisição** — JSON (mesmo contrato do `pagamentos`, sem `multipart`/`multer`)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `reserve_id` | string | Sim | ObjectId (24 hex) da reserva em `agendamentos` |
| `reserve_number` | string | Sim | Aceito pelo DTO (compatibilidade com o corpo do front); não é usado — o protocolo vem da transição de status |
| `amount` | number | Sim | Um de `80`, `120`, `160` |
| `duration_hours` | number | Sim | Um de `1`, `2`, `3` |
| `user.name` | string | Sim | Nome do cliente |
| `user.email` | string (email) | Sim | E-mail do cliente |
| `user.phone` | string | Sim | Telefone do cliente |
| `slots` | array (1 a 3 itens) | Sim | Cada item: `id`, `date`, `start_time`, `end_time`, `court`, `unit` |
| `proof_file.file_name` | string | Sim | Nome do arquivo original (não é usado como object key) |
| `proof_file.mime_type` | string | Sim | `image/jpeg` \| `image/png` \| `application/pdf` |
| `proof_file.base64` | string | Sim | Conteúdo em base64 — decodificado deve ter no máximo 5 MiB |
| `public_reserve_token` | string | Não | Token do link público de reserva (autorização quando não há usuário autenticado) |
| `usuario_id` | string | Não | uid do Firebase do requester — repassado pelo proxy do `pagamentos` quando a requisição é autenticada; ausente no fluxo público (vai como `null` no payload da fila) |
| `requester_email` | string | Não | E-mail do requester autenticado — repassado pelo proxy do `pagamentos` |

**Resposta de sucesso** — `202 Accepted`
```json
{
  "message": "Comprovante recebido e em processamento",
  "data": {
    "agendamento_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "numero_protocolo": "0348871290",
    "proof_key": "3f2c1a9e-6b7d-4e2a-9c1b-7a5d0e2f9c11.png",
    "file_url": "http://localhost:9000/comprovantes/3f2c1a9e-6b7d-4e2a-9c1b-7a5d0e2f9c11.png",
    "queued": true
  }
}
```

**Mensagem publicada na fila `comprovante.validar`**
```json
{
  "agendamento_id": "665f1a2b3c4d5e6f7a8b9c0d",
  "usuario_id": null,
  "email": "maria@example.com",
  "file_url": "http://localhost:9000/comprovantes/3f2c1a9e-....png",
  "proof_key": "3f2c1a9e-....png",
  "bucket": "comprovantes",
  "mime_type": "image/png",
  "timestamp": "2026-09-10T20:00:00.000Z"
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `Dados inválidos` (DTO) / formato de comprovante inválido / comprovante acima de 5 MB / base64 inválido (middleware de validação estática) / `"Os dados do pagamento nao correspondem a reserva"` (reconciliação) |
| 401 | `{"message": "Unauthorized"}` — `x-api-key` ausente ou incorreta |
| 403 | `"Acesso negado para esta reserva"` — nem e-mail nem `public_reserve_token` autorizam |
| 404 | `"Reserva nao encontrada"` |
| 409 | `"A reserva nao esta pendente de pagamento"` — reserva já em `waiting_approve`/`approved`/`rejected`/`cancelled`/`expired` |
| 502 | Falha ao subir para o MinIO, ou falha ao gravar/enfileirar após o upload (com compensação — ver acima) |

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5006/api/v1/agendamentos/comprovantes" \
  -H "x-api-key: dev-injection-key" \
  -H "Content-Type: application/json" \
  -d '{
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve_number": "R-0001",
    "amount": 80,
    "duration_hours": 1,
    "user": { "name": "Maria Silva", "email": "maria@example.com", "phone": "11999999999" },
    "slots": [{ "id": "665f1a2b3c4d5e6f7a8b9c00", "date": "2026-09-10", "start_time": "18:00", "end_time": "19:00", "court": "Quadra 1", "unit": "Unidade Centro" }],
    "proof_file": { "file_name": "comprovante.png", "mime_type": "image/png", "base64": "<base64>" },
    "usuario_id": "firebase-uid-123",
    "requester_email": "maria@example.com"
  }'
```

> Este endpoint normalmente **não** é chamado direto pelo front — o front continua chamando
> `POST /transaction-history` (ou `/public/transaction-history`) no `beach-center-bff-pagamentos`,
> que repassa a chamada para cá.

## Referências

- Rota: `src/applications/routes/routes.ts`, `src/applications/routes/comprovante.route.ts`
- Middlewares: `src/applications/middlewares/internal-api-key.middleware.ts`, `.../static-file-validation.middleware.ts`
- Controller: `src/applications/controllers/comprovante/upload/upload-comprovante.controller.ts`
- DTO: `src/applications/dto/inject-comprovante.dto.ts`
- Usecase: `src/domain/usecases/comprovante/inject-comprovante/inject-comprovante.usecase.ts`
- Regra compartilhada: `src/domain/usecases/shared/reconcile-reserve.ts`
- Modelos: `src/domain/models/comprovante.model.ts`, `src/domain/models/reserve-slot.model.ts`
- Ports: `src/domain/ports/input/comprovante.input-port.ts`, `src/domain/ports/output/{reserva-client,object-storage,validation-queue}.port.ts`
- Adapters: `src/infra/adapters/object-storage/{put-object,delete-object}/*.adapter.ts` (+ `s3-client.ts`), `src/infra/adapters/reserva-client/{read-reserva,check-public-link,set-in-review,set-proof,revert-to-pending}/*.adapter.ts` (+ `agendamentos-http.ts`), `src/infra/adapters/validation-queue/publish/publish-validation-event.adapter.ts`
- Config: `src/config/env.ts` (`MINIO_*`, `RABBITMQ_*`, `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY`, `INJECTION_INTERNAL_API_KEY`, `PORT` — default `5006`), `src/config/container.ts`
- Erros de domínio: `src/domain/errors.ts`
- Consumidores/dependências externas: [`../beach-center-bff-agendamentos/reserva.md`](../beach-center-bff-agendamentos/reserva.md) (`PATCH /reservas/:id/status`, `PATCH /reservas/:id/payment`, `GET /reservas/:id`, `GET /links-reserva-publica/:token/reservas/:id/authorization`), [`../beach-center-bff-pagamentos/comprovante-pagamento.md`](../beach-center-bff-pagamentos/comprovante-pagamento.md) (chamador — proxy)
- Infra (task 008): `beach-center-server/docker-compose.dev.yml` — serviços `minio`, `minio-init` (cria o bucket `comprovantes`), `rabbitmq`, `injection`
