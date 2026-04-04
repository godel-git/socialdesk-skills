---
name: socialdesk-node
description: Use when writing code with the socialdesk-node SDK - creating webhook handlers, sending messages (sendHintMessage, sendDirectMessage, sendHSMMessage), assigning conversations, adding labels, or scaffolding a Socialdesk integration project
---

# socialdesk-node SDK

Official Node.js/TypeScript SDK for the Socialdesk API. Provides `WebhookHelper` for parsing events and `SocialdeskClient` for calling endpoints.

For API concepts, webhook events, and architecture, see the **socialdesk-api** skill.

## Install

```bash
npm install socialdesk-node
```

## Core Pattern

Every Socialdesk integration follows this pattern:

```typescript
import { WebhookHelper, EventType, MessageDirection } from 'socialdesk-node';

async function handleWebhook(body: any) {
  const webhook = new WebhookHelper(body);
  const client = webhook.createClient(); // pre-authenticated with event's token

  if (webhook.isEventType(EventType.DirectMessage)) {
    const payload = webhook.getPayload();
    if (payload.direction === MessageDirection.Outgoing) return; // CRITICAL: avoid infinite loop
    // ...handle incoming message
  }
}
```

## WebhookHelper

Parses webhook event body. All getter methods work across event types.

```typescript
const webhook = new WebhookHelper(req.body);

webhook.getEventType()             // EventType enum value
webhook.getPayload()               // Raw typed payload
webhook.getAccessToken()           // Bearer token (valid 2h)
webhook.getConversationId()        // Works for ANY event type
webhook.getMessageId()             // Works for ANY event type
webhook.getMessageText()           // Works for ANY event type
webhook.getChannelInstance()       // Channel ID (needed for HSM)
webhook.getInstanceId()            // Your app instance ID
webhook.getSettings<T>()           // Parsed app settings (auto JSON.parse)
webhook.getConversationContext<T>() // Full conversation context
webhook.getApplicationContext<T>()  // Your app's persisted context
webhook.isEventType(EventType.X)   // Type check
webhook.createClient(baseURL?)     // SocialdeskClient (default: https://api.socialdesk.cr)
```

## SocialdeskClient Methods

### sendHintMessage — Info visible only to agent

```typescript
await client.sendHintMessage({
  text: 'Client asked about pricing',              // required (or attachments)
  message_id: webhook.getMessageId(),               // required
  conversation_id: webhook.getConversationId(),      // required
  callbacks: [                                       // optional: action buttons
    { id: 'cb_1', label: 'Send quote', value: 'send_quote' },
  ],
  quick_replies: [                                   // optional: suggested replies
    { text: 'Basic plan is $79/month.' },
  ],
  attachments: [{ type: 'IMAGE', url: '...', fileName: '...' }],  // optional
  interactions: [{ label: 'whatsapp', config: { ... } }],          // optional
  application_context: { intent: 'pricing', step: 1 },             // optional: persisted state
});
```

### sendDirectMessage — Message to participant

Requires "automatic replies" permission. Without it, silently falls back to hint message.

```typescript
await client.sendDirectMessage({
  text: 'Hello! An advisor will help you shortly.',
  message_id: webhook.getMessageId(),
  conversation_id: webhook.getConversationId(),
  interactions: [{                                   // optional: WhatsApp buttons
    label: 'whatsapp',
    config: {
      type: 'button',
      action: {
        buttons: [
          { type: 'reply', reply: { id: 'yes', title: 'Yes please' } },
          { type: 'reply', reply: { id: 'no', title: 'No thanks' } },
        ],
      },
    },
  }],
  application_context: { auto_replied: true },       // optional
});
```

Note: `callbacks` and `quick_replies` are NOT supported on direct messages.

### sendHSMMessage — Template message (WhatsApp)

Use `account_contact_id` (existing contact) OR `contact` (new), never both.

```typescript
// Existing contact
await client.sendHSMMessage({
  channel_id: webhook.getChannelInstance(),
  account_contact_id: 'ac_123',
  raw_text: 'Hello {{1}}, your appointment is on {{2}}',
  parameters: ['Maria', 'March 25'],
  external_template_id: 'appointment_v1',
});

// New contact
await client.sendHSMMessage({
  channel_id: webhook.getChannelInstance(),
  contact: { account_id: 'acc_001', first_name: 'Maria', phone: '+50688881234' },
  raw_text: 'Welcome {{1}}!',
  parameters: ['Maria'],
  external_template_id: 'welcome_v1',
});

// Scheduled
await client.sendHSMMessage({
  channel_id: webhook.getChannelInstance(),
  account_contact_id: 'ac_123',
  raw_text: 'Reminder: your appointment is tomorrow at {{1}}',
  parameters: ['3:00 PM'],
  external_template_id: 'reminder_v1',
  send_at: '2026-03-25T08:00:00Z',      // ISO 8601, must be future
});
```

### assignConversation

```typescript
await client.assignConversation({
  conversation_id: webhook.getConversationId(),
  entity: 'TEAM',           // 'USER' or 'TEAM'
  entity_id: 'team_sales',
  account_id: 'acc_001',    // optional for apps
  force_assign: true,        // optional: reassign even if already assigned
});
```

### addLabel

```typescript
await client.addLabel({
  conversation_id: webhook.getConversationId(),
  labelKey: 'qualified_lead',
  account_id: 'acc_001',    // optional for apps
});
```

## Complete Example: Express Webhook

```typescript
import express from 'express';
import { WebhookHelper, EventType, MessageDirection } from 'socialdesk-node';

const app = express();
app.use(express.json());

app.post('/webhook', async (req, res) => {
  const webhook = new WebhookHelper(req.body);
  const client = webhook.createClient();
  const settings = req.body.extension_settings ?? {};

  if (webhook.isEventType(EventType.DirectMessage)) {
    const payload = webhook.getPayload();
    if (payload.direction === MessageDirection.Outgoing) {
      return res.json({ ok: true }); // Skip own messages
    }

    const text = webhook.getMessageText().toLowerCase();
    const ctx = webhook.getApplicationContext<{ step?: number }>();

    await client.sendHintMessage({
      text: `Message received: "${text}"`,
      message_id: webhook.getMessageId(),
      conversation_id: webhook.getConversationId(),
      callbacks: [{ id: 'cb_reply', label: 'Auto-reply', value: 'auto_reply' }],
      application_context: { step: (ctx.step ?? 0) + 1 },
    });
  }

  if (webhook.isEventType(EventType.ReplyMessageCallback)) {
    const payload = webhook.getPayload();

    if (payload.id === 'cb_reply') {
      await client.sendDirectMessage({
        text: 'Thanks for your message! We will get back to you soon.',
        message_id: payload.message.id,
        conversation_id: payload.conversation.id,
      });
    }
  }

  if (webhook.isEventType(EventType.ConversationFinalized)) {
    console.log('Conversation closed:', webhook.getConversationId());
  }

  res.json({ ok: true });
});

app.listen(3000);
```

## AWS Lambda Example

```typescript
import { WebhookHelper, EventType, MessageDirection } from 'socialdesk-node';

export const handler = async (event: any) => {
  const body = JSON.parse(event.body);
  const webhook = new WebhookHelper(body);
  const client = webhook.createClient();

  if (webhook.isEventType(EventType.DirectMessage)) {
    if (webhook.getPayload().direction === MessageDirection.Outgoing) {
      return { statusCode: 200, body: JSON.stringify({ ok: true }) };
    }
    await client.sendDirectMessage({
      text: 'Thanks for contacting us!',
      message_id: webhook.getMessageId(),
      conversation_id: webhook.getConversationId(),
    });
  }

  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
};
```

## Exported Types

```typescript
import {
  // Enums
  EventType,                    // DirectMessage, ReplyMessage, ReplyMessageCallback, HSMMessage, ConversationFinalized
  MessageDirection,             // Incoming, Outgoing
  SenderType,                   // Participant, Agent, Extension
  AttachmentType,               // IMAGE, VIDEO, AUDIO, TEXT, CONTACTS
  // Webhook
  WebhookEvent, WebhookPayload,
  DirectMessagePayload, ReplyMessagePayload, ReplyMessageCallbackPayload,
  HSMMessagePayload, ConversationFinalizedPayload,
  // Shared
  MessageSender, Participant, Agent, Attachment, Interaction, QuickReply, Callback,
  // API options
  HintMessageOptions, DirectMessageOptions, HSMMessageOptions,
  AssignConversationOptions, AddLabelOptions,
} from 'socialdesk-node';
```

## Project Scaffold

```
my-socialdesk-app/
  src/
    index.ts                # Express/Lambda entry + webhook route
    handlers/
      direct-message.ts     # DIRECT_MESSAGE handler
      callback.ts           # REPLY_MESSAGE_CALLBACK handler
      finalized.ts          # CONVERSATION_FINALIZED handler
  package.json
  tsconfig.json
```

Or clone the official demo:
```bash
git clone https://github.com/godel-git/socialdesk-demo-app.git
```
