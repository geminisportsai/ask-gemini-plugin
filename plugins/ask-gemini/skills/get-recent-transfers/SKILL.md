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
- **"We" / "us" / "our" / "my team"** → call `listMyOrganizationsTeams` first and use the resulting `team_id`. That's the user-org-owned scope.

### 1b.ii — Decision table for `direction`, `window`, and `season`

| User phrasing | `direction` | `window` | `season` |
|---|---|---|---|
| "signings", "who did [club] sign", "incoming", "buys", "[club] bought" | `IN` | (from window phrasing) | (from season phrasing) |
| "sales", "outgoing", "who left", "who did [club] sell", "departures" | `OUT` | (from window phrasing) | (from season phrasing) |
| "transfers", "movement", "loan deals", "moves", "activity", no direction word | `ALL` | (from window phrasing) | (from season phrasing) |
| "this summer", "summer window", "summer transfers" | (from direction) | `SUMMER` | omit |
| "January", "winter window", "January transfers" | (from direction) | `WINTER` | omit |
| "last summer" | (from direction) | `SUMMER` (previous year — pass the prior `season` if available) | omit |
| "this season", "this year" | (from direction) | omit | current `season_id` |
| "career", "ever", "all time", no temporal cue | (from direction) | omit | omit |

If the user gives both a window and a season ("Arsenal's summer signings this season"), pass both — the resolver pins the window to the season's calendar year (SUMMER → season start year, WINTER → season end year).

Backend semantics (so you don't try to over-constrain or post-filter):

- The Transfer entity has **no** `window` or `season_id` column. The resolver derives both from the transfer date: SUMMER ≈ Jun 1 – Sep 1, WINTER ≈ Jan 1 – Feb 28.
- A `season` ID resolves to a date range `[Jun 1 of start_year, Feb 28 of end_year]`.
- Combining `season` + `window` pins the year (e.g., `season=2023/24` + `window=WINTER` → Jan–Feb 2024 only).
- An unknown `seasonId` returns `[]` — treat this as the empty-result halt below.

Don't try to filter by a window field in your response; pass the args to the tool and trust the returned rows.

### 1b.iii — Empty-result halt

If `listTransfersByTeamId` returns `[]`, respond with this exact wording (substitute the actual club + window/season) and **stop**:

> No transfers found for [club] in [window/season].

Do **not** fan out to `transfersByPlayerIds` as a "let me try another way" fallback. Do **not** silently broaden the filter. Do **not** invent transfers from contract data or other tools.

## Step 2: Empty-resolver halt

For any path that depends on `listMyOrganizationsTeams`, `listMyOrganizationsEligibleTeams`, or `listMyOrganizationsLeagues`, follow the standard halt rule.

If `listMyOrganizationsTeams` returns empty for an "our / my team" prompt, respond with this exact message and stop:

> Your organization doesn't have any teams configured, so I can't identify your team. Please add a team to your organization in the settings and try again.

If `listMyOrganizationsEligibleTeams` has no match for the named club, respond:

> That team isn't in the leagues your organization has added. Please add the relevant league in your organization settings and try again.

If `listMyOrganizationsLeagues` returns empty for a league-scoped prompt, respond:

> Your organization doesn't have any leagues configured. Please add a league in your organization settings and try again.

If the named league isn't in the resolver's results, respond:

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

- `listTransfersByTeamId` returns `[]`: use the exact halt wording from Step 1b.iii.
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
4. **Don't drop or downgrade direction.** "Signings" must map to `IN`, "outgoing" to `OUT`, "loan deals / movement / transfers" with no direction word to `ALL`. Picking `ALL` when the user said "signings" is wrong.
5. **Don't infer a window the user didn't ask for.** Default to last 12 months only when the user gave no temporal cue; do not silently truncate "all transfers" to last 12 months.
6. **Don't invent fees.** Undisclosed fees stay undisclosed.
7. **Don't pick the wrong player.** If `searchPlayers` returns a player whose attributes contradict the prompt, re-search with a more specific query before continuing.
