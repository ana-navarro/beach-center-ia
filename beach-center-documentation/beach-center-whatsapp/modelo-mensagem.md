# Modelo de mensagem (templates de WhatsApp)

## Visão geral

O recurso `modelo-mensagem` (`/api/v1/message-templates`) administra os **textos das mensagens** que o serviço **beach-center-whatsapp** envia — tanto as respostas automáticas do bot de atendimento (`webhook`) quanto as notificações ativas (`POST /messages/send-by-key`). Os templates ficam na coleção `whatsapp_message_templates`.

Cada template tem:

| Campo | Descrição |
|---|---|
| `key` | Identificador único (ex.: `main_menu`, `cancelamento-reembolso`) |
| `title` / `description` | Rótulos para a UI de administração |
| `editable_body` | Parte editável pelo admin (contém os `{{placeholders}}`) |
| `fixed_body` | Parte fixa (ex.: as opções numeradas de um menu) — não editável |
| `body` | Corpo composto (`editable_body` + `\n\n` + `fixed_body`) — recomputado a cada update |
| `editable` | `false` bloqueia edição pelo app (`403` no `PATCH`) |
| `variables` | Lista de variáveis suportadas (ex.: `["nome","protocolo"]`) |
| `active` | Só templates `active: true` são usados pelo `MessageTemplateService.get` |

**Seed idempotente:** no boot (`index.js` → `seedMessageTemplates()`), toda `key` de `src/domain/default-message-templates.js` que ainda não existe no banco é inserida (`$setOnInsert` — **nunca** sobrescreve o `editable_body` que o admin já ajustou). Se um template não está no banco, o `MessageTemplateService` cai para o default em código.

## Autenticação/autorização

| Rota | Guard |
|---|---|
| `GET /message-templates` | `authOrInternalApiKey` + `requireRole('ADMIN')` |
| `PATCH /message-templates/:key` | `authOrInternalApiKey` + `requireRole('ADMIN')` |

`authOrInternalApiKey`: aceita `x-api-key` = `WHATSAPP_INTERNAL_API_KEY` (marca `req.internalAuth = true`, que passa direto pelo `requireRole`) **ou** `Authorization: Bearer <idToken Firebase>` de um usuário `ADMIN` (`user_type === 'ADMIN'` na coleção `users`). Sem nenhum dos dois → `401`; Bearer de não-ADMIN → `403 {"message":"Acesso negado"}`.

## Endpoints

### GET /api/v1/message-templates

Lista todos os templates (ordenados por `key`). Cada item vem com um campo extra `preview` = corpo composto atual (`editable_body` + `fixed_body`).

**Resposta de sucesso** — `200 OK`
```json
{
  "data": [
    {
      "key": "cancelamento-reembolso",
      "title": "Cancelamento com reembolso por conversa",
      "description": "Enviado quando um agendamento pago (approved) e cancelado por protocolo (task 006b)",
      "editable": true,
      "editable_body": "Ola, {{nome}}! Sua reserva {{protocolo}} foi cancelada...",
      "fixed_body": "",
      "variables": ["nome", "protocolo"],
      "active": true,
      "preview": "Ola, {{nome}}! Sua reserva {{protocolo}} foi cancelada..."
    }
  ]
}
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 401 | Sem `x-api-key` válida nem Bearer |
| 403 | `{"message":"Acesso negado"}` — Bearer de não-ADMIN |

**Exemplo de chamada**
```bash
curl http://whatsapp:5004/api/v1/message-templates \
  -H "Authorization: Bearer $ID_TOKEN_ADMIN"
```

---

### PATCH /api/v1/message-templates/:key

Atualiza o **texto editável** de um template. Recompõe o `body` (`editable_body` + `fixed_body`) e devolve o registro com `preview`.

**Parâmetros de path**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `key` | string | Sim | Chave do template |

**Corpo da requisição**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `editable_body` | string | Sim | Novo texto editável (pode ser `""`). Se `undefined` → `400` |

**Resposta de sucesso** — `200 OK`
```json
{ "data": { "key": "cancelamento-reembolso", "editable_body": "Novo texto {{nome}}", "body": "Novo texto {{nome}}", "preview": "Novo texto {{nome}}", "...": "..." } }
```

**Erros possíveis**

| Código HTTP | Causa |
|---|---|
| 400 | `{"message":"Texto editavel nao informado"}` — `editable_body` ausente do corpo |
| 401 / 403 | Auth (ver acima) |
| 403 | `{"message":"Esta mensagem nao pode ser editada pelo app"}` — template com `editable: false` |
| 404 | `{"message":"Mensagem nao encontrada"}` — `key` inexistente no banco |

**Exemplo de chamada**
```bash
curl -X PATCH http://whatsapp:5004/api/v1/message-templates/cancelamento-reembolso \
  -H "Authorization: Bearer $ID_TOKEN_ADMIN" -H "Content-Type: application/json" \
  -d '{ "editable_body": "Ola, {{nome}}! A reserva {{protocolo}} foi cancelada. Responda aqui para tratarmos o reembolso." }'
```

## Referências

- Rota: `src/applications/routes/message-template.route.js` (montada como `/message-templates`, sob `/api/v1`)
- Controller: `src/applications/controllers/messages/message-template.controller.js`
- Middleware: `src/applications/middlewares/auth.middleware.js` (`authOrInternalApiKey`, `requireRole`)
- Serviço: `src/infra/services/message-template.service.js` (`composeBody`, `render`, `get`)
- Schema: `src/infra/schemas/message-template.schema.js` (coleção `whatsapp_message_templates`)
- Defaults + seed: `src/domain/default-message-templates.js`, `src/infra/seed/seed-message-templates.js`
- Env: `WHATSAPP_INTERNAL_API_KEY`
- Recursos irmãos: [`mensagem.md`](mensagem.md), [`webhook.md`](webhook.md)
