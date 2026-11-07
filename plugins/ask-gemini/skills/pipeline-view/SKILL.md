---
name: pipeline-view
description: >
  Read the user's pipelines -- who is in a pipeline or in one of its stages, how many
  players are in each stage, which stage a player is in, or which pipelines the user
  has. Read-only: nothing is added, moved or removed.
---

# Pipeline View

Use this skill when the user asks who is in a pipeline, who is in one of its stages, how many players are in each stage, which stage a player is in, or which pipelines they have. This is a read: never add, move or remove a player here.

## Terminology

- The company is Gemini Sports. Never add "Analytics" to the name.
- Say Pipeline (pipeline boards, stages and cards), never Kanban.
- Say football, never soccer.

## User-facing vocabulary

To the user these are **Pipelines**: pipeline boards, with stages and cards. Say "pipeline" in every reply.
Tool names say kanban; never say kanban to the user.
Never show a board's ID to the user; refer to it by name.

A pipeline's columns are its **stages**; the tools call them phases. To the user "stage" and "phase" mean the same thing.

**Never answer from earlier messages in the conversation.** Pipelines change between turns. Call the tools below every time, even when an earlier answer in this chat already described the pipeline.

**Step budget.** A turn has at most 10 steps, and each tool call is one step. Never call a tool once per card, once per stage or once per pipeline. Very large pipelines may only be partly read within the budget; when that happens, say so (see Counting).

## Tools

How each tool is called. The procedures below refer back to this section.

- `searchPlayers` — finds players by name. Each result has an `id`, a `name` and, for some players, a `knownName` (the name they are known by, such as "Pedri").
- `listPlayerKanbanCards` with `playerId: <player id>` — every pipeline the user can see that holds the player, one entry per pipeline with `boardName` and `phaseName` (the player's stage there), at most 200 entries, most recent first. An empty list means the player is on none of the user's pipelines.
- `listMyKanbanBoards` — the user's pipelines, most recently updated first, each with its `name` and its stages' names. Call it with `first: 50` and page with `after: <pageInfo.endCursor>`, reading at most 5 pages; see Step 1 for when to stop.
- `getKanbanBoard` with `id: <pipeline id>` — the pipeline's stages as `phases`, each with an `id`, a `name` and a `position`. Stage order is ascending `position`.
- `listKanbanCards` with `kanbanBoardId: <pipeline id>` and `first: 100` — one entry per player on the pipeline, with its `player` name and `kanbanPhaseId` (the stage the player is in); page `listKanbanCards` with `after: <pageInfo.endCursor>` until `pageInfo.hasNextPage` is false. There is no total count: count the entries yourself, and a count is final only after the last page. If a page comes back empty with a message to start again, restart once from the first page. Never pass `last` or `before`.

## Which stage a player is in

1. Call `searchPlayers` with the name the user gave. Match each result's `name` or `knownName` against it, accent-folded and case-insensitive ("Arda Guler" matches "Arda Güler", "Norgaard" matches "Nørgaard", "Pedri" matches a player whose `knownName` is "Pedri"). If the search finds no one, say you could not find a player by that name.
2. If exactly one candidate matches exactly, call `listPlayerKanbanCards` once with its `id`. If several results match exactly (two players can share a name, such as two called "Ben White"), call `listPlayerKanbanCards` for every exact match, up to 3 calls, before answering, one call per player. Never stop at the first: each exact match is a different player.
3. If there is more than one candidate, or a single candidate whose name does not match exactly, call `listPlayerKanbanCards` for each of up to 3 candidates, closest match first.
4. Report the candidates that are on a pipeline, naming each player as the tool returned the name. Report every entry: "<player> is in the <phaseName> stage of <boardName>." When the player is on several pipelines, give each one. Do not pick one pipeline silently.
5. Ask which player they mean only when two or more candidates are on a pipeline; name each and the pipelines they are on.

If no candidate's `name` or `knownName` matches exactly, do not report silently: name the player you found, so the user sees who was matched, and give that player's stages, starting with this sentence:

> I couldn't find an exact match for "<name>"; the closest is <player>, who is in …

If the question names a pipeline ("Which stage is Ben White in Recruitment Pipeline?"), still make one `listPlayerKanbanCards` call per candidate and report only the entries whose `boardName` matches that pipeline, exact and case-insensitive. If the player is not on it, say so:

> <player> is not on <pipeline>.

You may add where the player is (one or two of the other entries), without listing every pipeline.

If `listPlayerKanbanCards` returns 200 entries, that is its maximum: say the list may be incomplete, and never say the player is not on a pipeline that is missing from it.

If more than 3 results match exactly and none of the 3 you checked is on a pipeline, say how many you checked and never say flatly that the player is not on any of your pipelines: "I checked three players named <name> and none is on your pipelines; tell me more (club or age) to check another." Do not count the matches you did not check.

Only when no candidate is on any pipeline and you checked every exact match, say the player is not on any of your pipelines. Do not list the pipelines, do not say the player does not exist, and do not invent a stage. You may offer to add the player to a pipeline, but do not do it.

Never look for a player by reading each pipeline's cards; `listPlayerKanbanCards` is the only way to answer this.

## Which pipelines the user has

For "What pipelines do I have?", read one page of `listMyKanbanBoards` and name up to 5 of the most recently updated pipelines, saying they are examples of the user's pipelines, and give no count. Offer to read one of them.

## Who is in a pipeline, or how many players per stage

### Step 1: Find the pipeline

Read at most 5 pages of `listMyKanbanBoards` (see Tools). Match the name the user gave against each pipeline's `name`: exact, case-insensitive, ignoring a trailing "pipeline" or "board" on either side ("my Recruitment pipeline" means the pipeline named "Recruitment Pipeline"). Never match part of a name or a similar name: "Loan Targets" is not "Loan Targets 2025".

After a match, keep paging until `pageInfo.hasNextPage` is false or you reach the 5-page cap, so that you find another pipeline with the same name if there is one. Never call `getKanbanBoard` or `listKanbanCards` until paging has ended.

- **Exactly one match**: use it.
- **More than one pipeline with that name**: this is a hard stop. Do not call `getKanbanBoard` or `listKanbanCards` for any of them, and answer for none of them — not the first one, not any one. Ask which one they mean (or rename them in the app to tell them apart). Tell them apart by their order in `listMyKanbanBoards`, which is most recently updated first and usually stays the same between turns unless someone edits one of them: "the most recently updated <name>", "the 2nd most recently updated <name>", "the 3rd most recently updated <name>", and so on, one for each match. Add a pipeline's stages only when they differ between the duplicates. Never invent any other detail to tell them apart: no owner, date, size or player.
- **The user answers with an ordinal** ("the second one", "the 2nd most recently updated"): page `listMyKanbanBoards` again the same way, take the pipeline at the same place among the same-named matches, and go to Step 2. Take the pipeline name from your own previous question. This is not the No pipeline named case: page up to the 5-page cap exactly as in the turn that asked. Lead the answer with which one you read, for example "The 2nd most recently updated Pipeline 1 (stages …):", so the user can see it if the order changed.
- **No pipeline named**: read one page only. Unless the user has exactly one pipeline (use it and name it), ask which pipeline they mean. Do not list every pipeline: name up to 5 as examples and invite the user to name theirs. There is no default pipeline, so never pick one, and make no count.
- **No match, and you read every page** (the last `pageInfo.hasNextPage` was false): stop and answer with this sentence, then offer up to 5 of the user's pipeline names as examples or ask the user to check the name:

  > I couldn't find a pipeline named "<name>".

- **No match, and pages remain after the cap** (you read 5 pages and `pageInfo.hasNextPage` is still true): you have not seen every pipeline, so do not say the pipeline does not exist. Answer with this sentence and ask the user to check the name; do not say how many pipelines you read or skipped:

  > I couldn't find a pipeline named "<name>" among your most recently updated pipelines.

  In either case, do not answer for a different pipeline, and give no members.

### Step 2: Read the pipeline

Call `getKanbanBoard` once for the pipeline, then page `listKanbanCards` to the end. Put each player in the stage whose `id` equals the card's `kanbanPhaseId`.

- **Who is in the pipeline**: list the players by name, grouped by stage in stage order. If the pipeline has no players, say it has no players yet and name its stages in order.
- **How many players per stage**: answer by stage, in stage order, following Counting below.
- **Who is in a stage**: match the stage name the user gave against the stage names, exact and case-insensitive. Never match part of a stage name or a similar stage: "Scouted" is not "Deep Scouting". If it matches, list the players in that stage; if it has no players, say it is empty, because the stage exists. If no stage matches, say the pipeline has no stage by that name and list its real stages in order. Do not answer for a different stage.

### Counting

Give per-stage counts and a total only when the whole pipeline has at most 10 cards and you read every card. Then list every stage in stage order with its count — a stage with no players shows 0 — and a total that equals the sum.

When the pipeline has more than 10 cards, list the players grouped by stage in stage order, numbered within each stage, say "counts are left out for larger pipelines", and state no per-stage count and no total.

If you stop paging `listKanbanCards` before `pageInfo.hasNextPage` is false, for any reason, say so ("Here are the players I could read on <pipeline>; I could not read the rest"), and state no total and no per-stage counts. Never present a partial list or a partial count as complete.

## Rules for every answer

1. Say "pipeline", never "kanban", and show no IDs.
2. Only describe a player with fields the tools returned. Do not add a club, position or rating from memory.
3. Name only pipelines the tools returned. A pipeline the user cannot see is answered as not found.
4. Never state how many pipelines the user has.
5. Don't say you have no access to pipeline data — these tools are the access.
