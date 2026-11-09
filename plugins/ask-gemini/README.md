# Gemini Sports for Claude

Gemini Sports is the decision and planning layer for professional football clubs. This plugin and connector let staff at a Gemini Sports club work with their club's private workspace from Claude: find and compare players, read scout reports and notes, review transfers, loans and contracts, plan the squad, and manage watchlists and pipeline boards.

## Install

```
/plugin install ask-gemini --marketplace geminisportsai/ask-gemini-plugin
```

## Who can use it

Gemini Sports organisations are private and invitation-only. You need an active Gemini Sports account at a club. Claude can reach only what your role in your club already allows: contracts, clauses, squad wages and squad planning depend on your role, and the compliance overview (BETA) depends on your role and on your club having Compliance switched on.

## Sign-in

The first time Claude uses the connector, you sign in through OAuth with your phone number and a one-time code sent to you by SMS. Claude stores the access token that sign-in returns and sends it with each request. You can disconnect at any time from Claude's connector settings.

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

Claude asks before it uses a tool that makes a change, in line with your Claude permission settings.

## Data flows

This plugin contains instructions (skills) and a connection setting only. It runs no code on your computer and sends nothing to any destination other than those listed here.

Where data goes:

1. **The Gemini Sports MCP server**, `https://gql.geminisports.io/mcp`. This is the only destination for tool calls.
2. **Gemini Sports sign-in (OAuth)**, Gemini Sports' sign-in service, at `https://gql.geminisports.io`, used only when you sign in or your session is refreshed. The SMS code is sent by Gemini Sports' sign-in service through Amazon Web Services (Amazon Simple Notification Service).

If you set the `ASK_GEMINI_MCP_URL` environment variable before starting Claude, tool calls go to that URL instead, and sign-in uses whatever sign-in service that server advertises.

What is sent:

- Your access token with each request.
- The tool Claude chooses and its inputs, built from your request: search filters, player, team, match, watchlist or board names and IDs, and the stat question you asked. For changes, it also sends the watchlist or board name and the players to add or remove.
- At sign-in: your phone number and the code you enter.

What comes back: the player, team, match, report, watchlist and board information your club's Gemini Sports account allows you to see. Claude uses it to answer you in the conversation. Stat questions answered by SQL are scoped to your club, not to your individual role. Results returned by Gemini Sports become part of your Claude conversation and are handled under your Claude account's data and retention settings.

What is not sent: the plugin does not read your local files, your other connectors or the rest of your Claude conversation. Only the inputs to each tool call leave Claude for Gemini Sports.

How Gemini Sports keeps and logs requests: Gemini Sports keeps request logs for up to 12 months, with identifiers masked, to run, secure and troubleshoot the service. See the Gemini Sports for Claude privacy policy at https://github.com/geminisportsai/ask-gemini-plugin/blob/main/PRIVACY.md.

## Good to know

- Claude guides; your staff make every call. Check important figures in the Gemini Sports app before you act on them.
- Compliance figures inform your planning and never block a move.
- Watchlists never show financial information, at any permission level.

## Support

Email cody@geminisports.ai or visit https://github.com/geminisportsai/ask-gemini-plugin#readme.
