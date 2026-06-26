---
name: watchlist-removal
description: >
  Remove players from watchlists.
  Use when the user wants to take a player off one of their watchlists.
---

# Watchlist Removal

Use this skill when the user wants to remove a player from one of their watchlists.

## Step 1: List Existing Watchlists

Call `listMyUserWatchlists` to retrieve the user's current watchlists. Present them so the user can identify which watchlist to remove a player from. If the user has already specified the watchlist, proceed directly.

**Use the `watchlistId` field, never `id`.** Each `listMyUserWatchlists` entry is a *membership* record with two ids: `id` is the membership row (NOT a watchlist), and `watchlistId` is the actual watchlist. `listWatchlistPlayers` and `removePlayerFromWatchlist` both need the **`watchlistId`** value — passing the membership `id` fails with "Watchlist not found". The display name lives under the nested `watchlist` object.

## Step 2: List Players in the Target Watchlist

Call `listWatchlistPlayers` with the watchlist's **`watchlistId`** (from Step 1) to see which players are on it. Present the player list so the user can confirm which player to remove.

## Step 3: Find the Player (if not specified)

If the user has not identified a specific player from the watchlist, use `searchPlayers` to find the player by name or attributes. Match the search result against the players listed in Step 2 to get the correct player ID.

**Confirm before removing when the name is ambiguous.** Removal is a write action that mutates the user's data. If `searchPlayers` returns multiple plausible matches for the named player (e.g., several players named "Haaland", several with the same surname), do NOT silently remove the first match — ask the user which player they mean (list them with distinguishing detail like club/position). Where possible, narrow to the player who is actually on the watchlist (Step 2). Only proceed once a single player is unambiguously identified.

## Step 4: Remove the Player

Call `removePlayerFromWatchlist` with the player ID and the watchlist's **`watchlistId`** (from Step 1 — not the membership `id`) to remove the player.

## Step 5: Confirm the Removal

Call `listWatchlistPlayers` with the watchlist's **`watchlistId`** again to verify the player no longer appears in the watchlist. Present the updated watchlist contents to the user as confirmation.
