---
name: get-recent-transfers
description: >
  Look up transfer history or recent transfer activity — a single player's
  transfers or a named club's recent in/out moves. It cannot search transfers
  by fee or across a whole league yet. Use when the user asks about transfers,
  signings, or career club history.
---

# Get Recent Transfers

Use this skill when the user asks about **transfers** — a single player's transfer history, or one club's recent signings or sales. This intent is distinct from `lookup_contracts` (which covers active contract clauses) and from `summarize_player` (which covers overall profile).

## Step 0: Questions this skill cannot answer yet — a fee threshold or a whole league

Transfers can be looked up for one player or one club. There is no search by transfer fee and no search across a whole league yet. Check the question for these before calling any tool:

- **A fee threshold** — transfers or players filtered by a fee: "bought or sold for more than 10M", "moved for more than 10M in the past five years", "completed a transfer for more than 20M", "exclude players who moved for more than X", "signings over €30M".
- **A whole league** — transfers across a league, country or competition rather than one club: "transfers in the Hungarian league", "how many players were bought or sold in NB I", "Premier League transfers this summer".

When the question has either and is not about one or a few named players, one named club, or the user's own club ("we", "our"), reply with exactly this, and call no tool:

> I can't search transfers by fee or across a whole league yet. I can show one club's transfers, with each fee as listed, for a window you choose — for example, a club's signings last summer. Which club and window would you like?

- Never call `listTransfersByTeamId` for several clubs to build a league answer, and never answer from a handful of clubs as if they were the league.
- Never give a count, a total, a fee or a list of transfers for a league or for a fee threshold, and never estimate one.
- When a fee threshold comes with one or a few named players ("Has Rice ever moved for more than £100M?", "Show Pedri's transfers over €20M"), one named club ("Arsenal's signings over €20M this summer") or the user's own club ("Our signings over €20M"), do not give the reply above. Open with "I can't filter transfers by fee yet, so here are all of [X]'s transfers, with each fee as listed:" — [X] is the player, the club or the user's club, with the window after "transfers" when the question gives one — and answer as usual (**Step 1** for a player, **Step 1b** for a club) — every row, none dropped, counted or totalled by fee.

## Step 1: Pick the right tool path from the user's phrasing

Phrasing determines whether the query is keyed by a player or by a club.

| Prompt shape | Resolution path | Final tool |
|---|---|---|
| "Where has X played?" / "List X's transfers" / "X's career clubs" (single player) | `searchPlayers` → resolve `player_id` | `transfersByPlayerId` |
| "Compare transfers of X and Y" (multiple named players) | `searchPlayers` per name → list of `player_id`s | `transfersByPlayerIds` |
| "Did X move clubs?" / "Has X been transferred?" (yes/no on a specific player) | `searchPlayers` → `player_id` | `transfersByPlayerId` |
| "What did [club] sign this window?" / "[club]'s signings" / "[club]'s incoming" / "[club]'s outgoing" (named club, any direction) | See **Step 1b** below | `listTransfersByTeamId` |
| "[club]'s loan deals" / "who is on loan at [club]" / "who did we loan out" (named club or the user's own) | See **1b.v** below | `listLoansByTeamId` |
| "Our recent signings" / "Who did we sign?" / "My team's outgoings" (user's own org) | See **Step 1b** below | `listTransfersByTeamId` |

For per-player paths (rows 1-3), if `searchPlayers` returns the wrong player (different position, club, or nationality than the user implied), **re-call `searchPlayers` with a more specific query** before continuing. Do not pick the first hit when it contradicts the prompt.

## Step 1b: Club-scoped transfer queries — `listTransfersByTeamId`

When the user names a club (or refers to their own org's team) and asks about that club's transfer activity, use `listTransfersByTeamId`. This is the **only** correct tool for club-scoped transfer queries; a loan question also calls `listLoansByTeamId` (see **1b.v**). **Do not fan out** from `listMyOrganizationsEligibleTeams` + `transfersByPlayerIds` — that was a prior workaround for a tool gap and doesn't scale past a tiny roster.

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
| "loan deals", "loans", "loaned out", "on loan", "loan returns" | `listLoansByTeamId` with `ALL`, plus one `IN` call for loan returns (see **1b.v**) | omit | this season's `id`, for the loan-returns call only (see **1b.v**) |
| "this summer", "summer window", "summer transfers" | (from direction) | `SUMMER` | omit |
| "January", "winter window", "January transfers" | (from direction) | `WINTER` | omit |
| "last summer" | (from direction) | `SUMMER` (previous year — pass the prior `season` if available) | omit |
| "this season", "this year", "last season", "2025/26" | (from direction) | omit | season `id` from `listSeasons` (see **1b.iii**) |
| "career", "ever", "all time", or no temporal cue on a non-loan question | (from direction) | omit | omit |

If the user gives both a window and a season ("Arsenal's summer signings this season"), pass both — the resolver pins the window to the season's calendar year (SUMMER → season start year, WINTER → season end year).

Backend semantics (so you don't try to over-constrain or post-filter):

- The Transfer entity has **no** `window` or `season_id` column. The resolver derives both from the transfer date: SUMMER ≈ Jun 1 – Sep 1, WINTER ≈ Jan 1 – Feb 28.
- A `season` ID resolves to a date range `[Jun 1 of start_year, Feb 28 of end_year]`.
- Combining `season` + `window` pins the year (e.g., `season=2023/24` + `window=WINTER` → Jan–Feb 2024 only).
- An unknown `seasonId` returns `[]` — treat this as the empty-result halt below.

Don't try to filter by a window field in your response; pass the args to the tool and trust the returned rows for the window and season. Loan questions take their loans from `listLoansByTeamId` — see **1b.v**.

### 1b.iii — Resolve a named season with `listSeasons`

When the question names a season — "this season", "last season", "2025/26", "2024-25" — call `listSeasons` once, pick the matching season, and pass its `id` as `season`. Never send a season question without `season`: the unscoped call returns the club's most recent transfers, not that season's.

- `listSeasons` returns two seasons for most years: a **split season** (`endYear` = `startYear` + 1, e.g. 2026/27) and a **calendar season** (`endYear` = `startYear`). Both carry the same `displayYear`, so never choose by `displayYear`.
- Clubs in leagues that run autumn to spring (England, Spain, Germany, Italy, France, Portugal, the Netherlands and most of Europe) use the **split season**. Use the calendar season only for leagues that play inside one calendar year (e.g. MLS, Brazil, Scandinavia).
- "this season" is the split season whose `startYear` is the current year when today is on or after 1 June, and the previous year before that. "last season" is the one before it. "2025/26" is `startYear` 2025, `endYear` 2026.
- For a calendar-season club, "this season" is the calendar season whose `startYear` is the current year (in October 2026, the 2026 season), and "last season" is the year before.
- A loan question with no time cue means this season: resolve it as "this season" for the **Loan Returns** call and name the season in the answer (e.g. "in 2026/27", or "in 2026" for a calendar-season club), so the user can see which season was used.
- Pass `first: 100` on every `listTransfersByTeamId` call. The tool's default of 20 cuts a busy club's season off partway, and 100 is the server-side cap.

### 1b.iv — Empty-result halt

If `listTransfersByTeamId` returns `[]`, respond with this exact wording (substitute the actual club + window/season) and **stop**:

> No transfers found for [club] in [window/season].

Do **not** fan out to `transfersByPlayerIds` as a "let me try another way" fallback. Do **not** silently broaden the filter. Do **not** invent transfers from contract data or other tools.

### 1b.v — Loan questions

When the user asks about **loans** ("loan deals", "who did [club] loan out", "loan signings", "who is on loan at [club]", "loan returns"), use `listLoansByTeamId`. It lists the club's loans that are running now and already knows which moves are loans, so never decide from a transfer's fee whether a move is a loan.

**Season.** The loan list is current state only: it has no season or window filter and holds no loan history.

- A question about this season, or with no time cue, is answered normally. Name the season in the answer (e.g. "in 2026/27").
- A question about a past season ("loan deals in 2023/24", "last season's loans") is answered with exactly this, and nothing more: "Only current loans are recorded, so I can't list [Club]'s loan deals in [season]. Would you like the loans running now?" Do not build a past season's loans from `listTransfersByTeamId` rows.

**Step A — Fetch the current loans.** With the `teamId` from **1b.i**, call `listLoansByTeamId` once with `direction: "ALL"`. It takes no season or window. Skip this step for a returns-only question. Each row has:

- `direction` — `OUT`: the club's player out on loan at `loanTeamName`; `IN`: a player on loan at the club from `parentTeamName`.
- `playerName`, `parentTeamName` (the club the player is on loan from) and `loanTeamName` (the club the player is on loan at).
- `transfer` — the move that started the loan, with its `date`, `fee` and `marketValue`. It is null when no transfer row matched the loan.

**Step B — Fetch loan returns.** Only when the answer shows **Loan Returns** (see Step C). To resolve this season, first pick the club's season format — split or calendar — from its league, as **1b.iii** says, then take "this season" in that format. Call `listTransfersByTeamId` with `direction: "IN"`, that `season`, `first: 100` and `feeDisclosed: false`. A row is a loan return only when its `fee` reads "End of loan" (or similar) or it has `likelyLoanReturn: true` — the server computed that flag from the player's earlier move out. Never infer a return yourself: any other row from this call is not a loan and is never listed, named or counted. An empty result here is not the **1b.iv** halt; it only means there are no loan returns. If the result ends with "[truncated from", say the loan returns may be incomplete.

**Step C — Write the answer.** The single `ALL` call covers both directions; show only the sections the question asks for:

- "who did [club] loan out", "loans out", "loaned out" → **Outgoing Loans** only.
- "who is on loan at [club]", "loans in", "loan signings" → **Incoming Loans** only.
- "which players returned from loan", "[club]'s loan returns", "who came back from loan" → **Loan Returns** only, from Step B alone.
- "loan deals", "loans" or any other broad loan question → all three sections: **Outgoing Loans**, **Incoming Loans** and **Loan Returns**.

Open with "Here are [Club]'s current loans ([season]):" and list the moves under those headings, leaving out a heading with no moves:

- **Outgoing Loans** — the `OUT` rows: "**[playerName]** → [loanTeamName] · [transfer date] · [transfer fee, or "Undisclosed" when it is null] · market value [marketValue]" (leave the market value out when it is null).
- **Incoming Loans** — the `IN` rows: "**[playerName]** ← [parentTeamName] · [transfer date]".
- **Loan Returns** — the returns from Step B: "**[player]** ← back from [club the player returned from] · [date]". Add "(likely)" to a return known only from `likelyLoanReturn: true`.

When a row's `transfer` is null, write "date and fee not recorded" in place of its date and fee.

When the requested direction has no loans, say so for that direction and list nothing in its place — never fill a loan section with other transfers:

- Outgoing only: "[Club] has no players currently out on loan."
- Incoming only: "[Club] has no players currently in on loan."
- Returns only: "No loan returns were found for [Club] in [season]."
- A general question with `[]` from `listLoansByTeamId`: "[Club] has no players currently out on loan or in on loan." Still give the Loan Returns section when Step B found returns.

End every loan answer with this line: "Loans to clubs in leagues your organization doesn't follow may not appear."

- Every `listLoansByTeamId` row is listed, whatever its fee: a loan with an undisclosed fee is still a loan, shown as "Undisclosed".
- Never list a permanent transfer, a free transfer or an undisclosed-fee move from `listTransfersByTeamId` in these sections: Outgoing Loans and Incoming Loans hold only `listLoansByTeamId` rows, and Loan Returns holds only the returns from Step B.
- Never say the data cannot tell which moves are loans — `listLoansByTeamId` does.

Example shape (placeholders, not real players):

> Here are Club Q's current loans (2026/27):
>
> **Outgoing Loans**
> - **Player A** → Club X · 2026-08-14 · Undisclosed · market value €8.0M
>
> **Incoming Loans**
> - **Player C** ← Club Z · date and fee not recorded
>
> **Loan Returns**
> - **Player B** ← back from Club Y · 2026-06-30 (likely)
>
> Loans to clubs in leagues your organization doesn't follow may not appear.

## Step 2: Empty-resolver halt

For any path that depends on `listMyOrganizationsTeams` or `listMyOrganizationsEligibleTeams`, follow the standard halt rule. A league-wide question never reaches a resolver: it gets the **Step 0** reply.

If `listMyOrganizationsTeams` returns empty for an "our / my team" prompt, respond with this exact message and stop:

> Your organization doesn't have any teams configured, so I can't identify your team. Please add a team to your organization in the settings and try again.

If `listMyOrganizationsEligibleTeams` has no match for the named club, respond:

> That team isn't in the leagues your organization has added. Please add the relevant league in your organization settings and try again.

Do not silently drop the filter. Do not substitute another club. Do not fall back to SQL.

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
- A club's loans → `listLoansByTeamId(teamId, direction: ALL)` plus the loan-returns call, per **1b.v**.

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
4. **Don't drop or downgrade direction.** "Signings" must map to `IN`, "outgoing" to `OUT`, "movement / transfers" with no direction word to `ALL`, and loan questions to `listLoansByTeamId` with `ALL` (see **1b.v**). Picking `ALL` when the user said "signings" is wrong.
5. **Don't infer a window the user didn't ask for.** Default to last 12 months only when the user gave no temporal cue; do not silently truncate "all transfers" to last 12 months.
6. **Don't invent fees.** Undisclosed fees stay undisclosed.
7. **Don't pick the wrong player.** If `searchPlayers` returns a player whose attributes contradict the prompt, re-search with a more specific query before continuing.
8. **Don't fan out club by club across a league.** Calling `listTransfersByTeamId` once per club is not a league search, and a few clubs are not the league. A league-wide or fee-threshold question gets the **Step 0** reply.
