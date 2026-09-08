# Forma de Pagamento

## Visão geral
Endpoint de consulta das formas de pagamento disponíveis para checkout, exposto pelo serviço
`services/beach-center-bff-pagamentos` (base `/api/v1`). A lista é **estática** — definida em
código (`ListPaymentMethodsUsecase`), não vem de banco de dados nem de configuração externa.

## Autenticação/autorização
`authMiddleware` (Bearer) — qualquer usuário autenticado (`idToken` Firebase válido, existente na
coleção `usuarios`), sem restrição de `user_type`.

## Endpoints

### GET /api/v1/payment-methods
Retorna a lista fixa de formas de pagamento suportadas pelo checkout. Aciona
`ListPaymentMethodsUsecase` (`domain/usecases/payment-methods/list-payment-methods.usecase.ts`),
que não depende de nenhuma porta de infraestrutura — devolve a constante `PAYMENT_METHODS` em
memória.

**Parâmetros de path/query** — nenhum.

**Corpo da requisição** — nenhum.

**Resposta de sucesso** — `200 OK`
```json
{
  "message": "Formas de pagamento consultadas com sucesso",
  "data": [
    { "value": "PIX", "label": "PIX", "enabled": true },
    { "value": "CREDIT_CARD", "label": "Cartao de credito", "enabled": true }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | `Authorization` ausente (`"Token nao fornecido"`) ou token inválido/expirado (`"Token invalido ou expirado"`) |
| 403 | `idToken` válido, mas usuário não encontrado na coleção `usuarios` (`"Usuario nao encontrado no sistema"`) |

**Exemplo de chamada**
```bash
curl "http://localhost:5001/api/v1/payment-methods" \
  -H "Authorization: Bearer <idToken>"
```

## Referências
- Rota: `services/beach-center-bff-pagamentos/src/applications/routes/routes.ts`
- Controller: `services/beach-center-bff-pagamentos/src/applications/controllers/payment-methods/payment-methods.controller.ts`
- Usecase: `services/beach-center-bff-pagamentos/src/domain/usecases/payment-methods/list-payment-methods.usecase.ts`
- Modelo: `services/beach-center-bff-pagamentos/src/domain/models/payment.model.ts` (`IPaymentMethodOption`)
- Middleware de auth: `services/beach-center-bff-pagamentos/src/applications/middlewares/auth.middleware.ts` (`authMiddleware`)
