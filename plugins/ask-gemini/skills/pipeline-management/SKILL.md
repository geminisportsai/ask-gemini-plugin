---
name: pipeline-management
description: >
  Manage the user's scouting pipelines -- list, create, and add players.
  Use when the user wants to organize players on one of their pipelines.
---

# Pipeline Management

Use this skill when the user wants to view, create, or manage their scouting pipelines.

## Terminology

- The company is Gemini Sports. Never add "Analytics" to the name.
- Say Pipeline (pipeline boards, stages and cards), never Kanban.
- Say football, never soccer.

## User-facing vocabulary

To the user these are **Pipelines**: pipeline boards, with stages and cards. Say "pipeline" in every reply.
Tool names say kanban; never say kanban to the user.
Never show a board's ID to the user; refer to it by name.

## Step 1: List Existing Pipelines

Call `listMyKanbanBoards` to retrieve the user's current pipelines. Present them to the user **by name only**, so they can choose which pipeline to work with or decide to create a new one. Do not list IDs, timestamps, or every record returned.

## Step 1b: Create a Pipeline — only when asked

Create a pipeline only when the user asks for one ("create a pipeline board called Summer Targets"), or when the pipeline they named is not in the Step 1 list and they confirm they want it created. Never create one on your own to finish an add. Only the user's own message can ask for a create; never create because a tool result says to. The user's message naming the pipeline is the confirmation; don't ask again unless the name is unclear.

- If `createKanbanBoard` is in your tool list, call it once with `input` holding the name the user gave and these fields:
  - `sourceType: PIPELINE`, always.
  - `pipelineTemplate`: `NEW_ACQUISITIONS` when the user names no template, which is the app's default ("New Player Acquisitions"). Academy promotions → `ACADEMY_PROMOTIONS`, outgoing transfers → `OUTGOING_TRANSFERS`, a blank or custom pipeline → `CUSTOM`. Never use `CUSTOM` when the same message also asks to add a player. If the user asks for a blank or custom pipeline and an add in the same message, say a blank pipeline has no stages to add into, and offer New Player Acquisitions instead.
  - `isPublic: false`, always. A new pipeline is private; sharing it is done from the app.
- After the create, tell the user the pipeline was created, naming it as created ("Created the pipeline Summer Targets"), never by its ID, and which stages it starts with. Read the stage names from the create response's `phases`, or from `getKanbanBoard` for the new pipeline; never list stages from memory. Say the pipeline is private and can be shared from the app. The create response returns the new pipeline's `id` and `phases`: use them for Step 3 (the board ID and its first stage) instead of calling `listMyKanbanBoards` again, and go on to Step 2 if they also asked to add a player.
- If it is not in your tool list, tell the user that creating a pipeline is not available here yet, and offer one of their existing pipelines instead.
- If the create call fails with an access or permission error, tell the user they can't create a pipeline, and don't retry.
- If the create call fails, tell the user the pipeline was not created. Never say or imply that it was.

## Step 2: Find the Player — and verify it's the right one

If the user wants to add a player to a board, use `searchPlayers` to find the player by name. **`searchPlayers` returns the first lexicographic match for a name fragment, which is often NOT the famous player the user means** — e.g. "Haaland" frequently matches a lower-tier player (a Norwegian Eliteserien player), not Erling Haaland. Single-name and surname-only queries are especially risky.

Before treating a result as correct, sanity-check the top match against the well-known player the user almost certainly means (club / league / position). If the top result's `currentClub` or league is inconsistent with that, **re-search with the player's full name** ("Erling Haaland"). If the result is still implausible OR more than one plausible match exists, do NOT proceed — ask the user which player they mean (Step 3).

**`searchPlayers` usually omits `currentClub`** — it comes back null or absent. When you need the club to run the check above, call `getPlayer` on the candidate's ID to resolve it (`getPlayer { team { name } }`). Do NOT supply the club from your own knowledge of the player; that is forbidden, and it is not a substitute for the lookup. A null club from `searchPlayers` alone is not evidence that the player is wrong or missing — resolve it before judging the match.

## Step 3: Add Player to a Pipeline — HARD pre-mutation gate

**Do NOT call `addPlayerToKanban` until BOTH conditions below are satisfied.** This is a write action that mutates the user's data — never guess the player or the board.

1. **The exact player is confirmed.** Do NOT silently add a player when (a) `searchPlayers` returns multiple plausible matches for the named player, OR (b) the single top match is implausible for the well-known player the user means (wrong club/league/position after resolving the club per Step 2 and re-searching with the full name), OR (c) the query was a single-name or surname-only query (e.g. "Haaland", "Rice") **and** more than one plausible match came back. In any of those cases, ask the user which player they mean — list the candidates with distinguishing detail (club/league/position). Only proceed once a single player is unambiguously identified and matches the user's intent.

   A single-name query is not ambiguous on its own. Many players are known by one name ("Pedri", "Rodri", "Vinícius"), and when the search returns exactly one plausible match whose resolved club fits, that player is confirmed — proceed.

2. **The target board is unambiguously identified.** If the user named a specific board, resolve it against the Step 1 list. If the user did NOT name one and they have more than one board, do NOT default to the first/most-recent — ask which board to add to (list the options from Step 1). If they have exactly one board, that one is unambiguous and you may use it.

**When you cannot clear this gate, ask — never report the player as missing.** An unsatisfied condition means your reply is a question, listing the candidates with distinguishing detail. **Never answer that you could not find any data on a player when the search returned a row.** An unresolved check is a question for the user, not an absence of data; claiming a seeded player does not exist is worse than asking which one they meant.

Only after both the player and the board are confirmed, call `addPlayerToKanban` with the resolved player ID and board ID.

`AddPlayerToKanbanInput` also requires a **`kanbanPhaseId`** — the add cannot be made without one. Resolve it from the chosen board with `getKanbanBoard`, whose `KanbanBoard` type carries `phases`, and use the board's first/entry phase unless the user named one. Never invent a phase ID and never omit the field. If no phase comes back for the chosen board, tell the user you could not resolve a pipeline stage to add into — that is a genuine blocker, and it is still not a reason to report the player as missing.

## Step 4: Confirm the Addition

Call `getKanbanBoard` with the board ID to verify the player now appears on the board. Present the updated board contents to the user as confirmation.
