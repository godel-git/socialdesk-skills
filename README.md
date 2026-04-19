```
 ____   ___   ____ ___    _    _     ____  _____ ____  _  __
/ ___| / _ \ / ___|_ _|  / \  | |   |  _ \| ____/ ___|| |/ /
\___ \| | | | |    | |  / _ \ | |   | | | |  _| \___ \| ' / 
 ___) | |_| | |___ | | / ___ \| |___| |_| | |___ ___) | . \ 
|____/ \___/ \____|___/_/   \_\_____|____/|_____|____/|_|\_\

 ____  _  _____ _     _     ____  
/ ___|| |/ /_ _| |   | |   / ___| 
\___ \| ' / | || |   | |   \___ \ 
 ___) | . \ | || |___| |___ ___) |
|____/|_|\_\___|_____|_____|____/ 

  ┌──────────────────────────────────────────────────┐
  │  Skills publicos de Socialdesk para Claude Code  │
  │                                                  │
  │  Prompts · API · SDK                             │
  │  Todo para construir sobre Socialdesk.           │
  └──────────────────────────────────────────────────┘
```

# Socialdesk Skills

Coleccion publica de skills de [Claude Code](https://claude.ai/claude-code) para trabajar con la plataforma de **Socialdesk**. Cada skill le enseña a Claude como interactuar con una parte especifica del ecosistema.

## Skills disponibles

### `socialdesk-ai-assistant`

Skill para crear y gestionar prompts personalizados del **Socialdesk AI Assistant**. Guia paso a paso para escribir prompts con comandos `[[...]]` (handoff, etiquetas, webhooks, callbacks) listos para pegar en la configuracion de la extension.

### `socialdesk-api`

Referencia completa de la **API REST de Socialdesk** para construir integraciones y bots. Cubre los endpoints principales (`/messages/reply`, `/messages/HSM`, `/conversations/assign`, `/conversations/labels/add`), tipos de webhook, manejo de tokens y errores comunes.

### `socialdesk-node`

Documentacion del **SDK oficial de Node.js/TypeScript** (`socialdesk-node`). Incluye `WebhookHelper` para parsear eventos y `SocialdeskClient` para llamar la API, con ejemplos para Express y AWS Lambda.

## Como usar

Agrega este repositorio como plugin de Claude Code:

```bash
claude plugins add /ruta/a/socialdesk-skills
```

O referencia los skills individuales desde tu proyecto.

## Estructura

```
socialdesk-skills/
├── socialdesk-ai-assistant/
│   └── SKILL.md          # Guia de prompts para AI Assistant
├── socialdesk-api/
│   ├── SKILL.md           # Documentacion de la API
│   └── api-reference.md   # Referencia detallada de endpoints
├── socialdesk-node/
│   └── SKILL.md           # SDK de Node.js/TypeScript
└── docs/
    └── superpowers/       # Planes y specs de desarrollo
```
