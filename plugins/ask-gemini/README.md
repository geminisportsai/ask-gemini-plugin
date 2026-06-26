# ask-gemini (Claude plugin)

Football scouting & analytics assistant: bundles the model-invoked skills for
every intent plus the remote MCP server connection.

## Install

**In a Claude Code session (slash commands):**

```
/plugin marketplace add geminisportsai/ask-gemini-plugin
/plugin install ask-gemini@ask-gemini
```

**From the command line (Claude Code CLI):**

```
claude plugin marketplace add https://github.com/geminisportsai/ask-gemini-plugin
claude plugin install ask-gemini@ask-gemini
```

**In Cowork:** Customize → Plugins → Personal plugins **`+`** → **Create plugin**
→ **Add marketplace** → paste
`https://github.com/geminisportsai/ask-gemini-plugin` → **Sync** → **Install**.

## MCP endpoint

`.mcp.json` **currently defaults to staging**. To target another environment, set
`ASK_GEMINI_MCP_URL` before launching Claude Code (e.g. the production gateway
`https://gql.geminisports.io/mcp`). The MCP server
requires OAuth on first use.
