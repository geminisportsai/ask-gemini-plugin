# Gemini Sports for Claude

Gemini Sports is the decision and planning layer for professional football
clubs. This plugin and connector let staff at a Gemini Sports club work with
their club's private workspace from Claude: find and compare players, read scout
reports and notes, review transfers, loans and contracts, plan the squad, and
manage watchlists and pipeline boards.

This repository is the public Claude plugin marketplace for it (plugin id
`ask-gemini`). The plugin bundles the model-invoked skills for every intent and
the connection to the remote Gemini Sports MCP server.

- [Who can use it](#who-can-use-it)
- [Install](#install)
- [Sign-in](#sign-in)
- [What it can do](#what-it-can-do)
- [Data flows](#data-flows)
- [MCP endpoint](#mcp-endpoint)
- [Good to know](#good-to-know)
- [Support](#support)
- [Privacy policy](PRIVACY.md)

## Who can use it

Gemini Sports organizations are private and invitation-only. You need an active
Gemini Sports account at a club. If your club is not on Gemini Sports yet,
[book a demo with our team](https://geminisports.ai/talk-to-team).

Claude only reaches your own club's data. Contracts, clauses, squad wages and
squad planning depend on your role, and the compliance overview (BETA) depends
on your role and on your club having Compliance switched on.

## Install

### In Claude Code

Install the plugin from this marketplace in one step:

```
/plugin install ask-gemini --marketplace geminisportsai/ask-gemini-plugin
```

On Claude Code older than 2.1.275, add the Gemini Sports marketplace, then
install the plugin:

```
/plugin marketplace add geminisportsai/ask-gemini-plugin
```

```
/plugin install ask-gemini@ask-gemini
```

From the command line (Claude Code CLI):

```
claude plugin marketplace add https://github.com/geminisportsai/ask-gemini-plugin
claude plugin install ask-gemini@ask-gemini
```

From the Claude Code App: in the **Code** tab, enter the prompt

> run the following commands
> `claude plugin marketplace add https://github.com/geminisportsai/ask-gemini-plugin`
> `claude plugin install ask-gemini@ask-gemini`

then **quit and reopen the app** for the plugin to load.

### In Claude.ai

Add a custom connector in your Claude connector settings and enter this URL:
`https://gql.geminisports.io/mcp`

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

## Sign-in

The first time Claude uses the connector, you sign in through Gemini Sports'
sign-in service (OAuth), hosted on Amazon Web Services, with your phone number
and a one-time code sent to you by SMS. Claude stores the access token that
sign-in returns and sends it with each request. You can disconnect at any time
from Claude's connector settings.

## What it can do

- Search and filter players; get player summaries; find similar players; see how a player fits your team.
- Review transfers and loans; check contracts and clauses (depending on your role).
- See squad wages and squad plans (depending on your role) and the compliance overview (BETA).
- Read scout reports and notes; browse matches, match reports and injuries; see your home and dashboard insights.
- Look up season and match stats. These are answered by read-only SQL queries that run on the Gemini Sports server, inside your club's access rules.
- View watchlists and pipeline boards.

Some tools also make changes in your Gemini Sports workspace:

- Create a watchlist, and add players to or remove them from a watchlist.
- Create a private pipeline board, and add players to or remove them from a board.
- Delete a conversation thread.

Claude asks before it uses a tool that makes a change, in line with your Claude
permission settings.

## Data flows

This plugin contains instructions (skills) and a connection setting only. It
runs no code on your computer and sends nothing to any destination other than
those listed here.

### Where data goes

1. **The Gemini Sports MCP server**: `https://gql.geminisports.io/mcp` for the
   Claude.ai connector; the plugin's exact host is listed in the Data flows of
   [plugins/ask-gemini/README.md](plugins/ask-gemini/README.md). This is the
   only destination for tool calls.
2. **Gemini Sports sign-in (OAuth)**: Gemini Sports' sign-in service, hosted on
   Amazon Web Services, which the MCP server advertises. It is used only when
   you sign in or your session is refreshed. The SMS code is sent by Gemini
   Sports' sign-in service through Amazon Web Services (Amazon Simple
   Notification Service).

### What is sent

- Your access token with each request.
- The tool Claude chooses and its inputs, built from your request: search filters, player, team, match, watchlist or board names and IDs, and the stat question you asked. For changes, it also sends the watchlist or board name and the players to add or remove.
- At sign-in: your phone number and the code you enter.

### What comes back

The player, team, match, report, watchlist and board information from your
club's workspace that each tool returns for your account. Claude uses it to
answer you in the conversation. Stat questions answered by SQL are scoped to
your club, not to your individual role. Results returned by Gemini Sports become
part of your Claude conversation and are handled under your Claude account's
data and retention settings.

### What is not sent

The plugin does not read your local files, your other connectors or the rest of
your Claude conversation. Only the inputs to each tool call leave Claude for
Gemini Sports.

### Logs and privacy

Gemini Sports keeps request logs for up to 12 months, with identifiers masked,
to run, secure and troubleshoot the service. See the
[Gemini Sports for Claude privacy policy](PRIVACY.md) for the full details.

## MCP endpoint

The plugin connects to the Gemini Sports MCP server. Its exact host is listed
in the Data flows of [plugins/ask-gemini/README.md](plugins/ask-gemini/README.md).
Override it for another environment by setting `ASK_GEMINI_MCP_URL` before
launching Claude Code, e.g.:

```
ASK_GEMINI_MCP_URL=https://<your-mcp-host>/mcp
```

If you set it, tool calls go to that URL instead, and sign-in uses whatever
sign-in service that server advertises.

## Good to know

- Claude guides; your staff make every call. Check important figures in the Gemini Sports app before you act on them.
- Compliance figures inform your planning and never block a move.
- Watchlists never show financial information, at any permission level.

## Support

Email [cody@geminisports.ai](mailto:cody@geminisports.ai).

## License

Copyright (c) 2026 Gemini Sports Inc. All rights reserved. The plugin carries this
notice as `plugins/ask-gemini/LICENSE` from the next plugin release.

---

The plugin under `plugins/ask-gemini/` is generated and published from the
private `ask-gemini` repo on each release — do not edit it by hand.
