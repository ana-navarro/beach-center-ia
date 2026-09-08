# Checkout

## Visão geral
Endpoint responsável por iniciar o pagamento de uma reserva de quadra. É exposto pelo serviço
`services/beach-center-bff-pagamentos` e, ao ser chamado, primeiro cria a reserva no serviço
`beach-center-bff-agendamentos` (`POST /reservas`) e, em seguida, aciona o gateway de pagamento
configurado (mock ou Getnet) para gerar a cobrança. Base path do serviço: `/api/v1`.

O comportamento da resposta depende diretamente da variável de ambiente `PAYMENT_PROVIDER`:

- `PAYMENT_PROVIDER` ausente ou diferente de `"getnet"` → usa `MockCheckoutAdapter`. O pagamento é
  **aprovado automaticamente** (`payment_status: "APPROVED"`), sem chamar nenhuma API externa;
  `payment_id` é gerado no formato `mock_<12 chars>`; `payment_url` **não é retornado** (o mock
  devolve `redirect_url: ""`, que o usecase descarta); a reserva já é promovida a
  `status: "approved"` dentro do próprio usecase (`PATCH /reservas/:id/status` é chamado para o
  serviço de agendamentos).
- `PAYMENT_PROVIDER=getnet` → usa `GetnetCheckoutAdapter` (sandbox ou produção, conforme
  `GETNET_ENV`), que chama a API real da Getnet (`POST {GETNET_BASE_URL}/v1/payments/checkout`).
  Exige `GETNET_CLIENT_ID`, `GETNET_CLIENT_SECRET` e `GETNET_SELLER_ID`. O pagamento fica
  `payment_status: "PENDING"`, `payment_url` é preenchido com a URL de redirecionamento da Getnet e
  a reserva permanece `status: "pending"` até a confirmação chegar via webhook
  (`POST /webhooks/getnet`).

## Autenticação/autorização
`authMiddleware` (Bearer) — exige `Authorization: Bearer <idToken Firebase>` de um usuário
autenticado e existente na coleção `usuarios` (qualquer `user_type`, sem restrição de papel).

## Endpoints

### POST /api/v1/checkout
Cria a reserva remotamente em `agendamentos` e gera a cobrança no provedor de pagamento
configurado. Aciona `CreateCheckoutUsecase` (`domain/usecases/checkout/create-checkout.usecase.ts`).

Fluxo interno (independente do provedor):
1. Registra o início da transação no log de auditoria interno (`ITransactionLogPort`,
   coleção `payment_transaction_history` — não exposto por nenhum endpoint público).
2. `POST {AGENDAMENTOS_API_URL}/reservas` para criar a reserva (repassa o header
   `Authorization` recebido, se presente).
3. Chama o gateway de pagamento (`IPaymentGatewayPort`).
4. `PATCH {AGENDAMENTOS_API_URL}/reservas/:id/payment` para gravar `checkout_id`/`payment_id`/
   `payment_method` na reserva.
5. Se `payment_status === "APPROVED"` (sempre o caso no mock), chama
   `PATCH {AGENDAMENTOS_API_URL}/reservas/:id/status {status:"approved"}`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome do cliente |
| `email` | string (email) | Sim | E-mail do cliente |
| `phone` | string | Sim | Telefone do cliente |
| `scheduling_id` | string[] | Sim | 1 a 3 ObjectIds (24 chars hex) de horários (`agendamentos`). Máximo 3 — mensagem `"E permitido selecionar no maximo 3 horarios"` |
| `total` | number | Sim | Valor total da reserva, `>= 0` |
| `price` | number | Não | Preço unitário/base, `>= 0` |
| `discount` | number \| null | Não | Desconto aplicado, `>= 0` |
| `equipment.self_equipment` | boolean | Sim | Se o cliente leva o próprio equipamento |
| `equipment.equipment` | string[] | Condicional | Obrigatório e com `min(1)` quando `self_equipment: false`; ignorado quando `true` |
| `status` | string | Não | Um de `pending`, `approved`, `rejected`, `cancelled` (default `pending`) |
| `payment_method` | string | Sim | `PIX` ou `CREDIT_CARD` |

**Resposta de sucesso** — `201 Created`

Exemplo com `PAYMENT_PROVIDER=mock` (padrão):
```json
{
  "message": "Checkout gerado com sucesso",
  "data": {
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "reserve": {
      "id": "665f1a2b3c4d5e6f7a8b9c0d",
      "name": "Maria Silva",
      "email": "maria@example.com",
      "phone": "11999999999",
      "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c00"],
      "total": 80,
      "discount": null,
      "status": "approved",
      "equipment": { "self_equipment": true },
      "number": "R-0001",
      "payment_id": "mock_AbCdEfGhIjKl",
      "checkout_id": "mock_AbCdEfGhIjKl",
      "payment_method": "PIX"
    },
    "payment_status": "APPROVED",
    "payment_method": "PIX",
    "payment_id": "mock_AbCdEfGhIjKl",
    "mock": true
  }
}
```

Com `PAYMENT_PROVIDER=getnet`, `data.payment_status` é `"PENDING"`, `data.payment_url` vem
preenchido com a URL de redirecionamento da Getnet, `data.reserve.status` permanece `"pending"` e
`data.mock` não é retornado.

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido — `message` é a mensagem real do yup para o primeiro campo que falhar (ex.: `"E permitido selecionar no maximo 3 horarios"`), `errors` traz um item por campo inválido. **Diferente do serviço `usuarios`**, que sempre retorna `"Dados inválidos"` fixo — aqui `handle-http-error.ts` repassa `error.message` do yup sem sobrescrever |
| 401 | `Authorization` ausente (`"Token nao fornecido"`) ou inválido/expirado (`"Token invalido ou expirado"`) |
| 403 | `idToken` válido, mas usuário não existe na coleção `usuarios` (`"Usuario nao encontrado no sistema"`) |
| 404 | `scheduling_id` referencia horário inexistente em `agendamentos` (`"Agendamento nao encontrado"`, mapeado a partir do 404 remoto por `create-reserve.adapter.ts`) |
| 500 | Falha inesperada (ex.: horário já reservado/pago — `agendamentos` pode responder com um status que `create-reserve.adapter.ts` não trata especialmente, propagando como erro genérico) ou falha na chamada ao gateway de pagamento |

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5001/api/v1/checkout" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Maria Silva",
    "email": "maria@example.com",
    "phone": "11999999999",
    "scheduling_id": ["665f1a2b3c4d5e6f7a8b9c00"],
    "total": 80,
    "equipment": { "self_equipment": true },
    "payment_method": "PIX"
  }'
```

## Referências
- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/checkout/checkout.controller.ts`
- DTO: `services/beach-center-bff-pagamentos/src/applications/dto/create-checkout.dto.ts`
- Usecase: `services/beach-center-bff-pagamentos/src/domain/usecases/checkout/create-checkout.usecase.ts`
- Adapters de gateway: `services/beach-center-bff-pagamentos/src/infra/adapters/getnet/checkout/mock-checkout.adapter.ts`, `.../getnet-checkout.adapter.ts`
- Adapter de reserva: `services/beach-center-bff-pagamentos/src/infra/adapters/reserve/create-reserve.adapter.ts`
- Erros de domínio: `services/beach-center-bff-pagamentos/src/domain/errors.ts`
- Tratamento HTTP de erro: `services/beach-center-bff-pagamentos/src/applications/controllers/shared/handle-http-error.ts`
- Config/container: `services/beach-center-bff-pagamentos/src/config/env.ts`, `services/beach-center-bff-pagamentos/src/config/container.ts`
