---
name: kanban-management
description: >
  Manage kanban scouting boards -- list, create, and add players.
  Use when the user wants to organize players on a scouting kanban board.
---

# Kanban Management

Use this skill when the user wants to view, create, or manage their kanban scouting boards.

## Step 1: List Existing Boards

Call `listMyKanbanBoards` to retrieve the user's current kanban boards. Present them so the user can choose which board to work with, or decide to create a new one.

## Step 2: Find the Player — and verify it's the right one

If the user wants to add a player to a board, use `searchPlayers` to find the player by name. **`searchPlayers` returns the first lexicographic match for a name fragment, which is often NOT the famous player the user means** — e.g. "Haaland" frequently matches a lower-tier player (a Norwegian Eliteserien player), not Erling Haaland. Single-name and surname-only queries are especially risky.

Before treating a result as correct, sanity-check the top match against the well-known player the user almost certainly means (club / league / position). If the top result's `currentClub` or league is inconsistent with that, **re-search with the player's full name** ("Erling Haaland"). If the result is still implausible OR more than one plausible match exists, do NOT proceed — ask the user which player they mean (Step 3).

## Step 3: Add Player to Board — HARD pre-mutation gate

**Do NOT call `addPlayerToKanban` until BOTH conditions below are satisfied.** This is a write action that mutates the user's data — never guess the player or the board.

1. **The exact player is confirmed.** Do NOT silently add a player when (a) `searchPlayers` returns multiple plausible matches for the named player, OR (b) the single top match is implausible for the well-known player the user means (wrong club/league/position after re-searching with the full name per Step 2), OR (c) the query was an ambiguous single-name/surname query (e.g. "Haaland", "Rice"). In any of those cases, ask the user which player they mean — list the candidates with distinguishing detail (club/league/position). Only proceed once a single player is unambiguously identified and matches the user's intent.
2. **The target board is unambiguously identified.** If the user named a specific board, resolve it against the Step 1 list. If the user did NOT name one and they have more than one board, do NOT default to the first/most-recent — ask which board to add to (list the options from Step 1). If they have exactly one board, that one is unambiguous and you may use it.

Only after both the player and the board are confirmed, call `addPlayerToKanban` with the resolved player ID and board ID.

## Step 4: Confirm the Addition

Call `getKanbanBoard` with the board ID to verify the player now appears on the board. Present the updated board contents to the user as confirmation.
