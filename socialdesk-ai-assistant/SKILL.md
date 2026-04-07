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
- Add a `[[...]]` command (handoff, label, webhook, form callback) to their prompt and doesn't know the exact syntax.
- Validate commands already present in a draft prompt.

Do NOT use this skill when the user wants to:

- Write SDK or integration code (webhook handlers, `WebhookHelper`, `SocialdeskClient` calls, Express/Lambda endpoints). Point them to the **socialdesk-node** skill.
- Build backend automations, cron jobs, or external services that talk to the Socialdesk API.
- Configure non-prompt fields of the app (channel IDs, tokens, model selection).

## What this produces

A single plain-text block, organized in numbered sections, ready to paste into the `openAIPrompt` field of the AI Assistant extension settings. Nothing else — no code, no JSON wrapper, no surrounding explanation inside the deliverable itself.

## What the parent prompt already covers (DO NOT repeat)

The parent system prompt in `/src/packages/basic-assistant/ai-assistant/middlewares/run-prompt.middleware.ts` (lines 52–341) is the source of truth and already enforces all of the following. The custom prompt must NOT restate them — doing so wastes tokens and can conflict with the parent rules:

- The JSON output shape the assistant must return (`text`, `commands`, `interactions`, `attachments`).
- How `[[...]]` command syntax works and where commands go in the response.
- WhatsApp Cloud API rules: button counts, list counts, title and description length limits.
- "Don't invent information", no-loop behavior, not re-asking for data already provided.
- Current time and timezone context injection.
- When to use interactions (buttons/lists) vs. plain text.
- Attachment rules and markdown-to-plain-text conversion.

If the user's draft contains any of the above, remove it and explain that it's already enforced globally.

## Flow

**Language rule:** Conduct the Q&A in the same language the user is writing in. The final generated prompt should be in the same language as the user's target audience (usually the business's customers). The example below happens to be Spanish.

### Step 1 — Detect mode

- **Draft provided** → parse it. Identify which required sections are present, which are missing, and list every `[[...]]` command found. Run the validator (Step 5) on those commands before anything else. This is *reconstruct* mode.
- **Blank start** → guided Q&A. Ask one question at a time. Do not dump a template on the user. This is *guided* mode.

### Step 2 — Required sections loop

Always collect these four, in order, one question at a time:

1. **Role & identity** — company/brand name, channel (WhatsApp, web, etc.), tone and personality.
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

Placeholders like `<nombre>`, `<correo>`, `<telefono>` are substituted at runtime by the bot from chat/contact data — the user does not need to fill them in manually.

### Step 5 — Command validator

When the user provides a draft, extract every `[[...]]` and validate against the four valid command types from `shared/types.ts`. Use these exact grammars:

- `[[ASSIGN_TO:(USER|TEAM):<id>]]` — `<id>` is a non-empty string (typically a 24-char MongoDB ObjectId, but don't hard-require length). No spaces, no `]` inside the id.
- `[[ADD_LABEL:<key>]]` — `<key>` is a non-empty token, typically lowercase alphanumeric with optional hyphens/underscores.
- `[[WEBHOOK:<METHOD>|<params>]]` — `<METHOD>` is uppercase letters (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`). `<params>` is `key=value` pairs separated by `&`. Supported keys: `URL`, `payload`, `headers`. `payload` and `headers` are JSON values (object `{...}` or array `[...]`) and may contain `&` safely because the parser reads balanced JSON. **`URL` must NOT contain `&`** — the parser reads URL up to the next `&`, so any `&` in the URL truncates the command.
- `[[ASSISTANT_CALLBACK:<SUBTYPE>|<ACTION>?<querystring>]]` — `<SUBTYPE>` is `FORM_ASSISTANT` or `ASSIGN_ASSISTANT`; `<ACTION>` is `PREFILLED_FORM_SUBMIT` (for `FORM_ASSISTANT`) or `ASSIGN` (for `ASSIGN_ASSISTANT`); `<querystring>` is standard `key=value&key=value`.

On any error: explain the problem in plain language, propose a corrected version, and confirm with the user before substituting. Example phrasing of a correction:

> Found `[[ASSING_TO:TEAM:abc]]` — looks like a typo in `ASSING_TO`. Did you mean `[[ASSIGN_TO:TEAM:abc]]`? I'll replace it if you confirm.

### Step 6 — Assemble & present

Deliver the final prompt as a single plain-text block, organized in numbered sections in the style of the sales-bot example below. No preamble inside the block, no code fences inside the block, no restatement of parent-prompt rules.

**Token budget:** Aim to keep the final prompt concise. If the knowledge base section grows beyond roughly 40-60 lines, ask the user whether everything is essential or if some content belongs elsewhere (FAQs, help docs, internal wikis).

## Command reference

Cheat sheet of the four valid command families. These are the only commands the AI Assistant runtime recognizes.

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

Placeholders wrapped in angle brackets (`<nombre>`, `<correo>`, `<telefono>`, etc.) are replaced by the bot at runtime from chat and contact data.

## Command-building guidance for non-technical users

When a command needs an ID, URL, or form name the user doesn't know, ask plainly and never invent a value.

- **Team or user ID:** "¿Sabes el ID del equipo o usuario al que debe asignarse la conversación? Si no lo tienes a mano, puedes encontrarlo en la configuración de Socialdesk (equipos/usuarios). No te inventes un ID — mejor dejarlo pendiente y lo completamos después."
- **Webhook URL:** "¿Cuál es la URL completa del webhook (empieza con `https://`)? ¿Necesita algún header de autenticación, por ejemplo un `Authorization: Bearer ...`?"
- **Webhook payload:** "¿Qué campos quieres enviar? Los que vienen del chat (nombre del contacto, teléfono, correo) puedo dejarlos como `<nombre>`, `<telefono>`, `<correo>` y el bot los rellena solo."
- **Label key:** "¿Qué etiqueta quieres agregar? Usa el *key* exacto como aparece en Socialdesk (por ejemplo `seguimientopendiente`, sin espacios ni mayúsculas)."
- **Form name:** "¿Cómo se llama el formulario en Socialdesk? Es el nombre interno, no el título visible."

If the user doesn't have a value, leave a clearly marked `TODO` in the draft (e.g. `[[ASSIGN_TO:TEAM:TODO_TEAM_ID]]`) and remind them to replace it before saving. Never invent a plausible-looking ID or URL.

## Example output (abbreviated)

The following is an abbreviated example of what the final deliverable should look like. Numbered sections, Spanish, plain text, business-specific content only.

```
1. ROL DEL ASISTENTE
Eres el asistente virtual de "Muebles del Valle" en WhatsApp. Tu tono es
cercano, servicial y breve. Solo respondes temas relacionados con la tienda.

2. MENSAJE INICIAL
Saluda por el nombre del contacto si está disponible y ofrece tres opciones:
ver catálogo, consultar una orden existente, o hablar con un asesor.

3. BASE DE CONOCIMIENTO
- Vendemos muebles de sala, comedor y dormitorio, fabricados en Costa Rica.
- Horario de tienda física: lunes a sábado, 9am a 6pm.
- Envíos a todo el país en 3 a 5 días hábiles.
- Métodos de pago: SINPE, tarjeta y transferencia bancaria.

4. DERIVACIÓN A HUMANOS
Si el cliente pide hablar con un asesor, si menciona un reclamo, o si pregunta
algo que no está en la base de conocimiento, agrega la etiqueta de seguimiento
y asigna la conversación al equipo de ventas:
[[ADD_LABEL:seguimientopendiente]]
[[ASSIGN_TO:TEAM:<team_id>]]
Luego responde: "Un asesor te atenderá en unos minutos."
```

*Nota: los IDs entre `<>` son placeholders — reemplaza por los reales de tu cuenta de Socialdesk.*

Keep the final deliverable in this style: numbered H2-equivalent sections, short paragraphs, commands inlined where they belong.

## Anti-patterns

- Do not repeat anything the parent prompt already covers (output format, command syntax rules, WhatsApp limits, no-loop, attachments, markdown — see the dedicated section above).
- Do not assume optional sections. Ask before including business hours, FAQs, language rules, or restrictions.
- Do not invent team IDs, user IDs, webhook URLs, form names, or label keys. Ask the user; if unknown, leave a `TODO` placeholder.
- Do not output malformed `[[...]]` commands. Every command must match one of the four valid families in the command reference.
- Do not write SDK code, webhook handlers, or backend logic — that belongs in the **socialdesk-node** skill.
- Do not wrap the final deliverable in code fences or add preamble/closing text inside the pasteable block.
