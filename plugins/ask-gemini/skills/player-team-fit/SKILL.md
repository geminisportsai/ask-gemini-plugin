---
name: player-team-fit
description: >
  Score a player against one or more teams using the team-style fit model.
  Use when the user asks how well a player fits a specific team, which
  teams a player fits best, who they could be sold to, or to compare a
  player's fit across multiple named teams.
---

# Player Team Fit

Use this skill when the user wants a **team-style fit score** between a player and one or more teams. The fit score is on a 0–100 scale (higher = better stylistic match) and is computed server-side from manager-vector similarity over the player's actual manager history. **Do not compute the score yourself or fall back to SQL** — the math is non-trivial and the GraphQL tools below are the only correct path.

## Step 1: Resolve the player to an ID

Every prompt for this intent names at least one player. Call `searchPlayers` first with the player's name to get their player ID. If the result is empty or ambiguous, tell the user you couldn't identify the player and ask for clarification — do not proceed.

## Step 2: Pick the right tool for the prompt shape

| Prompt shape | Tool | Notes |
|---|---|---|
| "How does X fit team Y?" (one team) | `getPlayerTeamFit` | Resolve team Y via `listMyOrganizationsEligibleTeams` (see Step 3), then call with `(playerId, teamId)`. |
| "Compare X's fit at A vs B" (2 teams) | `getPlayerTeamFit` × 2 | One call per team, present side-by-side. |
| "How does X fit A, B, C?" (3+ named teams) | `rankTeamsByPlayerFit` with `teamIds` | Resolve all named team IDs first, pass as `teamIds`. Sorted result is convenient. |
| "What teams does X fit best?" / "Who could I sell X to?" (open-ended) | `rankTeamsByPlayerFit` with `leagueIds` | **Always scope by ALL of the org's leagues.** Collect every league with `listMyOrganizationsLeagues` (see "Getting every league" below), then pass every league_id as `leagueIds`. Do not call this tool fully unscoped (only `playerId`, no `teamIds`, no `leagueIds`) — see pitfall 5. |
| "What [league] teams fit X?" (league-scoped) | `rankTeamsByPlayerFit` with `leagueIds` | Resolve the league via `listMyOrganizationsLeagues` — read every page (see "Getting every league") — and pass it as `leagueIds`. |

**Selling ("Who could I sell X to?", "Where could X move?")**: the answer is a list of possible buyers, so a club that cannot buy the player is not an answer. These exclusions apply only for selling — for "What teams does X fit best?" exclude nothing.

- Exclude the organization's own clubs (`listMyOrganizationsTeams`): drop the organization's clubs by team id.
- Exclude the player's current club (`clubName` from `searchPlayers`), matched by name — there is no id for it here. **Never use your own knowledge of which club the player plays for** — your memory is often out of date (loans, transfers) and the only allowed source is `clubName` in this turn's `searchPlayers` result. If `clubName` is missing or null (`teamName: null` does not count as a club), remove no club as the player's current club — keep every club the ranking returned other than the organization's own — do not claim the current club was excluded, and say in one line that you couldn't confirm the player's current club.
- Call `rankTeamsByPlayerFit` with `first` set to N + 1 + the number of clubs `listMyOrganizationsTeams` returned, capped at 50 (N = the number of teams you will show, 10 by default), so the list still has N teams after the exclusions. Say in one line which clubs you left out.

`first` defaults to 10 on `rankTeamsByPlayerFit`. Use a smaller value (e.g., 5) when the user asks for a top-N explicitly ("top 3 teams"). Maximum is 50.

### Getting every league

`listMyOrganizationsLeagues` is **paged** — it returns 20 leagues per page by default, and an organization commonly has more (the big five leagues are often not on the first page). Call it with `first: 100`, and while `pageInfo.hasNextPage` is true, call it again with `after: <pageInfo.endCursor>`. If a result is marked truncated, continue from the `resumeAfterCursor` it gives instead. Use the league_ids from **every** page. Scoring only the first page silently drops whole leagues, so the same question gives different teams depending on how many pages were read.

## Step 3: Resolve team / league names to IDs before calling the fit tools

- **Named teams** ("Liverpool", "Bayern Munich", "Real Madrid"): call `listMyOrganizationsEligibleTeams` with `search: "Bayern Munich"` and pick the matching team_id from the result. **Do not** use `listMyOrganizationsTeams` for this — that's the narrow list of teams the org actively manages (typically just the user's own club), and most named teams won't be in it. The eligible-teams list covers every team in the leagues the org has added, which is the same scope the fit math operates against.
  - **Search by the club's full name.** The search matches names, not nicknames or abbreviations — "PSG" finds nothing. Expand a short name before searching: PSG → Paris Saint-Germain, Spurs → Tottenham Hotspur, Man Utd / Man U → Manchester United, Man City → Manchester City, Barça → Barcelona, Bayern → Bayern Munich, Inter → Inter Milan, Atleti → Atlético Madrid, Juve → Juventus, BVB → Borussia Dortmund. If a search still finds nothing, retry once with the other form of the name.
  - **Pick the senior side in the expected league.** Results are not ranked and include youth, reserve and B sides (U19, U21, II, B) and same-name clubs abroad (a "Liverpool" in Uruguay, "Hotspurs" in Malta). Choose the senior first team in the league the user means — e.g., Liverpool in the Premier League.
- **Named leagues** ("Premier League", "Bundesliga"): collect every league (see "Getting every league") and pick the matching league_id. If the league isn't in the full list, it's out of scope for this organization — tell the user, do not silently drop the `leagueIds` filter.
- **If a named team still can't be found** after expanding the name, say so plainly: "I couldn't find a team called '<name>' — try the club's full name (e.g., Paris Saint-Germain)." Do **not** say the team's league isn't covered. Never silently substitute a different team or skip the filter, and still score the teams you did find.

## Step 4: Handle empty / null results

- `getPlayerTeamFit` returns `null` when no manager-vector data is available for that (player, team) pair. Surface that fact to the user ("I don't have stylistic data for [player] at [team]") — do not guess a score or fall back to a different tool.
- `rankTeamsByPlayerFit` returns an empty array when the player has no manager-context history in the org. Tell the user — do not substitute an unscoped player list or invent a ranking.

## Step 5: Present the result

- For a single (`getPlayerTeamFit`) score: state the score and what it means in one sentence (e.g., "Bukayo Saka's fit score at Liverpool is 78/100 — a strong stylistic match.").
- For a ranked list (`rankTeamsByPlayerFit`): present the teams in order with their fit scores. Brief, no long preamble. If the list is shorter than the user asked for, note why (e.g., "Only 4 teams in your organization have stylistic data for this player.").
- Never invent or paraphrase the score. Pass through what the tool returned.
- Don't mention table names, manager vectors, or implementation details. Speak about "stylistic fit" or "team style fit" only.

## Common pitfalls

1. **Don't use `executeSqlQuery`** — this skill has no SQL tools. If you find yourself wanting to query `manager_vector` or `player_manager_context`, you're going the wrong way.
2. **Don't skip player name resolution** — calling `getPlayerTeamFit` with a name instead of an ID will fail. `searchPlayers` first, always.
3. **Don't default to "all teams"** when the user names specific teams. If they say "fit at Liverpool, Bayern, and PSG" use `rankTeamsByPlayerFit` with `teamIds`, not without — the user wants exactly those three teams scored.
4. **Don't substitute a different metric** if fit data isn't available. GPR or VAEP are not interchangeable with team-style fit.
5. **Don't call `rankTeamsByPlayerFit` fully unscoped.** A call with only `playerId` (no `teamIds` and no `leagueIds`) has failed with a 500 "query too broad" error in the past and is slower than a scoped call. For open-ended "best fit" / "who could I sell to" prompts, collect **every** league (all pages) and pass them as `leagueIds`. If you do hit a 500, **do not** give up or fall back to a generic profile-based answer — collect the leagues and retry with `leagueIds`.
6. **Don't stop at the first page of leagues.** One page of `listMyOrganizationsLeagues` is not the organization's leagues — follow `hasNextPage` to the end (see "Getting every league").
