# Socialdesk AI Assistant Prompt Builder — Skill Design

**Date:** 2026-04-06
**Location:** `/Users/bryanramirez/Sites/socialdesk-skills/socialdesk-ai-assistant/SKILL.md`

## Purpose

A Claude Code skill that helps any user (technical or not) author the **custom prompt** pasted into the Socialdesk "AI Assistant" app configuration (field `extensionSettings.openAIPrompt`, wrapped at runtime between `{{CUSTOM_INSTRUCTIONS_START}} ... {{CUSTOM_INSTRUCTIONS_END}}` by `run-prompt.middleware.ts`).

**Out of scope:** writing integration/SDK code, webhook handlers, or anything that isn't the prompt text itself.

## Why this skill exists

- Users write custom prompts by hand and repeat rules already enforced by the parent system prompt, wasting tokens.
- Non-technical users don't know the `[[...]]` command syntax or where to get IDs (team/user/form).
- Drafts often contain malformed commands that silently break flows.

## Core behavior

### 1. Hybrid entry mode
- **Draft provided** → parse it, detect existing sections, list what's missing, validate any `[[...]]` commands already present.
- **Blank start** → guided Q&A, one question at a time.

### 2. Required sections (always collected)
1. **Role & identity** — company name, channel, tone/personality
2. **Initial message** — greeting style, whether to offer a menu
3. **Knowledge base** — what the business does, main products/services
4. **Human handoff** — when to escalate and to whom (with command-building assistance)

### 3. Optional sections (ask-to-include, not assumed)
- Business hours
- Language(s)
- Command-driven flows (self-registration, forms, webhooks)
- Free-form FAQs (location, pricing, policies — whatever the business wants)
- Business-specific restrictions

Not assumed because the example bot's options (pricing, demo scheduling, auto-registration) are particular to Socialdesk itself, not every business.

### 4. Command builder (assists non-technical users)

| User intent | Skill asks | Skill outputs |
|---|---|---|
| Hand off to an agent | User or team? ID? (explain where to find it) | `[[ASSIGN_TO:TEAM:<id>]]` or `[[ASSIGN_TO:USER:<id>]]` |
| Tag the conversation | Which label key? | `[[ADD_LABEL:<key>]]` |
| Send data to external system | URL, which fields to send | `[[WEBHOOK:POST\|URL=...&payload={...}]]` with `<placeholders>` |
| Trigger a form | Form name, fields to prefill | `[[ASSISTANT_CALLBACK:FORM_ASSISTANT\|PREFILLED_FORM_SUBMIT?formname=...]]` |

The skill must explain each command in plain language before inserting it, so the user understands what they're approving.

### 5. Command validator (draft mode)
Checks every `[[...]]` against the 4 valid types defined in `desk-extensions-serverless/src/packages/basic-assistant/ai-assistant/shared/types.ts`:
- `ASSIGN_TO` — requires `USER:` or `TEAM:` + ID
- `ADD_LABEL` — requires a key
- `WEBHOOK` — requires method + `URL=` + optional `payload=` / `headers=`
- `ASSISTANT_CALLBACK` — requires sub-type (e.g., `FORM_ASSISTANT|PREFILLED_FORM_SUBMIT?...`)

On malformed commands: explain the issue plainly, propose a fix, confirm with user.

### 6. Token economy — what the skill MUST NOT include in the output

The parent system prompt in `run-prompt.middleware.ts` (lines 52–341) already enforces these. The custom prompt should NOT repeat them:

- JSON output shape (`text`, `commands`, `interactions`, `attachments`)
- `[[...]]` command syntax explanation
- WhatsApp Cloud API rules (button limits, list limits, title lengths)
- "Don't invent information" / no-loop / no re-asking
- Time/timezone context
- When to use interactions vs plain text
- Attachment rules, markdown conversion

### 7. Final output
Plain text block ready to paste into `openAIPrompt`, organized in numbered sections following the style of the sales bot example the user provided. Nothing outside that block.

## Skill file structure

```
---
name: socialdesk-ai-assistant
description: Use when the user wants to create, structure, or fix a custom prompt for the Socialdesk AI Assistant app (the chatbot configured via openAIPrompt). Guides non-technical users through required sections, assists building [[...]] commands, and validates command syntax. NOT for SDK/integration code.
---

# Socialdesk AI Assistant Prompt Builder

## When to use / when not to use
## What this produces (and what the parent prompt already covers)
## Flow
  - Step 1: Detect mode
  - Step 2: Required sections loop
  - Step 3: Optional sections (ask-to-include)
  - Step 4: Command builder
  - Step 5: Command validator
  - Step 6: Assemble and present final prompt
## Command reference (cheat sheet, 4 types)
## Example output (abbreviated sales-bot style)
```

## Success criteria

- Given a blank start, the skill produces a valid, paste-ready custom prompt with all required sections and no duplication of parent-prompt rules.
- Given a draft, the skill identifies missing required sections, validates commands, and returns a corrected version.
- Every `[[...]]` command in the output passes the validator rules above.
- A non-technical user can complete the flow without knowing the raw command syntax.
