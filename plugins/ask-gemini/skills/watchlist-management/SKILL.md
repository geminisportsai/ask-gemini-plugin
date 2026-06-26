---
name: watchlist-management
description: >
  Manage player watchlists -- list, create, and add players.
  Use when the user wants to track players of interest by adding them to a watchlist.
---

# Watchlist Management

Use this skill when the user wants to view, create, or manage their player watchlists.

## Step 1: List Existing Watchlists

Call `listMyUserWatchlists` to retrieve the user's current watchlists. Present them so the user can choose which watchlist to work with, or decide to create a new one.

**Use the `watchlistId` field, never `id`, for watchlist operations.** Each `listMyUserWatchlists` entry is a *membership* record with TWO ids: `id` is the membership row (NOT a watchlist), and `watchlistId` is the actual watchlist. Every watchlist tool — `addPlayerToWatchlist`, `listWatchlistPlayers`, `removePlayerFromWatchlist` — needs the **`watchlistId`** value. Passing the membership `id` fails with "Watchlist not found". The watchlist's display name lives under the nested `watchlist` object; `isDefault` marks the default watchlist.

## Step 2: Find the Player

If the user wants to add a player, use `searchPlayers` to find the player by name or attributes. Confirm the correct player with the user before proceeding.

**Confirm by the data you actually have — do not invent a club.** `searchPlayers` returns scalar identity fields (name, known name) and does **not** include the player's current club/team (it comes back null). When you acknowledge or confirm the player, refer to them by name; do **not** append a club/team/league descriptor (e.g. "Darwin Núñez (Liverpool striker)") unless you fetched it this turn via `getPlayer { team { name } }`. Stating a remembered former club is a correctness bug — players move clubs and the database is the source of truth, not your prior knowledge.

## Step 3: Add Player to Watchlist — HARD pre-mutation gate

**Do NOT call `addPlayerToWatchlist` until BOTH conditions below are satisfied.** This is a write action that mutates the user's data — never guess the player or the watchlist.

1. **The exact player is confirmed.** If `searchPlayers` returns multiple plausible matches for the named player (e.g., several players named "Haaland", several players with the same surname, or ambiguous single-name queries), do NOT silently add the first match — ask the user which player they mean. List candidates using only detail the search actually returned (full name, known name); if you need club/position to tell them apart, fetch it per candidate via `getPlayer { team { name } positions }` rather than guessing it. Only proceed once a single player is unambiguously identified.
2. **The target watchlist is unambiguously identified.** If the user named a specific watchlist, resolve it against the Step 1 list. If the user did NOT name one and they have more than one watchlist, do NOT default to the first/most-recent — ask which watchlist to add to (list the options from Step 1). If they have exactly one watchlist, that one is unambiguous and you may use it.

Only after both the player and the watchlist are confirmed, call `addPlayerToWatchlist` with the resolved player ID and the watchlist's **`watchlistId`** (from Step 1 — not the membership `id`).

## Step 4: Confirm the Addition

Call `getWatchlist` with the watchlist ID to verify the player now appears in the watchlist. Present the updated watchlist contents to the user as confirmation.
