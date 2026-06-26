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

### From the Claude Code App

In the **Code** tab, enter the prompt:

> run the following commands
> `claude plugin marketplace add https://github.com/geminisportsai/ask-gemini-plugin`
> `claude plugin install ask-gemini@ask-gemini`

Then **quit and reopen the app** for the plugin to load.

### In Cowork (desktop app)

⚠️ **Not working yet.** Cowork currently can't add a custom GitHub marketplace —
it fails with _"Could not connect to the marketplace host"_
([anthropics/claude-code#28125](https://github.com/anthropics/claude-code/issues/28125),
open). Until that's fixed, use the **Claude Code CLI** method above. Once it's
resolved, the Cowork path will be: Customize → **Plugins** → **Personal plugins**
**`+`** → **Create plugin** → **Add marketplace** → paste
`https://github.com/geminisportsai/ask-gemini-plugin` → **Sync** → **Install**.
(The curated **Directory** does not list custom repos; "Add marketplace" is
nested under "Create plugin".)

## MCP endpoint

The MCP connector **currently defaults to staging**
(`https://gql.staging.geminisports.io/mcp`). Override it for another environment
by setting `ASK_GEMINI_MCP_URL` before launching Claude Code, e.g.:

```
ASK_GEMINI_MCP_URL=https://gql.dev.geminisports.io/mcp   # dev
```

> **TODO (production):** `https://gql.geminisports.io/mcp` is not live yet — the
> production MCP endpoint comes up with the first production deploy.

The MCP server requires OAuth — you'll be prompted to authorize the first time an
ask-gemini tool runs.

---

This repository is generated/published from the (private) `ask-gemini` repo —
do not edit by hand.
