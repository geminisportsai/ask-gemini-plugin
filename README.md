# ask-gemini-plugin

Public Claude Code plugin marketplace for **Ask Gemini** — Gemini Sports AI's
football scouting & analytics assistant.

The plugin bundles the model-invoked skills for every Ask Gemini intent and the
remote MCP server connection.

## Install

### In a Claude Code session (slash commands)

```
/plugin marketplace add geminisportsai/ask-gemini-plugin
/plugin install ask-gemini@ask-gemini
```

### From the command line (Claude Code CLI)

```
claude plugin marketplace add https://github.com/geminisportsai/ask-gemini-plugin
claude plugin install ask-gemini@ask-gemini
```

### In Cowork (desktop app)

Customize → **Plugins** → under **Personal plugins** click **`+`** → **Create
plugin** → **Add marketplace**, then paste
`https://github.com/geminisportsai/ask-gemini-plugin` and **Sync**. The
**ask-gemini** plugin then appears under Personal plugins — click **Install**.
(The curated **Directory** does not list custom repos; "Add marketplace" is
nested under "Create plugin".)

## MCP endpoint

The MCP connector **currently defaults to staging**
(`https://gql.staging.geminisports.io/mcp`). Override it for
another environment by setting `ASK_GEMINI_MCP_URL` before launching Claude Code,
e.g.:

```
ASK_GEMINI_MCP_URL=https://gql.geminisports.io/mcp   # production
ASK_GEMINI_MCP_URL=https://gql.dev.geminisports.io/mcp                          # dev
```

The MCP server requires OAuth — you'll be prompted to authorize the first time an
ask-gemini tool runs.

---

This repository is generated/published from the (private) `ask-gemini` repo —
do not edit by hand.
