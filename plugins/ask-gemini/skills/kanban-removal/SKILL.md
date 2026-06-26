---
name: kanban-removal
description: >
  Remove players from kanban scouting boards.
  Use when the user wants to take a player off one of their kanban boards.
---

# Kanban Removal

Use this skill when the user wants to remove a player from one of their kanban scouting boards.

## Step 1: List Existing Boards

Call `listMyKanbanBoards` to retrieve the user's current kanban boards. Present them so the user can identify which board to remove a player from. If the user has already specified the board, proceed directly.

## Step 2: List Cards on the Target Board

Call `listKanbanCards` with the board ID to see which players are on it. Present the card list so the user can confirm which player to remove.

## Step 3: Find the Player (if not specified)

If the user has not identified a specific player from the board, use `searchPlayers` to find the player by name or attributes. Match the search result against the cards listed in Step 2 to get the correct player ID.

**Confirm before removing when the name is ambiguous.** Removal is a write action that mutates the user's data. If `searchPlayers` returns multiple plausible matches for the named player (e.g., several players named "Haaland", several with the same surname), do NOT silently remove the first match — ask the user which player they mean (list them with distinguishing detail like club/position). Where possible, narrow to the player who is actually on the board (Step 2). Only proceed once a single player is unambiguously identified.

## Step 4: Remove the Player

Call `removePlayerFromKanban` with the player ID and board ID to remove the player from the kanban board.

## Step 5: Confirm the Removal

Call `listKanbanCards` with the board ID again to verify the player no longer appears on the board. Present the updated board contents to the user as confirmation.
