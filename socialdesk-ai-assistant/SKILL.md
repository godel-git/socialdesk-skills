---
name: socialdesk-ai-assistant
description: Use when the user wants to create, structure, or fix a custom prompt for the Socialdesk AI Assistant app (the chatbot configured via openAIPrompt). Guides users through required sections, assists building [[...]] commands, and validates command syntax. NOT for SDK/integration code.
---

# Socialdesk AI Assistant Prompt Builder

Helps any user — technical or not — author the custom prompt pasted into the Socialdesk "AI Assistant" app configuration (field `extensionSettings.openAIPrompt`). At runtime the text is wrapped between `{{CUSTOM_INSTRUCTIONS_START}} ... {{CUSTOM_INSTRUCTIONS_END}}` by `run-prompt.middleware.ts` and sent to OpenAI alongside a much larger parent system prompt.

## When to use

Use this skill when the user wants to:

- Write a new custom prompt for the Socialdesk AI Assistant app from scratch.
- Fix, review, or restructure an existing `openAIPrompt` draft.
- Add a `[[...]]` command (handoff, label, webhook, form callback, send email) to their prompt and doesn't know the exact syntax.
- Validate commands already present in a draft prompt.

Do NOT use this skill when the user wants to:

- Write SDK or integration code (webhook handlers, `WebhookHelper`, `SocialdeskClient` calls, Express/Lambda endpoints). Point them to the **socialdesk-node** skill.
- Build backend automations, cron jobs, or external services that talk to the Socialdesk API.
- Configure non-prompt fields of the app (channel IDs, tokens, model selection).

## What this produces

A Markdown file (`.md`) saved to the current working directory containing a single plain-text prompt block, organized in numbered sections, ready to copy into the `openAIPrompt` field of the AI Assistant extension settings. The file contents are the prompt and nothing else — no code, no JSON wrapper, no surrounding explanation inside the deliverable.

## What the parent prompt already covers (DO NOT repeat)

The parent system prompt in `/src/packages/basic-assistant/ai-assistant/middlewares/run-prompt.middleware.ts` (lines 52–341) is the source of truth and already enforces all of the following. The custom prompt must NOT restate them — doing so wastes tokens and can conflict with the parent rules:

- The JSON output shape the assistant must return (`text`, `commands`, `interactions`, `attachments`).
- How `[[...]]` command syntax works and where commands go in the response.
- "Don't invent information", no-loop behavior, not re-asking for data already provided.
- Current time and timezone context injection.
- When to use interactions (buttons/lists) vs. plain text.
- Attachment rules and markdown-to-plain-text conversion.

If the user's draft contains any of the above, remove it and explain that it's already enforced globally.

**Exception — WhatsApp character limits:** Although the parent prompt includes WhatsApp Cloud API limits, the model frequently violates them (e.g. row titles exceeding 24 chars cause error #131009). To reinforce compliance, the custom prompt MUST include a dedicated section with the exact character limits. See "WhatsApp interaction limits" in Step 6.

## Input types the bot can receive

The AI Assistant runtime automatically injects context when the user sends certain WhatsApp message types. The custom prompt does not need to handle parsing — the parent prompt takes care of that — but knowing these exist helps write better instructions for how the bot should react.

- **Location:** When a user shares a location, the bot receives `[User shared a location: latitude=<lat>, longitude=<lon>]`. If the business needs location-aware behavior (e.g. calculating delivery zones, confirming addresses), the custom prompt can include instructions for how to respond.
- **Orders:** When a user sends a WhatsApp catalog order, the bot receives `[User sent an order: {"items":[{"name":"...","quantity":N,"item_price":N,"currency":"..."}]}]`. If the business uses WhatsApp catalogs, the custom prompt can include instructions for how to process or confirm orders.

## Flow

**Language rule:** Conduct the Q&A in the same language the user is writing in. The final generated prompt should be in the same language as the user's target audience (usually the business's customers). The example below happens to be Spanish.

### Step 1 — Detect mode

- **Draft provided** → parse it. Identify which required sections are present, which are missing, and list every `[[...]]` command found. Run the validator (Step 5) on those commands before anything else. This is *reconstruct* mode.
- **Blank start** → guided Q&A. Ask one question at a time. Do not dump a template on the user. This is *guided* mode.

### Step 2 — Required sections loop

Always collect these four, in order, one question at a time:

1. **Role & identity** — company/brand name, tone and personality. Assume WhatsApp + Socialdesk unless the user explicitly mentions a different channel.
2. **Initial message** — how the bot greets, whether to offer a menu or open question.
3. **Knowledge base** — what the business does, main products/services, anything the bot must know to answer.
4. **Human handoff** — when to escalate and to whom. This is where the command builder (Step 4) typically kicks in for the first time.

Do not move on until each required section has an answer (or an explicit "skip" from the user).

### Step 3 — Optional sections

For each optional section, ask "¿quieres incluir X?" and only include it if the user says yes. Never assume.

- Business hours.
- Language(s) the bot should respond in.
- Command-driven flows (self-registration, form triggers, webhook calls).
- Free-form FAQs (location, pricing, policies — whatever the business wants).
- Business-specific restrictions (topics to avoid, disclaimers, compliance notes).

### Step 4 — Command builder

Non-technical users don't know `[[...]]` syntax. Ask plain-language questions and build the command behind the scenes. Always explain in plain language what the command will do *before* inserting it, and confirm.

| Intent | Ask | Emit |
|---|---|---|
| Hand off to agent | User or team? What's the ID? | `[[ASSIGN_TO:TEAM:<id>]]` or `[[ASSIGN_TO:USER:<id>]]` |
| Tag conversation | Which label key? | `[[ADD_LABEL:<key>]]` |
| Send data to external system | URL (no `&` in the URL itself), fields to send | `[[WEBHOOK:POST\|URL=...&payload={...}]]` with `<placeholder>` substitutions |
| Trigger a pre-filled form | Form name + fields to prefill | `[[ASSISTANT_CALLBACK:FORM_ASSISTANT\|PREFILLED_FORM_SUBMIT?formname=<name>&<field>=<value>]]` |
| Hand off to another AI assistant flow | Assistant id + step (advanced — ask user for raw values) | `[[ASSISTANT_CALLBACK:ASSIGN_ASSISTANT\|ASSIGN?id=<id>&step=<n>]]` |
| Send an email | Recipient(s), subject, body content | `[[SEND_EMAIL:to=<email>&subject=<asunto>&body=<cuerpo>]]` |

Placeholders like `<nombre>`, `<correo>`, `<telefono>` are substituted at runtime by the bot from chat/contact data — the user does not need to fill them in manually.

### Step 5 — Command validator

When the user provides a draft, extract every `[[...]]` and validate against the five valid command types from `shared/types.ts`. Use these exact grammars:

- `[[ASSIGN_TO:(USER|TEAM):<id>]]` — `<id>` is a non-empty string (typically a 24-char MongoDB ObjectId, but don't hard-require length). No spaces, no `]` inside the id.
- `[[ADD_LABEL:<key>]]` — `<key>` is a non-empty token, typically lowercase alphanumeric with optional hyphens/underscores.
- `[[WEBHOOK:<METHOD>|<params>]]` — `<METHOD>` is uppercase letters (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`). `<params>` is `key=value` pairs separated by `&`. Supported keys: `URL`, `payload`, `headers`. `payload` and `headers` are JSON values (object `{...}` or array `[...]`) and may contain `&` safely because the parser reads balanced JSON. **`URL` must NOT contain `&`** — the parser reads URL up to the next `&`, so any `&` in the URL truncates the command.
- `[[ASSISTANT_CALLBACK:<SUBTYPE>|<ACTION>?<querystring>]]` — `<SUBTYPE>` is `FORM_ASSISTANT` or `ASSIGN_ASSISTANT`; `<ACTION>` is `PREFILLED_FORM_SUBMIT` (for `FORM_ASSISTANT`) or `ASSIGN` (for `ASSIGN_ASSISTANT`); `<querystring>` is standard `key=value&key=value`.
- `[[SEND_EMAIL:to=<email>&subject=<subject>&body=<body>]]` — `to` is one or more email addresses separated by commas. `subject` is plain text. `body` is **always the last parameter** because it may contain `&` characters; the parser reads everything after `body=` to the closing `]]`. Body must be plain text only (no HTML). Only generate this command when the custom instructions explicitly require sending an email. Never invent recipient addresses — use only addresses from contact info, chat log, or custom instructions.

On any error: explain the problem in plain language, propose a corrected version, and confirm with the user before substituting. Example phrasing of a correction:

> Found `[[ASSING_TO:TEAM:abc]]` — looks like a typo in `ASSING_TO`. Did you mean `[[ASSIGN_TO:TEAM:abc]]`? I'll replace it if you confirm.

### Step 6 — Assemble & save to a Markdown file

Write the final prompt to a new `.md` file in the current working directory. Default filename: `socialdesk-prompt-<slug>.md` where `<slug>` is a short kebab-case identifier derived from the business name or bot purpose (e.g. `socialdesk-prompt-acme-ventas.md`). Ask the user to confirm or override the filename before writing.

The file content is the prompt itself — a single plain-text block organized in numbered sections in the style of the sales-bot example below. No preamble, no code fences, no markdown headings inside the prompt body (numbered uppercase section titles only, like `1. ROL DEL ASISTENTE`).

**WhatsApp interaction limits (MUST include):** Every generated prompt must contain a dedicated section (e.g. `N. LÍMITES DE INTERACCIONES WHATSAPP`) with these exact constraints. The parent prompt has them too, but the model frequently violates them, so redundancy is intentional here:

- Button title (`reply.title`): 1–20 characters. If longer, shorten to fit.
- Maximum buttons per message: 3.
- List menu button text (`action.button`): 1–20 characters.
- List row title (`row.title`): maximum 24 characters.
- List row description (`row.description`): maximum 72 characters.
- Maximum sections per list: 10.
- Maximum total rows across all sections: 10.
- Header text: maximum 60 characters.
- Footer text: maximum 60 characters.
- Body text with interactions: maximum 1000 characters.
- Body text without interactions: maximum 4000 characters.

**Number every instruction.** LLM agents follow numbered instructions more reliably than prose. Use two levels:

1. Top-level numbered sections: `1. ROL DEL ASISTENTE`, `2. MENSAJE INICIAL`, `3. BASE DE CONOCIMIENTO`, etc.
2. Inside each section, break rules and steps into numbered sub-items (`1.1`, `1.2`, ... or `- 1)`, `- 2)`, ...) instead of free-flowing paragraphs. One rule per line. Short imperative sentences.

Avoid long paragraphs of prose. If a section needs explanation, convert it to an enumerated list.

After writing the file, tell the user the absolute path and instruct them to copy its contents into the `openAIPrompt` field of the Socialdesk AI Assistant app configuration.

**Token budget:** Aim to keep the final prompt concise. If the knowledge base section grows beyond roughly 40-60 lines, ask the user whether everything is essential or if some content belongs elsewhere (FAQs, help docs, internal wikis).

## Command reference

Cheat sheet of the five valid command families. These are the only commands the AI Assistant runtime recognizes.

- **Assign conversation** — hand off to a specific user or team.
  - `[[ASSIGN_TO:USER:<id>]]`
  - `[[ASSIGN_TO:TEAM:<id>]]`
- **Add label** — tag the conversation with a label key.
  - `[[ADD_LABEL:<key>]]`
- **Call a webhook** — POST (or other method) to an external URL with an optional payload and headers.
  - `[[WEBHOOK:POST|URL=<url>&payload={...}&headers={...}]]`
  - **WEBHOOK grammar notes:** the method (`POST`, `GET`, etc.) must be uppercase. The `|` between method and params is literal. Supported keys: `URL`, `payload`, `headers`. `payload` and `headers` are JSON (`{...}` or `[...]`) and the parser reads balanced JSON, so `&` inside JSON strings is safe. **`URL` must NOT contain `&`** — the parser reads `URL` up to the next `&`, so any `&` in the URL itself will truncate the command. If your webhook URL has a query string with `&`, remove the query params or move them into `payload`.
- **Assistant callback — trigger a form** — launch a form flow, optionally prefilling fields.
  - `[[ASSISTANT_CALLBACK:FORM_ASSISTANT|PREFILLED_FORM_SUBMIT?formname=<name>&field=<value>]]`
- **Assistant callback — hand off to another assistant** — jump into a specific IVR/assistant step.
  - `[[ASSISTANT_CALLBACK:ASSIGN_ASSISTANT|ASSIGN?id=<id>&step=<n>]]`
- **Send email** — send an email via SendGrid. Only use when the custom instructions explicitly require it.
  - `[[SEND_EMAIL:to=<email>&subject=<subject>&body=<body>]]`
  - **SEND_EMAIL grammar notes:** `to` accepts multiple addresses separated by commas. `body` is **always the last parameter** — the parser reads everything after `body=` until `]]`, so `&` inside the body text is safe. Body must be plain text (no HTML). Never invent recipient addresses — use only addresses from the contact, chat log, or custom instructions.

Placeholders wrapped in angle brackets (`<nombre>`, `<correo>`, `<telefono>`, etc.) are replaced by the bot at runtime from chat and contact data.

## Command-building guidance for non-technical users

When a command needs an ID, URL, or form name the user doesn't know, ask plainly and never invent a value.

- **Team or user ID:** "¿Sabes el ID del equipo o usuario al que debe asignarse la conversación? Si no lo tienes a mano, puedes encontrarlo en la configuración de Socialdesk (equipos/usuarios). No te inventes un ID — mejor dejarlo pendiente y lo completamos después."
- **Webhook URL:** "¿Cuál es la URL completa del webhook (empieza con `https://`)? ¿Necesita algún header de autenticación, por ejemplo un `Authorization: Bearer ...`?"
- **Webhook payload:** "¿Qué campos quieres enviar? Los que vienen del chat (nombre del contacto, teléfono, correo) puedo dejarlos como `<nombre>`, `<telefono>`, `<correo>` y el bot los rellena solo."
- **Label key:** "¿Qué etiqueta quieres agregar? Usa el *key* exacto como aparece en Socialdesk (por ejemplo `seguimientopendiente`, sin espacios ni mayúsculas)."
- **Form name:** "¿Cómo se llama el formulario en Socialdesk? Es el nombre interno, no el título visible."
- **Email recipient:** "¿A qué dirección de correo se debe enviar? Si son varias, sepáralas por coma. Solo usa correos reales que conozcas — no te inventes ninguno."
- **Email content:** "¿Cuál es el asunto del correo? ¿Y qué debe decir el cuerpo? Puedo usar `<nombre>`, `<telefono>`, `<correo>` como datos del contacto que se rellenan automáticamente."

If the user doesn't have a value, leave a clearly marked `TODO` in the draft (e.g. `[[ASSIGN_TO:TEAM:TODO_TEAM_ID]]`) and remind them to replace it before saving. Never invent a plausible-looking ID or URL.

## Example output (abbreviated)

The following is an abbreviated example of what the final deliverable should look like. Numbered sections, Spanish, plain text, business-specific content only.

```
1. ROL DEL ASISTENTE
  1.1 Eres el asistente virtual de "Muebles del Valle" en WhatsApp, integrado con Socialdesk.
  1.2 Tu tono es cercano, servicial y breve.
  1.3 Solo respondes temas relacionados con la tienda.
  1.4 Si el usuario escribe en otro idioma, continúa en ese idioma.

2. MENSAJE INICIAL
  2.1 Saluda por el nombre del contacto si está disponible.
  2.2 Presenta tres opciones mediante lista interactiva:
      - Ver catálogo
      - Consultar una orden
      - Hablar con un asesor

3. BASE DE CONOCIMIENTO
  3.1 Vendemos muebles de sala, comedor y dormitorio, fabricados en Costa Rica.
  3.2 Horario de tienda física: lunes a sábado, 9am a 6pm.
  3.3 Envíos a todo el país en 3 a 5 días hábiles.
  3.4 Métodos de pago: SINPE, tarjeta y transferencia bancaria.

4. DERIVACIÓN A HUMANOS
  4.1 Deriva cuando el cliente pida un asesor, mencione un reclamo, o pregunte algo que no esté en la base de conocimiento.
  4.2 Ejecuta los comandos:
      [[ADD_LABEL:seguimientopendiente]]
      [[ASSIGN_TO:TEAM:<team_id>]]
  4.3 Luego responde: "Un asesor te atenderá en unos minutos."

5. LÍMITES DE INTERACCIONES WHATSAPP
  5.1 Título de botón: máximo 20 caracteres.
  5.2 Máximo 3 botones por mensaje.
  5.3 Texto del botón de lista: máximo 20 caracteres.
  5.4 Título de fila de lista: máximo 24 caracteres.
  5.5 Descripción de fila de lista: máximo 72 caracteres.
  5.6 Máximo 10 secciones por lista, máximo 10 filas en total.
  5.7 Texto de header: máximo 60 caracteres.
  5.8 Texto de footer: máximo 60 caracteres.
  5.9 Texto del cuerpo con interacciones: máximo 1000 caracteres.
  5.10 Texto del cuerpo sin interacciones: máximo 4000 caracteres.
  5.11 Si un título excede el límite, recórtalo para que quepa.
```

*Nota: los IDs entre `<>` son placeholders — reemplaza por los reales de tu cuenta de Socialdesk.*

Keep the final deliverable in this style: numbered top-level sections with numbered sub-items (`N.M`), short imperative sentences, commands inlined where they belong.

## Anti-patterns

- Do not repeat anything the parent prompt already covers (output format, command syntax rules, no-loop, attachments, markdown — see the dedicated section above) **except** WhatsApp character limits, which must always be included as a dedicated section for reinforcement.
- Do not assume optional sections. Ask before including business hours, FAQs, language rules, or restrictions.
- Do not invent team IDs, user IDs, webhook URLs, form names, or label keys. Ask the user; if unknown, leave a `TODO` placeholder.
- Do not output malformed `[[...]]` commands. Every command must match one of the five valid families in the command reference.
- Do not write SDK code, webhook handlers, or backend logic — that belongs in the **socialdesk-node** skill.
- Do not wrap the final deliverable in code fences or add preamble/closing text inside the pasteable block.
