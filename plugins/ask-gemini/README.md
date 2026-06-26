# ask-gemini (Claude plugin)

Football scouting & analytics assistant: bundles the model-invoked skills for
every Ask Gemini intent plus the remote MCP server connection. Installs in
**Claude Code** today; **Cowork** support is blocked by an upstream bug (see
below).

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

**In Cowork:** ⚠️ **Not working yet.** Cowork currently can't add a custom GitHub
marketplace — it fails with _"Could not connect to the marketplace host"_
([anthropics/claude-code#28125](https://github.com/anthropics/claude-code/issues/28125),
open). Until that's fixed, install via the **Claude Code CLI** above. Once it's
resolved, the Cowork path will be: Customize → Plugins → Personal plugins **`+`**
→ **Create plugin** → **Add marketplace** → paste
`https://github.com/geminisportsai/ask-gemini-plugin` → **Sync** → **Install**.

## MCP endpoint

`.mcp.json` **currently defaults to staging**. To target another environment, set
`ASK_GEMINI_MCP_URL` before launching Claude Code — e.g. dev
`https://gql.dev.geminisports.io/mcp`. The MCP server requires OAuth on first use.

> **TODO (production):** `https://gql.geminisports.io/mcp` is not live yet — the
> production MCP endpoint comes up with the first production deploy.

---

_Maintainers: this plugin is built and published from the private `ask-gemini`
repo (`scripts/build-claude-plugin.ts`) and synced here on each staging deploy —
do not edit by hand._
