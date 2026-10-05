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

If the user asks which watchlists they have ("Which watchlists do I have?", "What watchlists do I have?", "How many watchlists do I have?"), skip the matching below and do not ask which watchlist they mean: go straight to "Which watchlists" in Step 2 once every page is read.

If the user says "default" ("my default watchlist") or names no watchlist ("my watchlist"), do not match a name: go to the two "No watchlist named" bullets below. A plural or aggregate count question that names no watchlist ("how many players…": "How many players do we have on watchlists?", "How many players are on our watchlists?", "How many players across my watchlists?", "How many players are there across all my watchlists in total?") is not a default-list question: once every page is read, go to "Count across watchlists" in Step 2. A plural question about who is on the lists ("Who is on my watchlists?", "Show me the players on our watchlists") is not a count: treat it like "my watchlist" and follow the two "No watchlist named" bullets below. Otherwise, match the name the user gave against `watchlist.name` with a case-insensitive comparison, ignoring a trailing "watchlist" or "list" in what the user typed ("my Pietra watchlist" means the watchlist named "Pietra").

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

### Count across watchlists

List each `watchlist.name` from every page of Step 1 with its `watchlist.playerCount`, quoted exactly as the tool returned it, and mark the default one. Make no extra call per list.

- If the user has two or more watchlists, then make exactly one call: `listWatchlistPlayers` with `watchlistIds: [every watchlistId from Step 1]` and `first: 1`, and read its `totalCount`. That is the number of different players across the lists, already deduplicated by the app. Say "<totalCount> different players across your lists", and say once that a player on several lists is counted once.
- Never add up the `playerCount` values and never present a sum — not as a total, not as "entries", not as a check. The only total is `totalCount`.
- If the call fails or `totalCount` is missing, give the per-list counts and say the combined total isn't available right now. Do not compute one.
- Never page players for this question: one call, `first: 1`, no `after`, and never list the players.
- Never call `listWatchlistPlayers` without `watchlistId` or `watchlistIds`; the app refuses it.
- If the user has exactly one watchlist, give its name and count with no total and make no `listWatchlistPlayers` call. If the user has no watchlists, say they have no watchlists.

A singular, "default" or named watchlist question never takes this path and never gets a total across lists.

### Which watchlists

List each `watchlist.name` from every page of Step 1 with its `watchlist.playerCount`, and mark the default one. For "How many watchlists do I have?", also give the number of lists from Step 1's `totalCount`, and never the player total across lists.

## Common pitfalls

1. Don't answer from the conversation — call the tools every time.
2. Don't stop at the first page of `listMyUserWatchlists` — the watchlist the user named may be on a later page.
3. Don't pass the membership `id` — use `watchlistId`.
4. Don't say you have no access to watchlist data — these tools are the access.
5. Never compute a total yourself — no sum of `playerCount`, in any wording; the only total across lists is the tool's distinct `totalCount` (see "Count across watchlists").
