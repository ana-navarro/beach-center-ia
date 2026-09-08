# Reembolso

## Visão geral
Endpoint responsável por processar o reembolso (estorno) de um pagamento associado a uma reserva.
É exposto pelo serviço `services/beach-center-bff-pagamentos`. Base path: `/api/v1`.

**Importante:** o reembolso está **sempre mockado** (`MockRefundAdapter`), **independentemente** do
valor de `PAYMENT_PROVIDER`. Essa é uma decisão deliberada registrada em
`src/config/container.ts` ("O reembolso permanece mockado; a integração Getnet real fica isolada
no adapter") — não há hoje um caminho de reembolso real via Getnet acessível pela API, mesmo em
produção com `PAYMENT_PROVIDER=getnet`.

**Lacunas de regra de negócio conhecidas (comportamento atual, não corrigidas por esta
documentação):** o usecase `CreateRefundUsecase` **não consulta a reserva nem verifica se ela foi
paga** antes de reembolsar — ele chama diretamente o gateway de reembolso com os dados recebidos.
Como o gateway é sempre o mock, qualquer combinação de `reserve_id`/`payment_id`/`amount` é
"aprovada", inclusive valores diferentes do efetivamente pago e reservas nunca pagas. Também não há
diferenciação de fluxo entre `PIX` e `CREDIT_CARD` no reembolso — ambos passam pelo mesmo adapter
mockado.

## Autenticação/autorização
`authOrInternalApiKey` — aceita **um dos dois**:
- Header `x-api-key` igual a `PAGAMENTOS_INTERNAL_API_KEY` (chave interna que o próprio serviço
  `pagamentos` aceita, usada tipicamente por outros serviços do ecossistema); ou
- `authMiddleware` padrão (Bearer `idToken` Firebase de um usuário válido), caso a `x-api-key` não
  bata.

Se nem a `x-api-key` nem o Bearer forem válidos, cai no `authMiddleware`, que exige token Firebase.

## Endpoints

### POST /api/v1/refunds
Processa o reembolso de um pagamento. Aciona `CreateRefundUsecase`
(`domain/usecases/refund/create-refund.usecase.ts`), que chama o `IRefundGatewayPort` (sempre
`MockRefundAdapter`) e depois `PATCH {AGENDAMENTOS_API_URL}/reservas/:id/payment` para gravar
`refund_id`/`refund_status`/`refunded_at` na reserva em `agendamentos`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `reserve_id` | string | Sim | ObjectId (24 chars hex) da reserva |
| `payment_id` | string | Sim | Identificador do pagamento original (ex.: `mock_...` de um checkout aprovado) |
| `amount` | number | Sim | Valor a reembolsar, `>= 0`. **Sem validação de domínio** contra o valor efetivamente pago |
| `payment_method` | string | Não | `PIX` ou `CREDIT_CARD` — não afeta o comportamento do mock |

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Reembolso processado com sucesso",
  "data": {
    "refund_id": "refund_mock_AbCdEfGhIjKl_a1b2c3d4",
    "refund_status": "approved",
    "refunded_at": "2026-09-06T12:34:56.789Z"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Corpo inválido — mensagem real do yup (ex.: `reserve_id` fora do formato ObjectId) |
| 401 | Sem `Authorization` e sem `x-api-key` válida (`"Token nao fornecido"`), ou `x-api-key` errada e sem Bearer válido |
| 403 | `idToken` válido, mas usuário não encontrado na coleção `usuarios` |

Não há hoje erro 404/409 específico de "reserva não encontrada" ou "pagamento já reembolsado" —
ver nota de lacuna de negócio acima.

**Exemplo de chamada**
```bash
curl -X POST "http://localhost:5001/api/v1/refunds" \
  -H "x-api-key: <PAGAMENTOS_INTERNAL_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "reserve_id": "665f1a2b3c4d5e6f7a8b9c0d",
    "payment_id": "mock_AbCdEfGhIjKl",
    "amount": 80,
    "payment_method": "PIX"
  }'
```

## Referências
- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/refund/refund.controller.ts`
- DTO: `services/beach-center-bff-pagamentos/src/applications/dto/create-refund.dto.ts`
- Usecase: `services/beach-center-bff-pagamentos/src/domain/usecases/refund/create-refund.usecase.ts`
- Adapter de gateway: `services/beach-center-bff-pagamentos/src/infra/adapters/getnet/refund/mock-refund.adapter.ts` (real, não usado: `.../getnet-refund.adapter.ts`)
- Middleware de auth: `services/beach-center-bff-pagamentos/src/applications/middlewares/auth.middleware.ts` (`authOrInternalApiKey`)
- Wiring do provedor: `services/beach-center-bff-pagamentos/src/config/container.ts`
