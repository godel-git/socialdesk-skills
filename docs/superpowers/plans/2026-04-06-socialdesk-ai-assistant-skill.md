# Socialdesk AI Assistant Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a single Claude Code skill file that guides users (technical or not) through building a custom prompt for the Socialdesk AI Assistant app, assisting with `[[...]]` command construction and validation, and avoiding duplication of rules already enforced by the parent system prompt in `run-prompt.middleware.ts`.

**Architecture:** One `SKILL.md` file authored in `/Users/bryanramirez/Sites/socialdesk-skills/socialdesk-ai-assistant/`, following the existing convention of sibling skills (`socialdesk-node`, `socialdesk-api`). The skill is a text-only deliverable — no code, no tests beyond structural verification.

**Tech Stack:** Markdown + YAML frontmatter (Claude Code skill format).

**Spec:** `docs/superpowers/specs/2026-04-06-socialdesk-ai-assistant-skill-design.md`

---

## File Structure

- **Create:** `socialdesk-ai-assistant/SKILL.md` — the skill itself

No other files. No code, no tests. Verification is structural (frontmatter valid, required sections present, command cheat sheet matches `shared/types.ts`).

---

### Task 1: Create skill directory and author SKILL.md

**Files:**
- Create: `/Users/bryanramirez/Sites/socialdesk-skills/socialdesk-ai-assistant/SKILL.md`

- [ ] **Step 1: Create the directory**

```bash
mkdir -p /Users/bryanramirez/Sites/socialdesk-skills/socialdesk-ai-assistant
```

- [ ] **Step 2: Re-read the spec and the reference sources**

Before writing, re-read these files to ground the content:
- `docs/superpowers/specs/2026-04-06-socialdesk-ai-assistant-skill-design.md` (the spec)
- `/Users/bryanramirez/Sites/desk-extensions-serverless/src/packages/basic-assistant/ai-assistant/middlewares/run-prompt.middleware.ts` (to confirm what the parent prompt already covers — skill must NOT duplicate)
- `/Users/bryanramirez/Sites/desk-extensions-serverless/src/packages/basic-assistant/ai-assistant/shared/types.ts` (canonical list of valid `[[...]]` command types)
- `/Users/bryanramirez/Sites/socialdesk-skills/socialdesk-node/SKILL.md` (formatting convention for sibling skills)

- [ ] **Step 3: Write SKILL.md with the full structure below**

The file must contain these sections in order. Write real content in each — no placeholders.

**Frontmatter (exact):**
```yaml
---
name: socialdesk-ai-assistant
description: Use when the user wants to create, structure, or fix a custom prompt for the Socialdesk AI Assistant app (the chatbot configured via openAIPrompt). Guides users through required sections, assists building [[...]] commands, and validates command syntax. NOT for SDK/integration code.
---
```

**Sections (H2):**

1. `# Socialdesk AI Assistant Prompt Builder` (H1 title)
2. `## When to use` — trigger conditions (user wants to write/edit/fix a custom prompt for the AI Assistant app). Explicitly list when NOT to use (SDK code, webhook handlers, integrations — point to `socialdesk-node` skill).
3. `## What this produces` — a plain-text block ready to paste into the `openAIPrompt` field of the AI Assistant extension settings.
4. `## What the parent prompt already covers (DO NOT repeat)` — bulleted list so Claude avoids duplication. Include: JSON output shape, `[[...]]` syntax explanation, WhatsApp Cloud API rules (buttons/lists/limits), "don't invent info" / no-loop / no re-asking, time/timezone context, interactions vs plain text, attachment rules, markdown conversion. Cite `run-prompt.middleware.ts` lines 52–341 as the source of truth.
5. `## Flow` — numbered steps:
   - **Step 1 — Detect mode.** If user provides a draft → reconstruct mode (parse, identify sections present, validate commands). If blank → guided mode (ask one question at a time).
   - **Step 2 — Required sections loop.** Always collect: (1) role & identity, (2) initial message, (3) knowledge base, (4) human handoff. Ask one at a time.
   - **Step 3 — Optional sections.** Ask "¿quieres incluir X?" for each: business hours, language(s), command-driven flows, free-form FAQs, business-specific restrictions. Do not assume.
   - **Step 4 — Command builder.** When handoff or flows need commands, assist non-technical users by asking plain-language questions and building the `[[...]]` behind the scenes. Include the table from the spec (intent → questions → output).
   - **Step 5 — Command validator.** When a draft is provided, validate every `[[...]]` against the 4 valid types. On error: explain plainly, propose fix, confirm.
   - **Step 6 — Assemble & present.** Deliver the final prompt as a single plain-text block in numbered sections, styled like the sales-bot example.
6. `## Command reference` — cheat sheet of the 4 valid command types with one-line description and exact syntax example for each:
   - `[[ASSIGN_TO:USER:<id>]]` / `[[ASSIGN_TO:TEAM:<id>]]`
   - `[[ADD_LABEL:<key>]]`
   - `[[WEBHOOK:POST|URL=<url>&payload={...}&headers={...}]]`
   - `[[ASSISTANT_CALLBACK:FORM_ASSISTANT|PREFILLED_FORM_SUBMIT?formname=<name>&field=<value>]]`
   - `[[ASSISTANT_CALLBACK:ASSIGN_ASSISTANT|ASSIGN?id=<id>&step=<n>]]`
   Mention placeholders like `<nombre>`, `<correo>` get replaced by the bot from chat/contact data.
7. `## Command-building guidance for non-technical users` — scripts/phrasings the skill should use when asking about IDs, URLs, form names. Include guidance like "if they don't know the team ID, explain it's found in Socialdesk under Settings → Teams" (phrase it as guidance, not a hard-coded path Claude should invent).
8. `## Example output (abbreviated)` — a condensed version of the sales-bot example (3–4 sections only: rol, mensaje inicial, base de conocimiento, derivación con un comando), to show the target format. Keep under 40 lines.
9. `## Anti-patterns` — short list: don't repeat parent-prompt rules, don't assume optional sections, don't invent IDs/URLs, don't output malformed commands, don't write SDK code.

- [ ] **Step 4: Verify frontmatter is valid YAML and required fields exist**

```bash
head -5 /Users/bryanramirez/Sites/socialdesk-skills/socialdesk-ai-assistant/SKILL.md
```

Expected: frontmatter delimited by `---`, contains `name:` and `description:` fields.

- [ ] **Step 5: Verify all required H2 sections are present**

Use Grep to confirm each section header exists:
```
^## When to use
^## What this produces
^## What the parent prompt already covers
^## Flow
^## Command reference
^## Command-building guidance
^## Example output
^## Anti-patterns
```

Expected: all 8 patterns match.

- [ ] **Step 6: Verify the command reference lists all 4 valid command types**

Grep for each of: `ASSIGN_TO`, `ADD_LABEL`, `WEBHOOK`, `ASSISTANT_CALLBACK` in SKILL.md. Expected: each appears at least once in the Command reference section.

- [ ] **Step 7: Verify no duplication of parent-prompt rules**

Grep SKILL.md for phrases that would indicate the skill is telling Claude to repeat parent-prompt rules in the generated output:
- "JSON output" / "return JSON"
- "button limit" / "list limit"
- "timezone"

If any appear, they must be inside the "What the parent prompt already covers (DO NOT repeat)" section only. If they appear in the example output or the command reference as instructions to emit, remove them.

- [ ] **Step 8: Commit**

```bash
cd /Users/bryanramirez/Sites/socialdesk-skills
git add socialdesk-ai-assistant/SKILL.md
git commit -m "$(cat <<'EOF'
Add socialdesk-ai-assistant skill

Guides users through authoring custom prompts for the Socialdesk AI
Assistant app. Covers required/optional sections, assists building
[[...]] commands, validates command syntax, and avoids duplicating
rules already enforced by the parent system prompt.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Self-Review

- **Spec coverage:** all 7 spec sections (hybrid mode, required, optional, command builder, validator, token economy, final output) map to Flow steps 1–6 and the cheat sheet. ✓
- **Placeholders:** none. Each step has concrete content. ✓
- **Type consistency:** command type names match `shared/types.ts` exactly. ✓
