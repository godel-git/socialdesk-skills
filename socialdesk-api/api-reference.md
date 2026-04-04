# Socialdesk API Reference

Base URL: `https://api.socialdesk.cr/web-api`

All endpoints require `Authorization: Bearer {access_token}` and `Content-Type: application/json`.

---

## POST /messages/reply

Sends a hint message (default) or direct message (`send_as_direct_message: true`).

### Hint Message Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | Yes* | Message text visible to agent |
| `attachments` | array | Yes* | Files: `{ type, url, fileName }`. Types: `IMAGE`, `VIDEO`, `AUDIO`, `TEXT`, `CONTACTS` |
| `message_id` | string | Yes | ID of message being replied to (`payload.id`) |
| `conversation_id` | string | Yes | Conversation ID (`payload.conversation.id`) |
| `callbacks` | array | No | Action buttons: `{ id, label, value }` |
| `quick_replies` | array | No | Suggested replies: `{ text, attachments? }` |
| `interactions` | array | No | WhatsApp buttons/lists (see below) |
| `application_context` | object | No | Persisted per-app context |

*At least `text` or `attachments` required.

### Direct Message Parameters

Same as hint message plus:

| Field | Type | Required | Description |
|---|---|---|---|
| `send_as_direct_message` | boolean | Yes | Must be `true` |

Note: `callbacks` and `quick_replies` are ignored for direct messages. If the direct message permission is not enabled, the message falls back to a hint message.

### Interactions (WhatsApp)

Buttons:
```json
{
  "label": "whatsapp",
  "config": {
    "type": "button",
    "action": {
      "buttons": [
        { "type": "reply", "reply": { "id": "opt_1", "title": "Option 1" } },
        { "type": "reply", "reply": { "id": "opt_2", "title": "Option 2" } }
      ]
    }
  }
}
```

Lists:
```json
{
  "label": "whatsapp",
  "config": {
    "type": "list",
    "action": {
      "button": "View options",
      "sections": [
        {
          "title": "Section 1",
          "rows": [
            { "id": "1", "title": "Option A" },
            { "id": "2", "title": "Option B" }
          ]
        }
      ]
    }
  }
}
```

### Errors

| Code | Error | Cause |
|---|---|---|
| 400 | `You must send text or attachment` | Missing both `text` and `attachments` |
| 400 | `Message is required` | Missing `message_id` |
| 400 | `Conversation is required` | Missing `conversation_id` |
| 400 | `Cannot send both context and application_context` | Mutually exclusive fields |
| 401 | `Unauthorized` | Invalid or expired token |
| 403 | `Only app extensions can use application_context` | Only `APP` type instances can use this |

---

## POST /messages/HSM

Sends a template message (HSM) to a participant. Identify recipient with `account_contact_id` (existing) OR `contact` (new), not both.

### Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `channel_id` | string | Yes | Channel ID (from webhook `channel_instance`) |
| `account_contact_id` | string | Conditional | Existing contact ID |
| `contact` | object | Conditional | New contact: `{ account_id, phone, first_name?, last_name?, email?, image_url? }` |
| `raw_text` | string | Yes | Template text with `{{N}}` placeholders |
| `parameters` | array | No | Values for placeholders |
| `external_template_id` | string | No | WhatsApp Business template ID |
| `assign_entity` | string | No | `USER` or `TEAM` |
| `assign_entity_id` | string | No | Entity ID to assign |
| `conversation_id` | string | No | Existing conversation ID |
| `send_at` | string | No | Scheduled send time (ISO 8601). Omit for immediate |

### Errors

| Code | Error | Cause |
|---|---|---|
| 400 | `You must send text` | Missing `raw_text` |
| 400 | `You must include a channel_id` | Missing `channel_id` |
| 400 | `You must include an existing account_contact_id or new contact info` | Missing recipient |
| 400 | `Invalid entity parameter` | `assign_entity` not `USER` or `TEAM` |
| 400 | `Sent at can not be a past date` | `send_at` is in the past |
| 401 | `Unauthorized` | Invalid or expired token |
| 403 | `Forbidden` | No permission for channel/contact |

---

## POST /conversations/assign

Assigns a conversation to an agent or team.

### Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `conversation_id` | string | Yes | Conversation ID |
| `account_id` | string | Yes | Account ID |
| `entity` | string | Yes | `USER` (agent) or `TEAM` |
| `entity_id` | string | Yes | Agent or team ID |
| `force_assign` | boolean | No | Force reassignment (default: `false`) |

### Errors

| Code | Error | Cause |
|---|---|---|
| 400 | `conversation_id is required` | Missing `conversation_id` |
| 400 | `invalid entity parameter` | `entity` not `USER` or `TEAM` |
| 400 | `invalid entity_id parameter` | Missing `entity_id` |
| 401 | `Unauthorized` | Invalid or expired token |
| 403 | `Forbidden` | No permission |

---

## POST /conversations/labels/add

Adds a label to a conversation. Labels must be pre-created in Socialdesk.

### Parameters

| Field | Type | Required | Description |
|---|---|---|---|
| `conversation_id` | string | Yes | Conversation ID |
| `account_id` | string | No | Account ID (inferred for apps) |
| `labelKey` | string | Yes | Label key |

### Errors

| Code | Error | Cause |
|---|---|---|
| 401 | `Unauthorized` | Invalid or expired token |
| 403 | `Forbidden` | No permission |

---

## Webhook Event Payloads

### DIRECT_MESSAGE

```json
{
  "type": "DIRECT_MESSAGE",
  "payload": {
    "id": "msg_001",
    "direction": "INCOMING",
    "sender": { "name": "...", "picture": "...", "profile_id": "...", "type": "PARTICIPANT" },
    "text": "...",
    "attachments": [],
    "interactions": [],
    "send_at": "2026-03-23T10:30:00Z",
    "conversation": {
      "id": "conv_456",
      "topic": "...",
      "participants": [{ "id": "...", "name": "...", "picture": "..." }],
      "agents": [{ "id": "...", "name": "...", "picture": "..." }],
      "context": { "assistants": { "app_id": { /* your app context */ } } }
    }
  },
  "channel_instance": "ch_123",
  "instance": { "type": "APP", "id": "app_456", "settings": "{...}" },
  "extension_settings": { "webhook": "https://...", /* custom settings */ },
  "access_token": "eyJ...",
  "event_id": "evt_789",
  "event_time": "2026-03-23T10:30:00Z"
}
```

Sender types: `PARTICIPANT`, `AGENT`, `EXTENSION`
Direction: `INCOMING` (from participant), `OUTGOING` (from agent)

### REPLY_MESSAGE_CALLBACK

```json
{
  "type": "REPLY_MESSAGE_CALLBACK",
  "payload": {
    "conversation": { "id": "conv_456", "context": {} },
    "id": "cb_001",
    "label": "Send quote",
    "value": "send_quote",
    "message": { "id": "msg_001", "direction": "INCOMING", "text": "...", "attachments": [], "send_at": "..." }
  }
}
```

For prompt callbacks (shortcuts), `value` is an object with form data + `agentName` + `agentId`.

### CONVERSATION_FINALIZED

```json
{
  "type": "CONVERSATION_FINALIZED",
  "payload": {
    "id": "conv_456",
    "conversation": { "closedBySchedule": true, "id": "conv_456", "context": {} },
    "message": { "id": "msg_last", "direction": "INCOMING", "text": "...", "send_at": "..." }
  }
}
```

`closedBySchedule: true` = auto-closed by inactivity.


