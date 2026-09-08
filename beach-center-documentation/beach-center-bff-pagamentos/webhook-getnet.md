# Webhook Getnet

## Visão geral
Endpoint de callback (webhook) chamado pela Getnet (ou por qualquer chamador autorizado) para
notificar o resultado assíncrono de um pagamento iniciado via `POST /checkout` com
`PAYMENT_PROVIDER=getnet`. É exposto pelo serviço `services/beach-center-bff-pagamentos`
(base `/api/v1`). A URL de notificação enviada à Getnet no momento do checkout é montada por
`getnet-checkout.adapter.ts#buildNotificationUrl`:
`{API_URL}/api/v1/webhooks/getnet?token=<GETNET_WEBHOOK_TOKEN>` — em ambiente de
sandbox/homologação, `API_URL` precisa ser publicamente alcançável pela Getnet (não `localhost`).

Este endpoint **sempre responde `200`**, mesmo em caso de payload inválido, reserva inexistente ou
falha interna — a resposta vazia evita que a Getnet reenvie o webhook em loop por erro do lado do
serviço. Qualquer erro é apenas logado no console (`console.error`) e no log de auditoria interno.

## Autenticação/autorização
`webhookTokenMiddleware` — **não** usa Bearer/Firebase. Valida um token compartilhado, aceito via
**query string** (`?token=...`) **ou** header `x-api-key`, comparado a
`GETNET_WEBHOOK_TOKEN` (ou, na ausência dela, `PAGAMENTOS_WEBHOOK_TOKEN`).

⚠️ **Comportamento importante:** se nenhuma das duas variáveis (`GETNET_WEBHOOK_TOKEN` /
`PAGAMENTOS_WEBHOOK_TOKEN`) estiver configurada, o middleware **não bloqueia nada** — o endpoint
fica efetivamente **aberto** (qualquer requisição é aceita e processada como um webhook legítimo).
Isso é um comportamento atual do código-fonte, não uma correção proposta por esta documentação.

Não há verificação de assinatura HMAC nem deduplicação por `payment_id` — a única proteção contra
reprocessamento (replay) é o próprio estado da reserva (ver regra 4 abaixo).

## Endpoints

### POST /api/v1/webhooks/getnet
Processa a notificação de status de pagamento da Getnet. Aciona
`ProcessGetnetWebhookUsecase` (`domain/usecases/webhook/process-getnet-webhook.usecase.ts`).

Regras de negócio (todas resultam em `200` na resposta HTTP, independentemente do resultado
interno):
1. Se `order_id` não vier no payload, o webhook é ignorado (log `webhook_without_order_id`), sem
   nenhuma alteração.
2. `order_id` é usado como id da reserva (`GET {AGENDAMENTOS_API_URL}/reservas/:id`); se a reserva
   não existir, o webhook é ignorado (log `reserve_not_found`).
3. Se `payment_id` vier no payload, é gravado na reserva
   (`PATCH .../reservas/:id/payment {payment_id}`) antes da checagem de idempotência abaixo.
4. **Idempotência**: se `reserve.status !== "pending"` (já aprovada/rejeitada/cancelada
   anteriormente), o webhook é ignorado (log `reserve_not_pending`) — nenhuma nova alteração de
   status ocorre, mesmo em reenvios (replay) do mesmo evento.
5. Se a reserva está `pending`, o `status` do payload determina a ação:
   - `"APPROVED"` → `PATCH .../reservas/:id/status {status:"approved"}`.
   - `"DENIED"` ou `"CANCELED"` → `PATCH .../reservas/:id/status {status:"rejected"}`.
   - Qualquer outro valor de `status` → ignorado (log `webhook_status_ignored`), sem alteração.

**Concorrência:** a leitura (`GET reserva`) e a escrita (`PATCH status`) não são atômicas neste
serviço; dois webhooks simultâneos para a mesma reserva podem, em tese, ambos ler `status:"pending"`
antes que o primeiro efetive a escrita — não há lock/transação que impeça isso no lado de
`pagamentos`.

**Parâmetros de path/query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `token` | string (query) | Condicional | Alternativa ao header `x-api-key`; obrigatório apenas se `GETNET_WEBHOOK_TOKEN`/`PAGAMENTOS_WEBHOOK_TOKEN` estiver configurado |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `order_id` | string | Não (mas necessário para qualquer efeito) | Id da reserva em `agendamentos` |
| `payment_id` | string | Não | Id do pagamento no gateway |
| `status` | string | Sim | Status reportado pela Getnet — tratados: `APPROVED`, `DENIED`, `CANCELED`; qualquer outro valor é ignorado |

**Resposta de sucesso** — `200 OK`, corpo vazio (`res.status(200).end()`), sempre — mesmo quando o
payload é inválido, a reserva não existe, ou ocorre erro interno.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | `{"message": "Unauthorized"}` — apenas quando `GETNET_WEBHOOK_TOKEN`/`PAGAMENTOS_WEBHOOK_TOKEN` está configurado **e** o `token`/`x-api-key` recebido não confere |

Nenhum outro código de erro é retornado por este endpoint — falhas de validação/processamento são
absorvidas e resultam em `200` vazio (ver "Visão geral").

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5001/api/v1/webhooks/getnet?token=<GETNET_WEBHOOK_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "payment_id": "getnet-pay-1",
    "status": "APPROVED"
  }'
```

## Referências
- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/webhook/getnet-webhook.controller.ts`
- DTO: `services/beach-center-bff-pagamentos/src/applications/dto/getnet-webhook.dto.ts`
- Usecase: `services/beach-center-bff-pagamentos/src/domain/usecases/webhook/process-getnet-webhook.usecase.ts`
- Middleware de auth: `services/beach-center-bff-pagamentos/src/applications/middlewares/auth.middleware.ts` (`webhookTokenMiddleware`)
- Geração da URL de notificação: `services/beach-center-bff-pagamentos/src/infra/adapters/getnet/checkout/getnet-checkout.adapter.ts`
- Config: `services/beach-center-bff-pagamentos/src/config/env.ts` (`webhookToken`, `apiUrl`)
