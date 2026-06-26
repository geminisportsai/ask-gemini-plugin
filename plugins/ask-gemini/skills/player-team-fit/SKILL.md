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
| "What teams does X fit best?" / "Who could I sell X to?" (open-ended) | `rankTeamsByPlayerFit` with `leagueIds` | **Always scope by the org's leagues.** First call `listMyOrganizationsLeagues`, then pass every returned league_id as `leagueIds`. Calling this tool with only `playerId` (no `teamIds`, no `leagueIds`) reliably returns a 500 "query too broad" error, so never call it fully unscoped. |
| "What [league] teams fit X?" (league-scoped) | `rankTeamsByPlayerFit` with `leagueIds` | Resolve the league via `listMyOrganizationsLeagues`, pass as `leagueIds`. |

`first` defaults to 10 on `rankTeamsByPlayerFit`. Use a smaller value (e.g., 5) when the user asks for a top-N explicitly ("top 3 teams"). Maximum is 50.

## Step 3: Resolve team / league names to IDs before calling the fit tools

- **Named teams** ("Liverpool", "Bayern Munich", "Real Madrid"): call `listMyOrganizationsEligibleTeams` with `search: "Bayern Munich"` and pick the matching team_id from the result. **Do not** use `listMyOrganizationsTeams` for this — that's the narrow list of teams the org actively manages (typically just the user's own club), and most named teams won't be in it. The eligible-teams list covers every team in the leagues the org has added, which is the same scope the fit math operates against.
- **Named leagues** ("Premier League", "Bundesliga"): call `listMyOrganizationsLeagues` and pick the matching league_id. If the league isn't in the result, it's out of scope for this organization — tell the user, do not silently drop the `leagueIds` filter.
- **If `listMyOrganizationsEligibleTeams` returns no match for the named team**, tell the user that team isn't in the leagues their organization has added — do not silently substitute a different team or skip the filter.

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
5. **Never call `rankTeamsByPlayerFit` fully unscoped.** A call with only `playerId` (no `teamIds` and no `leagueIds`) reliably 500s with "query too broad". For open-ended "best fit" / "who could I sell to" prompts, resolve the org's leagues with `listMyOrganizationsLeagues` first and pass them as `leagueIds`. If you do hit that 500, **do not** give up or fall back to a generic profile-based answer — call `listMyOrganizationsLeagues` and retry with `leagueIds`.
