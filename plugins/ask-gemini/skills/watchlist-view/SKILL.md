---
name: watchlist-view
description: >
  Read the user's watchlists -- count the players on a watchlist, list who is on it,
  or list the watchlists the user has. Read-only: nothing is added or removed.
---

# Watchlist View

Use this skill when the user asks how many players are on a watchlist, who is on a watchlist, or which watchlists they have. This is a read: never add or remove a player here.

**Never answer a count or a player list from earlier messages in the conversation.** Watchlists change between turns. Call the tools below every time, even when an earlier answer in this chat already mentioned the watchlist.

## Step 1: Find the watchlist

Call `listMyUserWatchlists` with `first: 50`, then page `listMyUserWatchlists` with `after: <pageInfo.endCursor>` until `pageInfo.hasNextPage` is false. `totalCount` is how many watchlists the user has. Do not decide a watchlist is missing before you have read every page.

Each entry is a *membership* record. The watchlist's name is `watchlist.name`, and `isDefault` marks the default watchlist. Pass the `watchlistId` to every other tool, never the membership `id`.

If the user asks which watchlists they have ("Which watchlists do I have?", "What watchlists do I have?"), skip the matching below and do not ask which watchlist they mean: go straight to "Which watchlists" in Step 2 once every page is read.

If the user says "default" ("my default watchlist") or names no watchlist ("my watchlist"), do not match a name: go to the two "No watchlist named" bullets below. Treat a plural or aggregate question that names no watchlist ("How many players do we have on watchlists?", "How many players are on our watchlists?", "How many players across my watchlists?") exactly like "my watchlist" and follow the two "No watchlist named" bullets below. Otherwise, match the name the user gave against `watchlist.name` with a case-insensitive comparison, ignoring a trailing "watchlist" or "list" in what the user typed ("my Pietra watchlist" means the watchlist named "Pietra").

- **Exactly one match**: use it.
- **More than one match** (for example "Pietra" and "Pietra U21" when the user typed "pietra" and no name matches exactly): ask which one they mean, naming each.
- **No watchlist named, and one entry has `isDefault` true** ("How many players are on my watchlist?", "Who is on my default watchlist?"): use the default watchlist. Do not ask which watchlist they mean — this is a read, so answering from the default list is safe. Name the default list in the answer, leading with this sentence:

  > Your default watchlist, <name>, has <playerCount> players.

  <name> is its `watchlist.name`, and <playerCount> is the `watchlist.playerCount` of the default entry from Step 1. Write "player" when `playerCount` is 1. For a count, that sentence is the answer. For a player list, list the players after it; if you stopped paging early, follow that sentence with "Here are the first N of <playerCount> players" before the list.

  For a count or a player list, if the user has other watchlists, end with one line naming each other `watchlist.name` and offer to check another, for example: "You also have U21 Targets and Loan Watch — want me to check one of those?"
- **No watchlist named, and no default**: if the user has exactly one watchlist, use it and name it. If they have more than one, ask which watchlist they mean, naming each. If they have none, say they have no watchlists.
- **No match**: stop and answer with this template, listing every `watchlist.name` from every page:

  > I couldn't find a watchlist named "<name>". Your watchlists are: <name 1>, <name 2>, …

  Do not guess a close match, and do not answer for a different watchlist.

## Step 2: Answer

### How many players

Call `getWatchlist` with `id: <watchlistId>` and report its `playerCount`. If `getWatchlist` returns null, the watchlist is no longer available to the user — say so rather than reporting a number.

### Who is on it

Call `listWatchlistPlayers` with `watchlistId: <watchlistId>` and `first: 50`, then page `listWatchlistPlayers` with `after: <pageInfo.endCursor>` until `pageInfo.hasNextPage` is false. List every player returned, by name. Lead with the count from `getWatchlist`'s `playerCount` when you have it. If the watchlist is empty, say it has no players.

If you stop paging before `pageInfo.hasNextPage` is false, for any reason, say so: start with "Here are the first N of <playerCount> players", where N is how many you list. Never present a partial list as complete.

Only describe a player with fields this tool returned. Do not add a club, position or rating from memory.

### Which watchlists

List each `watchlist.name` from every page of Step 1 with its `watchlist.playerCount`, and mark the default one.

## Common pitfalls

1. Don't answer from the conversation — call the tools every time.
2. Don't stop at the first page of `listMyUserWatchlists` — the watchlist the user named may be on a later page.
3. Don't pass the membership `id` — use `watchlistId`.
4. Don't say you have no access to watchlist data — these tools are the access.
5. Don't add up `playerCount` across watchlists or report a total across lists — the app keeps counts per watchlist, and a player can be on several lists. Answer as for an unnamed watchlist (Step 1); if the user explicitly asked for a total across lists, say in one short clause that counts are kept per watchlist before that answer.
