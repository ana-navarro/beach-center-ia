# Link de Reserva Pública

## Visão geral

Recurso exposto pelo serviço **`beach-center-bff-agendamentos`** sob o prefixo `/api/v1/links-reserva-publica`. Um link de reserva pública é um token de uso único gerado pelo ADMIN que permite que uma pessoa **sem autenticação** crie uma reserva (por exemplo, para compartilhar por WhatsApp/e-mail). O ciclo de vida do token é uma máquina de estados simples: `active` → `processing` (durante a criação da reserva) → `used` (reserva criada com sucesso) — ou de volta para `active` se a criação falhar.

> **Nota de integração cross-serviço:** o endpoint `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` **não é destinado a clientes de frontend**. Ele é consumido internamente pelo `beach-center-bff-pagamentos`, que o chama (com `x-api-key`) para confirmar que uma reserva pertence a um link público válido e já utilizado antes de processar o pagamento dessa reserva.

## Autenticação/autorização

| Endpoint | Guard |
|---|---|
| `POST /links-reserva-publica` | `authMiddleware` + `requireRole("ADMIN")` (Bearer, apenas ADMIN) |
| `GET /links-reserva-publica/:token` | Pública (sem autenticação) |
| `POST /links-reserva-publica/:token/reservas` | Pública (sem autenticação) |
| `GET /links-reserva-publica/:token/reservas/:reserveId/authorization` | `internalApiKeyMiddleware` (header `x-api-key` obrigatório, igual a `AGENDAMENTOS_INTERNAL_API_KEY`) |

## Endpoints

### POST /api/v1/links-reserva-publica

Cria um novo link de reserva pública com token aleatório de 32 caracteres (`nanoid`, alfabeto alfanumérico). Aciona `container.publicReserveLink.createLink`, que persiste o link com `status: "active"`. A URL pública final (`data.url`) é montada no controller a partir de `PUBLIC_APP_URL` — **se essa variável de ambiente não estiver configurada, o campo `url` é omitido do payload de resposta** (não é enviado como `null`).

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `expires_at` | string (ISO date) | Não | Data de expiração do link. Se informada, **deve ser uma data futura** (yup `.min(new Date())`); se ausente, o link não expira por data. |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Link publico criado com sucesso",
  "data": {
    "token": "aB3xQ...32 chars",
    "status": "active",
    "expires_at": null,
    "created_by": "firebase-uid-do-admin",
    "url": "https://app.beachcenter.com/agendar/aB3xQ...32 chars"
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `expires_at` no passado ou payload inválido (`yup.ValidationError` → `"Dados inválidos"`) |
| 401 | Token Bearer ausente ou inválido |
| 403 | Usuário autenticado não é ADMIN |

**Exemplo de chamada**

```bash
curl -X POST "https://api.exemplo.com/api/v1/links-reserva-publica" \
  -H "Authorization: Bearer <idToken>" \
  -H "Content-Type: application/json" \
  -d '{"expires_at": "2026-12-31T23:59:59.000Z"}'
```

---

### GET /api/v1/links-reserva-publica/:token

Valida um token de link público, retornando apenas `token` e `expires_at` se o link estiver **ativo, não expirado e não usado**. Aciona `container.publicReserveLink.validateLink`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `token` | string | Sim | Token do link público |

**Resposta de sucesso**

`200 OK`
```json
{
  "message": "Link valido",
  "data": {
    "token": "aB3xQ...32 chars",
    "expires_at": null
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `token` ausente na URL (`"Token nao informado"` — checagem direta no controller, fora do padrão `handleHttpError`) |
| 404 | Link inexistente, `status` diferente de `active` (já `processing`/`used`) ou `expires_at` no passado — mensagem `"Link invalido, expirado ou ja utilizado"` |

**Exemplo de chamada**

```bash
curl "https://api.exemplo.com/api/v1/links-reserva-publica/aB3xQ...32chars"
```

---

### POST /api/v1/links-reserva-publica/:token/reservas

Cria uma reserva usando um link público, **sem exigir autenticação**. Orquestração composta em `container.publicReserveLink.createReserveWithToken`:

1. `claimLinkPort.execute(token)` — tenta atomicamente mudar o link de `active` para `processing` (só sucede se `status: "active"` e não expirado). Se falhar (token inexistente, já `processing`/`used`, ou expirado), lança `ConflictError("Link invalido, expirado ou ja utilizado")` **antes** de qualquer criação de reserva.
2. Com o link reivindicado, chama o mesmo usecase de criação de reserva usado por `POST /reservas` (`createReserveUseCase.execute`) — aplica todas as validações de horário compartilhadas: limite de 3 `scheduling_id`, sem duplicados, existência dos agendamentos, datas não passadas, ausência de conflito com evento agendado (`EventConflictService`) e disponibilidade (`available: true`). Ver documentação completa dessas regras em [`reserva.md`](./reserva.md#post-apiv1reservas).
3. Se a reserva for criada com sucesso, `markUsedLinkPort.execute(token, reserve.id)` marca o link como `used`, grava `used_at` e `reserve_id`.
4. Se a criação retornar `null` (algum `scheduling_id` não existe) **ou** lançar um erro de validação, `releaseLinkPort.execute(token)` devolve o link para `status: "active"`, permitindo reuso do mesmo token.

**Corpo da requisição** (idêntico ao de `POST /reservas`, ver [`reserva.md`](./reserva.md#post-apiv1reservas) para o detalhamento completo)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `name` | string | Sim | Nome do cliente |
| `email` | string | Sim | E-mail do cliente |
| `phone` | string | Sim | Telefone do cliente |
| `scheduling_id` | string[] (1 a 3) | Sim | IDs dos agendamentos (máx. 3, sem repetição) |
| `price` | number | Não | Preço unitário |
| `discount` | number | Não | Desconto aplicado |
| `total` | number | Sim | Valor total da reserva |
| `equipment` | object | Sim | `{ self_equipment: boolean, equipment?: string[] }` |
| `number` | string | Não | Número de protocolo customizado (senão gerado automaticamente) |

**Resposta de sucesso**

`201 Created`
```json
{
  "message": "Reserva criada com sucesso",
  "data": {
    "id": "65f1...",
    "number": "1234567890",
    "status": "pending",
    "name": "Fulano de Tal",
    "email": "fulano@exemplo.com",
    "phone": "11999999999",
    "scheduling_id": ["65f0..."],
    "total": 80,
    "equipment": { "self_equipment": true }
  }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | Payload inválido (yup); mais de 3 horários; horários repetidos; agendamento em data passada |
| 404 | Um ou mais `scheduling_id` inexistentes — shape especial `{message: "Agendamento nao encontrado", id: [scheduling_id enviados]}`, retornado quando o usecase devolve `reserve === null` |
| 409 | Token de link inválido/expirado/já utilizado (`"Link invalido, expirado ou ja utilizado"`); horário com conflito de evento agendado (`"Um ou mais horarios selecionados possuem excecao de agendamento"`); horário indisponível (`"Um ou mais horarios selecionados nao estao disponiveis"`) |

**Exemplo de chamada**

```bash
curl -X POST "https://api.exemplo.com/api/v1/links-reserva-publica/aB3xQ...32chars/reservas" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Fulano de Tal",
    "email": "fulano@exemplo.com",
    "phone": "11999999999",
    "scheduling_id": ["65f0..."],
    "total": 80,
    "equipment": {"self_equipment": true}
  }'
```

---

### GET /api/v1/links-reserva-publica/:token/reservas/:reserveId/authorization

Verifica se uma reserva específica foi criada por meio de um determinado link público — usado por `beach-center-bff-pagamentos` para autorizar o processamento de pagamento de reservas originadas de link público. Aciona `container.publicReserveLink.authorizePublicReserve(token, reserveId)`.

**Comportamento notável:** esta rota **nunca lança um erro de domínio** para o caso "não autorizado" — o resultado é sempre um booleano (`data.authorized`), e o status HTTP reflete diretamente esse booleano (200 se autorizado, 403 se não). Internamente, o adapter só considera autorizado um link com `status: "used"` cujo `reserve_id` bata exatamente com o `:reserveId` informado; um `reserveId` que não seja um ObjectId válido também resulta em `false` sem lançar exceção.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `token` | string | Sim | Token do link público |
| `reserveId` | string (ObjectId) | Sim | ID da reserva a validar |

**Resposta de sucesso**

`200 OK` (autorizado)
```json
{
  "message": "Link autorizado",
  "data": { "authorized": true }
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `token` ou `reserveId` ausentes na URL (`"Dados nao informados"`) |
| 401 | Header `x-api-key` ausente ou incorreto (`internalApiKeyMiddleware` retorna `{"message": "Unauthorized"}`, resposta própria em inglês, não passa por `handleHttpError`) |
| 403 | Link **não autorizado** para essa reserva — retornado como resposta normal (não como erro de domínio): `{"message": "Link nao autorizado", "data": {"authorized": false}}` |

**Exemplo de chamada**

```bash
curl "https://api.exemplo.com/api/v1/links-reserva-publica/aB3xQ...32chars/reservas/65f1.../authorization" \
  -H "x-api-key: <AGENDAMENTOS_INTERNAL_API_KEY>"
```

## Referências

- Rotas: `src/applications/routes/public-reserve-link.route.ts`
- Controllers: `src/applications/controllers/public-reserve-link/public-reserve-link.controller.ts`
- Usecase: `src/domain/usecases/public-reserve-link/public-reserve-link.usecase.ts`
- Usecase de criação de reserva (reutilizado): `src/domain/usecases/reserva/create/create-reserve.usecase.ts` e `src/domain/usecases/shared/reserve-scheduling.validator.ts` — ver [`reserva.md`](./reserva.md)
- DTO: `src/applications/dto/public-reserve-link.dto.ts`, `src/applications/dto/create-reserva.dto.ts`
- Model: `src/domain/models/public-reserve-link.model.ts`
- Ports: `src/domain/ports/input/public-reserve-link.input-port.ts`, `src/domain/ports/output/public-reserve-link-persistence.port.ts`
- Adapters: `src/infra/adapters/public-reserve-link/{create,find-valid,claim,mark-used,release,authorize}/*.adapter.ts`
- Middlewares: `src/applications/middlewares/auth.middleware.ts`, `src/applications/middlewares/internal-api-key.middleware.ts`
