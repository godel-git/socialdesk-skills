---
name: socialdesk-api
description: Use when building integrations, bots, or apps with the Socialdesk API - handling webhooks, understanding event types, authentication flow, or choosing between direct messages, hint messages, and HSM messages
---

# Socialdesk API

Socialdesk is a CPaaS platform for centralized messaging (WhatsApp, Instagram, Facebook Messenger). This skill covers the API concepts, webhook events, and integration architecture.

For SDK usage (`socialdesk-node`), see the **socialdesk-node** skill.

## Architecture

```
Participant (WhatsApp/IG) --> Socialdesk --> Webhook (your app)
                                  ^                |
                                  |                v
                          API response    Process event
```

1. Socialdesk sends webhook events to your app with an `access_token` (valid 2h)
2. Your app processes the event and calls the API using that token
3. No refresh flow needed — each webhook brings a fresh token

## Two Types of Apps

| Building a... | Primary message type | Description |
|---|---|---|
| **Chatbot** | Direct Message | Responds directly to the customer through the channel |
| **Agent assistant** | Hint Message | Shows info/suggestions only visible to the human agent |

## Base URL

```
https://api.socialdesk.cr/web-api
```

All endpoints require `Authorization: Bearer {access_token}` and `Content-Type: application/json`.

## Endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/messages/reply` | POST | Send hint messages (default) or direct messages (`send_as_direct_message: true`) |
| `/messages/HSM` | POST | Send template messages (HSM) |
| `/conversations/assign` | POST | Assign conversation to agent (`USER`) or team (`TEAM`) |
| `/conversations/labels/add` | POST | Add label to conversation |

See @api-reference.md for full parameter tables, payloads, and error codes.

## Webhook Event Types

| Event | Trigger | Key fields in payload |
|---|---|---|
| `DIRECT_MESSAGE` | Participant sends a message | `id`, `direction`, `sender`, `text`, `conversation` |
| `REPLY_MESSAGE` | App sends a hint message | `id`, `text`, `callbacks`, `quick_replies`, `conversation` |
| `REPLY_MESSAGE_CALLBACK` | Agent clicks a callback button | `id`, `label`, `value`, `conversation`, `message` |
| `HSM_MESSAGE` | Template message is sent | `id`, `text`, `rawText`, `parameters`, `conversation` |
| `CONVERSATION_FINALIZED` | Conversation is closed | `conversation.closedBySchedule`, `message` |

Every webhook event includes: `type`, `payload`, `channel_instance`, `instance` (your app info), `extension_settings`, `access_token`, `event_id`, `event_time`.

## Critical: Filter OUTGOING Messages

When your app calls `sendDirectMessage()`, Socialdesk generates a new `DIRECT_MESSAGE` event with `direction: OUTGOING`. If you don't filter these, your handler processes its own messages and creates an **infinite loop**.

```typescript
if (webhook.isEventType(EventType.DirectMessage)) {
  const payload = webhook.getPayload();
  // ALWAYS filter outgoing messages first
  if (payload.direction === MessageDirection.Outgoing) return;
  // ...handle incoming message
}
```

## application_context

Persist state across messages without your own database. Scoped per app — multiple apps coexist without conflicts.

- Send: include `application_context: { ... }` in any message
- Read: comes back in the next webhook at `payload.conversation.context.assistants.{app_instance_id}`
- Mutually exclusive with `context` field (use only `application_context`)

## extension_settings

Your app's global config, received in every webhook event under `extension_settings`. Use it to read values like team IDs, label keys, or custom config instead of hardcoding them.

Best practice: validate required settings on each webhook and return a clear error if missing.

## Handling Callbacks

When a hint message includes `callbacks`, each one renders as a button for the agent. When clicked, your app receives a `REPLY_MESSAGE_CALLBACK` event with the callback's `id` and `value`.

For prompt callbacks (configured as shortcuts in Socialdesk), the `value` is an object containing form data + `agentName` + `agentId`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Not filtering OUTGOING direct messages | Always check `payload.direction` — causes infinite webhook loops |
| Storing tokens long-term | Tokens expire in 2h. Always use the latest from webhook |
| Using `application_context` + `context` together | Mutually exclusive. Use only `application_context` |
| Sending `callbacks`/`quick_replies` with direct messages | Only work on hint messages |
| Forgetting `send_as_direct_message: true` | Without it, `/messages/reply` sends a hint message |
| Using `account_contact_id` AND `contact` in HSM | Pick one: existing contact or new contact object |
| Hardcoding team/label IDs | Read from `extension_settings` so each app instance can have different values |

## Error Format

```json
{ "status": 400, "error": "Descripcion del error" }
```

| Code | Description |
|---|---|
| `400` | Invalid request (missing fields or incorrect values) |
| `401` | Invalid or expired token |
| `403` | No permission for the requested resource |
| `404` | Resource not found |
