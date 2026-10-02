---
name: get-recent-transfers
description: >
  Look up transfer history or recent transfer activity — a single player's
  transfers, a named club's recent in/out moves, or transfers in a league
  within a window. Use when the user asks about transfers, signings, or
  career club history.
---

# Get Recent Transfers

Use this skill when the user asks about **transfers** — a single player's transfer history, a club's recent signings or sales, or transfers within a league in a given window. This intent is distinct from `lookup_contracts` (which covers active contract clauses) and from `summarize_player` (which covers overall profile).

## Step 1: Pick the right tool path from the user's phrasing

Phrasing determines whether the query is keyed by a player or by a club.

| Prompt shape | Resolution path | Final tool |
|---|---|---|
| "Where has X played?" / "List X's transfers" / "X's career clubs" (single player) | `searchPlayers` → resolve `player_id` | `transfersByPlayerId` |
| "Compare transfers of X and Y" (multiple named players) | `searchPlayers` per name → list of `player_id`s | `transfersByPlayerIds` |
| "Did X move clubs?" / "Has X been transferred?" (yes/no on a specific player) | `searchPlayers` → `player_id` | `transfersByPlayerId` |
| "What did [club] sign this window?" / "[club]'s signings" / "[club]'s incoming" / "[club]'s outgoing" / "[club]'s loan deals" (named club, any direction) | See **Step 1b** below | `listTransfersByTeamId` |
| "Our recent signings" / "Who did we sign?" / "My team's outgoings" (user's own org) | See **Step 1b** below | `listTransfersByTeamId` |

For per-player paths (rows 1-3), if `searchPlayers` returns the wrong player (different position, club, or nationality than the user implied), **re-call `searchPlayers` with a more specific query** before continuing. Do not pick the first hit when it contradicts the prompt.

## Step 1b: Club-scoped transfer queries — `listTransfersByTeamId`

When the user names a club (or refers to their own org's team) and asks about that club's transfer activity, use `listTransfersByTeamId`. This is the **only** correct tool for club-scoped transfer queries. **Do not fan out** from `listMyOrganizationsEligibleTeams` + `transfersByPlayerIds` — that was a prior workaround for a tool gap and doesn't scale past a tiny roster.

### 1b.i — Resolve the team to a `teamId`

- **Named club** ("Arsenal", "Bayern", "Real Madrid", "Chelsea") → call `listMyOrganizationsEligibleTeams` with `search: "<club name>"` and pick the matching `team_id`. The eligible-teams resolver covers every team in the leagues the org has added. If the named club isn't in the result set, follow the empty-resolver halt in **Step 2** — do not fabricate transfers from an unknown team and do not substitute a different club.
  - **Short names work.** The search resolves common short names itself ("PSG", "Spurs", "Man Utd", "Man City", "Wolves"), so pass the name the user used. It also returns other clubs whose names contain the word (a "Spurs" search also finds Maltese "Hotspurs" clubs), so apply the two rules below.
  - **Pick the senior side.** The results also hold a club's youth, reserve and women's teams (U19, U21, U23, B or II sides, women). Use the senior men's side unless the user asked for one of those.
  - **Prefer the expected country.** When clubs share a name (a "Chelsea" in Ghana as well as England), read each result's `league` and pick the club in the country the user means — for a well-known club, its home league.
- **"We" / "us" / "our" / "my team"** → call `listMyOrganizationsTeams` first and use the resulting `team_id`. That's the user-org-owned scope.

### 1b.ii — Decision table for `direction`, `window`, and `season`

| User phrasing | `direction` | `window` | `season` |
|---|---|---|---|
| "signings", "who did [club] sign", "incoming", "buys", "[club] bought" | `IN` | (from window phrasing) | (from season phrasing) |
| "sales", "outgoing", "who left", "who did [club] sell", "departures" | `OUT` | (from window phrasing) | (from season phrasing) |
| "transfers", "movement", "moves", "activity", no direction word | `ALL` | (from window phrasing) | (from season phrasing) |
| "loan deals", "loans", "loaned out", "loan returns" | `OUT` and `IN`, as two calls (see **1b.v**) | (from window phrasing) | (from season phrasing) |
| "this summer", "summer window", "summer transfers" | (from direction) | `SUMMER` | omit |
| "January", "winter window", "January transfers" | (from direction) | `WINTER` | omit |
| "last summer" | (from direction) | `SUMMER` (previous year — pass the prior `season` if available) | omit |
| "this season", "this year", "last season", "2025/26" | (from direction) | omit | season `id` from `listSeasons` (see **1b.iii**) |
| "loan deals", "loans", "loaned out", "loan returns" with no temporal cue | `OUT` and `IN`, as two calls (see **1b.v**) | omit | this season's `id` from `listSeasons` (see **1b.iii**) |
| "career", "ever", "all time", or no temporal cue on a non-loan question | (from direction) | omit | omit |

If the user gives both a window and a season ("Arsenal's summer signings this season"), pass both — the resolver pins the window to the season's calendar year (SUMMER → season start year, WINTER → season end year).

Backend semantics (so you don't try to over-constrain or post-filter):

- The Transfer entity has **no** `window` or `season_id` column. The resolver derives both from the transfer date: SUMMER ≈ Jun 1 – Sep 1, WINTER ≈ Jan 1 – Feb 28.
- A `season` ID resolves to a date range `[Jun 1 of start_year, Feb 28 of end_year]`.
- Combining `season` + `window` pins the year (e.g., `season=2023/24` + `window=WINTER` → Jan–Feb 2024 only).
- An unknown `seasonId` returns `[]` — treat this as the empty-result halt below.

Don't try to filter by a window field in your response; pass the args to the tool and trust the returned rows for the window and season. Loan questions are the one case where you filter the rows yourself — see **1b.v**.

### 1b.iii — Resolve a named season with `listSeasons`

When the question names a season — "this season", "last season", "2025/26", "2024-25" — call `listSeasons` once, pick the matching season, and pass its `id` as `season`. Never send a season question without `season`: the unscoped call returns the club's most recent transfers, not that season's.

- `listSeasons` returns two seasons for most years: a **split season** (`endYear` = `startYear` + 1, e.g. 2026/27) and a **calendar season** (`endYear` = `startYear`). Both carry the same `displayYear`, so never choose by `displayYear`.
- Clubs in leagues that run autumn to spring (England, Spain, Germany, Italy, France, Portugal, the Netherlands and most of Europe) use the **split season**. Use the calendar season only for leagues that play inside one calendar year (e.g. MLS, Brazil, Scandinavia).
- "this season" is the split season whose `startYear` is the current year when today is on or after 1 June, and the previous year before that. "last season" is the one before it. "2025/26" is `startYear` 2025, `endYear` 2026.
- A loan question with no time cue means this season: resolve it as "this season" and name the season in the answer (e.g. "in 2026/27"), so the user can see which season was used. A loan question that says "ever" or "all time" stays unscoped.
- Pass `first: 100` on every `listTransfersByTeamId` call. The tool's default of 20 cuts a busy club's season off partway, and 100 is the server-side cap.

### 1b.iv — Empty-result halt

If `listTransfersByTeamId` returns `[]`, respond with this exact wording (substitute the actual club + window/season) and **stop**:

> No transfers found for [club] in [window/season].

Do **not** fan out to `transfersByPlayerIds` as a "let me try another way" fallback. Do **not** silently broaden the filter. Do **not** invent transfers from contract data or other tools.

### 1b.v — Loan questions

When the user asks about **loans** ("loan deals", "who did [club] loan out", "loan signings", "loan returns"), follow these steps in order. A transfer row has no type field yet; the server marks what it can.

**Step A — Fetch the requested period.** Call `listTransfersByTeamId` with `direction: "OUT"` and again with `direction: "IN"`, each with the `season` and `window` resolved in **1b.ii** (only the ones the question names), `first: 100` and `feeDisclosed: false`. "Loan deals last summer" passes `window: "SUMMER"` with last summer's `season`, so no winter move arrives; "loan deals this season" passes the `season` alone. With `feeDisclosed: false` the server returns only moves whose fee is undisclosed or mentions a loan; amounts and free transfers never arrive. Two calls rather than one `ALL` call because a tool result over 50,000 characters is cut, and a busy club's season in one call loses its oldest moves. If a call without a `window` returns a result ending with "[truncated from", that direction did not arrive whole: call it again per window — `window: "SUMMER"`, then `window: "WINTER"`, each with the same `season` — and use those rows instead. If a call that already had a `window` is truncated, or a window's re-fetch is still truncated, say the list may be incomplete, and never claim a move is absent.

**Step B — Check the server filter is active.** An older backend ignores `feeDisclosed` without an error. The filter is not active when the rows lack the `likelyLoanReturn` field, or any row has a non-null `fee` that does not mention "loan" (an amount or "Free transfer"). In that case skip Steps C and D and follow **Fallback** below.

**Step C — Classify every row from both calls.** Classify every row from both calls, to the last row of each — the server has already dropped paid and free moves. A 30 June outgoing row is classified like any other. Each row is exactly one of:

- **Confirmed loan:** the row's `fee` or `contractDuration` contains "loan" (any case). A row whose `fee` reads "End of loan" (or similar) is a loan return.
- **Likely loan return:** an incoming row with `likelyLoanReturn: true` — the server computed that flag from the player's earlier move out. Never infer a return yourself: a row whose `likelyLoanReturn` is false or null stays an undisclosed move, whatever its date.
- **Undisclosed:** every other row. The data cannot tell a loan from an undisclosed transfer here, so it is not a loan for this answer.

When a player has two rows for the same move (same clubs, a few days apart), that is one move — count it once, with the later date.

**Step D — Write the answer.** List only confirmed loans and likely loan returns. Never list, name or describe an undisclosed move as a possible loan — no "may be loans" list, no player names, no "some of these could be loans".

Before writing, count the moves in your reasoning: "LOAN 1: …", "LOAN 2: …" for each confirmed loan and likely loan return you will list (a duplicate row gets no line of its own), then U = the number of undisclosed moves after merging duplicates. The numbers must match what you write.

- When at least one confirmed loan or likely loan return exists, open with "Here are [Club]'s loan moves in [season]:" and list them under these headings, leaving out any heading with no moves. Give each move's date and, when present, the loan end from `contractExpiryDate` (or `contractDuration`).
  - **Outgoing loans** — confirmed loans where the club is the `sourceTeam`.
  - **Incoming loans** — confirmed loans where the club is the `destinationTeam` and the row is not a return.
  - **Loan returns** — confirmed "End of loan" rows, in either direction.
  - **Likely loan returns** — rows with `likelyLoanReturn: true`. Always say "likely".
- When none qualifies but undisclosed moves exist, say "No moves marked as loans were found for [club] in [season]." — not "no loans": the data doesn't mark loans, so **never say there were no loans** while undisclosed moves remain.
- When U is above 0, end with exactly one line, with no names: "[U] other moves in [season] had no disclosed fee; the transfer data doesn't say whether they were loans, so they aren't listed."
- When the season has no move at all, say exactly:

> No loan deals or undisclosed moves were found for [club] in [season].

Example shape (placeholders, not real players):

> Here are Club Q's loan moves in 2026/27:
>
> **Likely loan returns**
> - Player B, back from Club Y (2026-06-30)
>
> 3 other moves in 2026/27 had no disclosed fee; the transfer data doesn't say whether they were loans, so they aren't listed.

**Fallback — the server filter is not active.** Filter the rows yourself. Go row by row through both results, to the last row of each — the rows are newest first, so the oldest moves sit at the end and are the easiest to miss. A row is considered only when its `fee` is null or contains "loan". Every other row is excluded, whatever the user asked.

- **Permanent — drop it:** never list a row whose `fee` is an amount (such as "€15M" or "€350K") or "Free transfer" in a loan answer. Drop these before you write anything. A fee such as "€10.0M" on a youth signing is still an amount, so the row is dropped.
- Write the answer as in Step D — confirmed loans only, and the same count line for the undisclosed moves — but write no **Likely loan returns** section: without the server's flag there is nothing to label a return.

## Step 2: Empty-resolver halt

For any path that depends on `listMyOrganizationsTeams`, `listMyOrganizationsEligibleTeams`, or `listMyOrganizationsLeagues`, follow the standard halt rule.

If `listMyOrganizationsTeams` returns empty for an "our / my team" prompt, respond with this exact message and stop:

> Your organization doesn't have any teams configured, so I can't identify your team. Please add a team to your organization in the settings and try again.

If `listMyOrganizationsEligibleTeams` has no match for the named club, respond:

> That team isn't in the leagues your organization has added. Please add the relevant league in your organization settings and try again.

`listMyOrganizationsLeagues` is paged (20 per page by default): call it with `first: 100` and, while `pageInfo.hasNextPage` is true, call again with `after: <pageInfo.endCursor>`. A league that is not on the first page is not out of scope.

If `listMyOrganizationsLeagues` returns empty for a league-scoped prompt, respond:

> Your organization doesn't have any leagues configured. Please add a league in your organization settings and try again.

If the named league isn't in the resolver's results after every page has been read, respond:

> Your organization doesn't have the [league name] league configured. Please add it in your organization settings and try again.

Do not silently drop the filter. Do not substitute another club or league. Do not fall back to SQL.

## Step 3: Resolve the time window (per-player paths only)

For per-player queries (`transfersByPlayerId` / `transfersByPlayerIds`), the tool doesn't have a server-side window filter — apply the window in your response:

| User phrasing | Window |
|---|---|
| "this summer", "this summer window" | The most-recent or currently-open summer window (June–September of the current year if open, else the prior summer). |
| "this winter", "January window" | The most-recent winter window (January of the current year, or upcoming if January hasn't started). |
| "this season" | All transfers within the current season's open windows. |
| "last summer / last winter" | The prior corresponding window. |
| "recent", "lately", no temporal phrase | Default to the last 12 months. |
| Named year ("in 2024", "summer 2023") | Literal year + window combination. |
| "career", "ever", "all time" | No window filter — return the full transfer history. |

For club-scoped queries, pass the resolved window/season as args to `listTransfersByTeamId` (see Step 1b.ii) rather than post-filtering.

## Step 4: Call the transfer tool

- Single player → `transfersByPlayerId(playerId)`, post-filter by Step 3 window.
- Multiple players → `transfersByPlayerIds([...playerIds])`, post-filter by Step 3 window.
- Named club / "our team" → `listTransfersByTeamId(teamId, direction, window?, season?)` per Step 1b.

Don't mix paths — a club-scoped prompt goes through `listTransfersByTeamId` only, not a fan-out via `transfersByPlayerIds`.

## Step 5: Empty / null handling

- `listTransfersByTeamId` returns `[]`: use the exact halt wording from Step 1b.iv.
- `transfersByPlayerId` / `transfersByPlayerIds` returns no rows: state plainly that no transfers matched the filters. Do not synthesize transfers from contract data or invent a "no transfer activity" club summary.
- A named player has no career transfers (rare — e.g., one-club player): say so explicitly. That is a valid answer, not a failure.

## Step 6: Present the result

For a single player, render a chronological table:

| Date | From | To | Fee | Window |
|---|---|---|---|---|
| 2024-07-12 | Borussia Dortmund | Real Madrid | €103M | Summer 2024 |
| 2020-08-15 | Birmingham City | Borussia Dortmund | €30M | Summer 2020 |

For club signings, list the players with fee + position:

> Arsenal's summer signings (2025):
> - Martín Zubimendi (CM) — €70M from Real Sociedad
> - Riccardo Calafiori (CB) — €45M from Bologna

If a fee is unknown / unreported, write "undisclosed". Do not estimate.

Pass through the loan / permanent / free-transfer distinction when the data has it.

**Label loan returns as loan returns.** When a transfer record represents a player **returning from loan / end of loan** to his parent club (the move is the player going back to the club that owns him, not a new signing or sale), describe it that way — "returned to <parent club> at the end of his loan" — rather than presenting it as a standard signing or sale. Reporting a loan return as a normal transfer misleads the user about what actually happened.

## Common pitfalls

1. **Don't conflate transfers with contracts.** "Where has Rice played" is a transfer question. "When does Rice's contract end" is a `summarize_player` / `lookup_contracts` question.
2. **Don't use `executeSqlQuery`.** No SQL fallback for transfer data.
3. **Don't fan out from `listMyOrganizationsEligibleTeams` + `transfersByPlayerIds` for a club-scoped prompt.** That was a workaround for a now-closed tool gap. `listTransfersByTeamId` is the only correct path.
4. **Don't drop or downgrade direction.** "Signings" must map to `IN`, "outgoing" to `OUT`, "movement / transfers" with no direction word to `ALL`, and loan questions to two calls, `OUT` and `IN` (see **1b.v**). Picking `ALL` when the user said "signings" is wrong.
5. **Don't infer a window the user didn't ask for.** Default to last 12 months only when the user gave no temporal cue; do not silently truncate "all transfers" to last 12 months.
6. **Don't invent fees.** Undisclosed fees stay undisclosed.
7. **Don't pick the wrong player.** If `searchPlayers` returns a player whose attributes contradict the prompt, re-search with a more specific query before continuing.
