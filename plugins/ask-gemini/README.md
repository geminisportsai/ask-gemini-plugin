# ask-gemini (Claude plugin)

Football scouting & analytics assistant: bundles the model-invoked skills for
every intent plus the remote MCP server connection.

## Install

```
/plugin marketplace add geminisportsai/ask-gemini-plugin
/plugin install ask-gemini@ask-gemini
```

## MCP endpoint

`.mcp.json` defaults to the production MCP gateway. To target another
environment, set `ASK_GEMINI_MCP_URL` before launching Claude Code.
