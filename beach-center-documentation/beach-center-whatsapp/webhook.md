# Webhook (recebimento de mensagens do WhatsApp)

## Visão geral

O recurso `webhook` (`/api/v1/webhook`) é o **canal de entrada** (inbound) do serviço **beach-center-whatsapp** — é o endpoint registrado na **Meta / WhatsApp Cloud API** para (1) validar a assinatura do webhook e (2) receber as mensagens que os clientes enviam. As mensagens de texto recebidas alimentam um **bot de atendimento por menus** (`ConversationService`), que responde pelo mesmo número via `WhatsAppService.sendTextMessage`.

O bot mantém o estado da conversa por telefone na coleção `conversation_sessions` (estados: `MAIN_MENU`, `RENTAL_MENU`, `WAITING_RESERVE_SEARCH`, `WAITING_RESERVE_SELECTION`, `RESERVE_ACTION_MENU`, `CONFIRM_CANCEL`). Todas as respostas saem de templates (ver [`modelo-mensagem.md`](modelo-mensagem.md)). O fluxo "Andamento de Reserva" consulta reservas direto no MongoDB (`ReserveService`) e o cancelamento chama `PATCH /reservas/protocol/:number/cancel` no `beach-center-bff-agendamentos`; "Realizar nova reserva" gera um link público via `POST /links-reserva-publica`.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `GET /webhook` | Nenhum middleware — valida `hub.verify_token` contra `WHATSAPP_VERIFY_TOKEN` |
| `POST /webhook` | Nenhum middleware — endpoint público chamado pela Meta |

> Não há verificação de assinatura `X-Hub-Signature` no código atual. O `POST` **sempre** responde `200` (mesmo em erro) para a Meta não reenfileirar o evento.

## Endpoints

### GET /api/v1/webhook

Handshake de verificação do webhook (a Meta chama uma vez ao configurar). Aciona `WhatsAppWebhookController.verify`.

**Parâmetros de query**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `hub.mode` | string | Sim | Precisa ser `subscribe` |
| `hub.verify_token` | string | Sim | Precisa bater com `WHATSAPP_VERIFY_TOKEN` |
| `hub.challenge` | string | Sim | Ecoado no corpo da resposta quando a validação passa |

**Resposta de sucesso** — `200 OK`, corpo = valor de `hub.challenge` (texto puro).

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 403 | `hub.mode` ≠ `subscribe` ou `hub.verify_token` incorreto |

**Exemplo de chamada**
```bash
curl "http://whatsapp:5004/api/v1/webhook?hub.mode=subscribe&hub.verify_token=$WHATSAPP_VERIFY_TOKEN&hub.challenge=1234"
```

---

### POST /api/v1/webhook

Recebe as notificações de mensagens. Aciona `WhatsAppWebhookController.handle`:

1. Extrai `entry[].changes[].value.messages[]` do payload da Cloud API.
2. Para cada mensagem com `from` + `text.body`: se `WHATSAPP_AUTO_REPLY_ENABLED !== 'false'`, chama `ConversationService.handleIncomingText(from, text)` e devolve a resposta ao remetente via `WhatsAppService.sendTextMessage`.
3. Responde `200` **sempre** (inclusive em exceção — logada no servidor).

**Corpo da requisição** — payload padrão da WhatsApp Cloud API (resumo):

| Campo | Tipo | Descrição |
|---|---|---|
| `entry[].changes[].value.messages[].from` | string | Telefone do remetente |
| `entry[].changes[].value.messages[].text.body` | string | Texto recebido |
| `entry[].changes[].value.messages[].type` | string | Tipo (`text`, ...) — só `text` é tratado |

**Resposta de sucesso** — `200 OK` (sem corpo).

**Exemplo de chamada**
```bash
curl -X POST http://whatsapp:5004/api/v1/webhook \
  -H "Content-Type: application/json" \
  -d '{ "entry": [ { "changes": [ { "value": { "messages": [ { "from": "5511999998888", "type": "text", "text": { "body": "oi" } } ] } } ] } ] }'
```

## Referências

- Rota: `src/applications/routes/webhook.route.js` (montada como `/webhook`, sob `/api/v1`)
- Controller: `src/applications/controllers/webhook/whatsapp-webhook.controller.js`
- Bot / máquina de estados: `src/infra/services/conversation.service.js` + `src/infra/schemas/conversation-session.schema.js` (coleção `conversation_sessions`)
- Serviços de apoio: `src/infra/services/reserve.service.js` (consulta/cancelamento de reserva), `src/infra/services/public-reserve-link.service.js` (link de nova reserva), `src/infra/services/whatsapp.service.js`, `src/infra/services/message-template.service.js`
- Schemas espelhados (somente leitura) de `agendamentos`: `src/infra/schemas/{reserve,scheduling,unit,court,user}.schema.js`
- Env: `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_AUTO_REPLY_ENABLED` (`'false'` desliga a auto-resposta), `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `AGENDAMENTOS_API_URL`, `AGENDAMENTOS_INTERNAL_API_KEY`, `AGENDAMENTOS_ADMIN_TOKEN`, `APP_SCHEDULING_URL`, `PUBLIC_RESERVE_LINK_EXPIRES_IN_HOURS`
- Recursos irmãos: [`mensagem.md`](mensagem.md), [`modelo-mensagem.md`](modelo-mensagem.md)
