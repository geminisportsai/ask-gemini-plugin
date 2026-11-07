# Gemini Sports for Claude

Public Claude plugin marketplace for **Gemini Sports** (plugin id `ask-gemini`),
the football scouting assistant from Gemini Sports.

The plugin lets staff at a Gemini Sports club work with their club's private
workspace from Claude: find and compare players, read scout reports and notes,
review transfers, loans and contracts, plan the squad, and manage watchlists and
pipeline boards. It bundles the model-invoked skills for every intent and the
connection to the remote Gemini Sports MCP server. You need an active Gemini
Sports account at a club.

## Install

### In a Claude Code session (slash commands)

```
/plugin install ask-gemini --marketplace geminisportsai/ask-gemini-plugin
```

On Claude Code older than 2.1.275, add the marketplace first:

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

## Sign-in and data flows

The first time Claude uses the connector, you sign in through OAuth with your
phone number and a one-time code sent by SMS. Tool calls, with the inputs Claude
builds from your request, go only to the Gemini Sports MCP server. The plugin
runs no code on your computer and does not read your local files.

The full disclosure of what is sent, to which host, and what comes back is in
the plugin's README, [plugins/ask-gemini/README.md](plugins/ask-gemini/README.md),
from the next plugin release.

## MCP endpoint

The MCP connector **currently defaults to staging**
(`https://gql.staging.geminisports.io/mcp`). The default moves to production in
a later release. Override it for another environment by setting
`ASK_GEMINI_MCP_URL` before launching Claude Code, e.g.:

```
ASK_GEMINI_MCP_URL=https://gql.dev.geminisports.io/mcp   # dev
```

If you set it, tool calls go to that URL instead, and sign-in uses whatever
sign-in service that server advertises.

## License

Copyright (c) 2026 Gemini Sports Inc. All rights reserved. The plugin carries this
notice as `plugins/ask-gemini/LICENSE` from the next plugin release.

---

This repository is generated/published from the (private) `ask-gemini` repo —
do not edit by hand.
