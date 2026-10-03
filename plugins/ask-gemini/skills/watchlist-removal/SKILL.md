---
name: watchlist-removal
description: >
  Remove players from watchlists.
  Use when the user wants to take a player off one of their watchlists.
---

# Watchlist Removal

Use this skill when the user wants to remove a player from one of their watchlists.

If the user names the watchlist ("remove Isak from my 49 watchlist"), follow **When the user names the watchlist**. If they name no watchlist ("take Alexander Isak off my watchlist"), follow **When the user names no watchlist**.

## When the user names no watchlist

No watchlist named: find the player, then call `getPlayer` and read its `watchlists` field — it lists every one of the user's watchlists that holds the player, in one call. Do not page through `listMyUserWatchlists` or `listWatchlistPlayers` to look for the player; that runs out of steps before it reaches every list.

### Step 1: Find the player

Call `searchPlayers` with the player's name. **Confirm before removing when the name is ambiguous** — see "Ambiguous player names" below.

### Step 2: Read the lists holding the player

Call `getPlayer` with the player's id. Each entry in `watchlists` has an `id` and a `name`; that `id` is the `watchlistId` that `removePlayerFromWatchlist` and `listWatchlistPlayers` take.

`getPlayer` returns a large result that is cut to fit; `watchlists` is re-appended after the cut. If `watchlists` is absent from the result, or named in `truncation.elidedKeys`, use **Fallback** below.

### Step 3: Act on what `watchlists` holds

- **Exactly one watchlist**: call `removePlayerFromWatchlist` with the player id and that list's id as the `watchlistId` (the `watchlists` entry's `id`, or the `watchlistId` on the Fallback); it returns the watchlist with its `name` and new `playerCount`. Make no other call after the removal (not `listWatchlistPlayers`, not `getWatchlist`, not `listMyUserWatchlists`): each extra call adds about ten seconds to the answer. Answer:

  > Removed <player> from watchlist "<name>". <player> was on only that one of your watchlists. Watchlist "<name>" now contains <playerCount> players.

  <name> is the `name` the removal returned, and <playerCount> is the `playerCount` it returned. Write "player" when `playerCount` is 1. When `playerCount` is 0, end with `Watchlist "<name>" is now empty.` instead. Do not list the players still on the watchlist; the user can ask who is on it.

  If `removePlayerFromWatchlist` returns an error, say the removal failed and do not claim the player was removed. If the result has no `playerCount`, say the player was removed without giving a count.
- **Two or more watchlists**: do not remove the player yet — ask which watchlist to remove them from, naming only the watchlists in `watchlists` (never the user's other lists). For example: `Alexander Isak is on 2 of your watchlists: "49" and "Shortlist". Which one should I remove him from?` Remove only after the user answers.
- **No watchlists**: `watchlists` is empty, so the player is on none of the user's lists — say they are not on any of your watchlists, and remove nothing. `watchlists` covers every list, so this is a complete answer.

### Fallback: when `watchlists` is missing

If `watchlists` is absent or listed in `truncation.elidedKeys`, read every watchlist before deciding: page `listMyUserWatchlists` with `first: 50` until `pageInfo.hasNextPage` is false, then for each list call `listWatchlistPlayers` with `first: 100` and its `watchlistId`, paging with `after: <pageInfo.endCursor>` until `pageInfo.hasNextPage` is false. Collect the lists whose players include the player's id, then act as in Step 3.

Never say the player is not on your watchlists while any watchlist is unchecked. If you cannot read every list, name the lists you checked and the ones you did not, and ask whether to keep looking.

## When the user names the watchlist

### Step 1: List Existing Watchlists

Call `listMyUserWatchlists` to retrieve the user's current watchlists. Present them so the user can identify which watchlist to remove a player from. If the user has already specified the watchlist, proceed directly.

**Use the `watchlistId` field, never `id`.** Each `listMyUserWatchlists` entry is a *membership* record with two ids: `id` is the membership row (NOT a watchlist), and `watchlistId` is the actual watchlist. `listWatchlistPlayers` and `removePlayerFromWatchlist` both need the **`watchlistId`** value — passing the membership `id` fails with "Watchlist not found". The display name lives under the nested `watchlist` object.

### Step 2: List Players in the Target Watchlist

Call `listWatchlistPlayers` with the watchlist's **`watchlistId`** (from Step 1) to see which players are on it. Present the player list so the user can confirm which player to remove.

### Step 3: Find the Player (if not specified)

If the user has not identified a specific player from the watchlist, use `searchPlayers` to find the player by name or attributes. Match the search result against the players listed in Step 2 to get the correct player ID. Follow "Ambiguous player names" below.

### Step 4: Remove the Player

Call `removePlayerFromWatchlist` with the player ID and the watchlist's **`watchlistId`** (from Step 1 — not the membership `id`) to remove the player.

### Step 5: Confirm the Removal

Call `listWatchlistPlayers` with the watchlist's **`watchlistId`** again to verify the player no longer appears in the watchlist. Present the updated watchlist contents to the user as confirmation.

## Ambiguous player names

**Confirm before removing when the name is ambiguous.** Removal is a write action that mutates the user's data. If `searchPlayers` returns multiple plausible matches for the named player (e.g., several players named "Haaland", several with the same surname), do NOT silently remove the first match — ask the user which player they mean (list them with distinguishing detail like club/position). Narrow first to the player who is actually on a watchlist: with a named watchlist, match against its players (Step 2); with no watchlist named, call `getPlayer` for at most the top 3 plausible matches (with more than 3, ask the user which player they mean first) and prefer the candidates whose `watchlists` is non-empty. Only proceed once a single player is unambiguously identified.
