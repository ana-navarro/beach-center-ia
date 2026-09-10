# Mensagem (envio ativo por WhatsApp)

## Visão geral

O recurso `mensagem` (`/api/v1/messages`) é o **canal de envio ativo** (outbound) do serviço **beach-center-whatsapp** — outros microsserviços do ecossistema pedem a este serviço que **envie** uma mensagem de WhatsApp para um número. O serviço fala com a **Meta / WhatsApp Cloud API** (`graph.facebook.com/<versão>/<phone_number_id>/messages`) através do `WhatsAppService`.

`beach-center-whatsapp` é um serviço **Node/JavaScript (CommonJS)**, **isento** da Arquitetura Hexagonal e de testes automatizados (decisão de projeto). Não usa Firebase para estas rotas.

Há dois endpoints:

- `POST /messages/send-text` — o chamador manda o **texto pronto**.
- `POST /messages/send-by-key` — o chamador manda uma **`key` de template** + variáveis; o serviço resolve o texto pelo `MessageTemplateService` (banco `whatsapp_message_templates` com fallback para `src/domain/default-message-templates.js`) e interpola `{{var}}`. **É o endpoint usado pelo `beach-center-bff-agendamentos`** nas notificações de cancelamento/reagendamento por protocolo (task 006b) — o chamador nunca compõe texto.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `POST /messages/send-text` | `x-api-key` = `WHATSAPP_INTERNAL_API_KEY` **quando a env estiver definida**. Se a env **não** estiver definida, a rota fica aberta |
| `POST /messages/send-by-key` | idem |

> A verificação é feita **dentro do controller** (não há middleware): se `process.env.WHATSAPP_INTERNAL_API_KEY` existe e o header `x-api-key` não bate → `401 {"message":"Unauthorized"}`. Se a env não existe, o header é ignorado. No `docker-compose.dev` a chave é `dev-whatsapp-key` (compartilhada com `agendamentos` via `WHATSAPP_API_URL` + `WHATSAPP_INTERNAL_API_KEY`).

## Endpoints

### POST /api/v1/messages/send-text

Envia uma mensagem de texto **já pronta** para um número. Aciona `WhatsAppService.sendTextMessage(to, text)` — normaliza o telefone (só dígitos) e faz `POST` na Cloud API com `type: "text"`.

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `to` | string | Sim | Telefone destino (com DDI/DDD; caracteres não-numéricos são removidos) |
| `text` | string | Sim | Texto da mensagem |

**Resposta de sucesso** — `200 OK` (repassa **cru** o corpo da Cloud API)
```json
{
  "messaging_product": "whatsapp",
  "contacts": [{ "input": "5511999998888", "wa_id": "5511999998888" }],
  "messages": [{ "id": "wamid.HBg..." }]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `{"message":"to and text are required"}` — falta `to` ou `text` |
| 401 | `{"message":"Unauthorized"}` — `x-api-key` exigida e incorreta |
| 500 | `{"message":"Error sending WhatsApp text message"}` — falha na Cloud API ou env `WHATSAPP_ACCESS_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID` ausente |

**Exemplo de chamada**
```bash
curl -X POST http://whatsapp:5004/api/v1/messages/send-text \
  -H "x-api-key: $WHATSAPP_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{ "to": "5511999998888", "text": "Sua reserva foi confirmada!" }'
```

---

### POST /api/v1/messages/send-by-key

Resolve um template por `key`, interpola `vars` e envia. Aciona `MessageTemplateService.get(key, vars)`:

1. Busca o template `active: true` em `whatsapp_message_templates` (`composeBody` = `editable_body` + `fixed_body`).
2. Se não houver no banco, usa o **default** de `src/domain/default-message-templates.js` para aquela `key`.
3. Interpola `{{var}}` com `vars` (variável ausente vira string vazia).
4. Se o corpo final ficar vazio (key desconhecida) → **404**.
5. Envia via `WhatsAppService.sendTextMessage`.

**Templates usados pela task 006b** (ver [`modelo-mensagem.md`](modelo-mensagem.md)):

| `key` | Quando | Variáveis |
|---|---|---|
| `cancelamento-reembolso` | `agendamentos` cancela por protocolo uma reserva/partida que estava `approved` | `nome`, `protocolo`, `motivo` (opcional) |
| `reagendamento-confirmado` | `agendamentos` reagenda por protocolo com sucesso | `nome`, `protocolo`, `novo_dia`, `novo_horario`, `quadra` |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `to` | string | Sim | Telefone destino |
| `key` | string | Sim | Chave do template |
| `vars` | object | Não | Mapa `{ nome: valor }` para interpolar `{{nome}}` — default `{}` |

**Resposta de sucesso** — `200 OK`
```json
{ "data": { "messaging_product": "whatsapp", "messages": [{ "id": "wamid.HBg..." }] } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `{"message":"to and key are required"}` |
| 401 | `{"message":"Unauthorized"}` — `x-api-key` exigida e incorreta |
| 404 | `{"message":"Message template not found for key: <key>"}` — key sem template no banco nem nos defaults |
| 500 | `{"message":"Error sending WhatsApp message by key"}` — falha na Cloud API ou env ausente |

**Exemplo de chamada**
```bash
curl -X POST http://whatsapp:5004/api/v1/messages/send-by-key \
  -H "x-api-key: $WHATSAPP_INTERNAL_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "to": "5511999998888",
    "key": "reagendamento-confirmado",
    "vars": { "nome": "João", "protocolo": "0348172950", "novo_dia": "2026-10-25", "novo_horario": "19:00", "quadra": "Quadra 2" }
  }'
```

## Referências

- Rota: `src/applications/routes/message.route.js` (montada em `src/applications/routes/routes.js` como `/messages`, sob `/api/v1`)
- Controllers: `src/applications/controllers/messages/send-text.controller.js`, `src/applications/controllers/messages/send-by-key.controller.js`
- Serviços: `src/infra/services/whatsapp.service.js` (Cloud API), `src/infra/services/message-template.service.js` (resolução + interpolação de template)
- Templates default: `src/domain/default-message-templates.js`
- Seed idempotente no boot: `src/infra/seed/seed-message-templates.js` (chamado em `index.js` após `mongoose.connect`)
- Env: `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_GRAPH_API_VERSION` (default `v23.0`), `WHATSAPP_INTERNAL_API_KEY`
- Consumidor (outro serviço): `beach-center-bff-agendamentos` — `src/infra/adapters/whatsapp/send-whatsapp-notification.adapter.ts` (`POST {WHATSAPP_API_URL}/messages/send-by-key`); ver [`../beach-center-bff-agendamentos/reserva.md`](../beach-center-bff-agendamentos/reserva.md) e [`../beach-center-bff-agendamentos/ranking-agendamento.md`](../beach-center-bff-agendamentos/ranking-agendamento.md)
- Recursos irmãos: [`modelo-mensagem.md`](modelo-mensagem.md), [`webhook.md`](webhook.md)
