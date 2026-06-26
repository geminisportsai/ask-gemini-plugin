---
name: player-search
description: >
  Search and filter players by criteria such as position, team, nationality,
  or role archetype. Use when the user wants to find players matching specific attributes.
---

# Player Search

Use this skill when the user wants to find players matching specific criteria -- by name, position, team, nationality, or role archetype.

## Step 1: Determine Search Type

There are two search approaches. Choose based on user intent:

| Approach | Tool | When to use |
|----------|------|-------------|
| Text search | `searchPlayers` | User provides a name or free-text query (e.g., "find Salah", "search for Brazilian wingers") |
| Structured filter | `filterPlayers` | User provides specific attribute criteria (e.g., "all centre backs in the Premier League") |

If the user's intent is ambiguous, prefer `searchPlayers` for name-based queries and `filterPlayers` for attribute-based queries.

## Step 2: Get Valid Filter Values

> **WARNING**: Never pass an empty filter object `{}` to `filterPlayers`. An empty filter causes a 500 server error. Always include at least one filter criterion.

Before calling `filterPlayers`, **always** use reference data tools to get valid IDs and values. This prevents empty results from typos, incorrect values, or mismatched IDs.

| Tool | Returns | Use for |
|------|---------|---------|
| `listPositions` | Valid position names and IDs | `positionIds` filter |
| `listMyOrganizationsTeams` | Teams in the user's active organization (one or more) | `teamIds` filter — both for named teams and for resolving first-person phrases like "my roster", "my team", "my squad", "our team", "our players", "the roster" |
| `listMyOrganizationsLeagues` | Leagues the user's organization has added (curated subset, not every league worldwide) | `leagueIds` filter — resolve league names like "Championship", "Serie A", "Ligue 1" to IDs |
| `listNationalities` | Valid nationality values | `nationalities` filter |
| `listRoleArchetypes` | Valid role archetype names (e.g., MOPPER, GAMEWEAVER, SAFECRACKER, ROADRUNNER, COMBO FORWARD, NUMBER 6, STOPPER) | `roleArchetypes` filter |

Call the relevant reference tool(s) based on the user's filter criteria **before** constructing the filter query.

### Resolving "My Roster" / "My Team" References

If the user refers to their own team(s) using phrases like **"my roster"**, **"my team"**, **"my squad"**, **"our roster"**, **"our team"**, **"our squad"**, **"our players"**, or **"the roster"**, call `listMyOrganizationsTeams` and pass **every** returned team_id into `filterPlayers.teamIds`. An organization can have multiple teams (e.g., first team plus an under-23 or B side) — never assume a single team.

**CRITICAL — empty result handling**: If `listMyOrganizationsTeams` returns an empty list (`edges: []`), the organization has no teams configured. **You MUST stop and respond. Do NOT call any further tools.**

This is a hard stop, not a soft warning. Specifically, do NOT:

- Call `filterPlayers` (with or without `teamIds`) — there is no roster to scope against
- Call any other tool — no further data gathering is allowed
- Mention partial findings, similar players, or alternative leagues — these mislead the user into thinking the data is incomplete rather than the org is unconfigured
- Substitute "all teams", "global", or "every Championship CB" for the missing roster — without a roster, the question literally has no answer
- Try to be "helpful" by providing context — the only helpful response is to direct the user to configure their org

**Respond with this exact message and nothing else** (no preamble, no caveats, no partial data):

> Your organization doesn't have any teams configured, so I can't identify your roster. Please add a team to your organization in the settings and try again.

The same applies to `listMyOrganizationsLeagues`. If the user names a league (e.g., "Championship") and the resolver returns an empty list or does not include that league, the league is out of scope for this organization. Respond with: "Your organization doesn't have the [league name] league configured. Please add it in your organization settings and try again." Do not silently drop the `leagueIds` filter or substitute another league.

Two common shapes (apply only when the resolvers return non-empty results):

- **"My roster" as the search pool** ("Which players on my roster are out of contract next summer?") — pass the resolved team_ids as `teamIds` along with the other filters (`maxMonthsRemaining`, etc.).
- **"My roster" as the benchmark** ("Show me center backs in the Championship who would upgrade my roster") — first call `filterPlayers` with `teamIds` + `positionIds` to get the incumbents, compute the benchmark metric (e.g., max GPR), then call `filterPlayers` again *without* `teamIds`, scoped to the target league + position, with `minGpr` set to the benchmark. If `teamIds` is empty, there is no incumbent to benchmark against and the request cannot be answered — stop.

### When to use `executeSqlQuery` instead of `filterPlayers`

`executeSqlQuery` (with `getSqlSchema`) is available for queries that `filterPlayers` cannot express — for example, filtering players by whether they have scout reports written about them, joining player data with scout report counts, or other ad-hoc analytics that aren't surfaced as a `filterPlayers` parameter.

Strict rules:

1. **Prefer `filterPlayers` first.** If the filter criterion is one of `filterPlayers`'s documented parameters (position, team, league, age, GPR, valuation, role archetype, nationality, etc.), use `filterPlayers`. Do not fall back to SQL.
2. **SQL is the last resort, not the escape hatch.** Reach for SQL only after you've confirmed `filterPlayers` literally cannot express the filter — for example, "players with at least N scout reports" requires joining the `scout_report` table.
3. **The empty-resolver halt above applies to SQL too.** When `listMyOrganizationsTeams` or `listMyOrganizationsLeagues` returns empty, do not "fall back" to SQL with unscoped or substituted filters. The halt is unconditional regardless of which tool you would have used.

## Step 3: Execute Search

### Text Search

Call `searchPlayers` with the search query:

```json
{
  "query": "Salah"
}
```

### Structured Filter

Call `filterPlayers` with the structured criteria, using values obtained from Step 2.

#### Complete Filter Parameter Reference

All filter parameters are optional, but **at least one must be provided**.

##### Lookup Filters (require reference data from Step 2)

| Parameter | Type | Description | Reference Tool |
|-----------|------|-------------|----------------|
| `outerClusterName` | `string` | Cluster name from player clustering analysis | -- |
| `roleArchetypes` | `string[]` | Playing style classifications | `listRoleArchetypes` |
| `positionIds` | `string[]` | Position UUIDs | `listPositions` |
| `leagueIds` | `string[]` | League UUIDs | `listMyOrganizationsLeagues` |
| `teamIds` | `string[]` | Team UUIDs | `listMyOrganizationsTeams` |
| `nationalities` | `string[]` | Nationality names (e.g., "Brazil", "England") | `listNationalities` |

##### Demographic Filters

| Parameter | Type | Description |
|-----------|------|-------------|
| `minAge` | `number` | Minimum player age |
| `maxAge` | `number` | Maximum player age |
| `minHeight` | `number` | Minimum height in **metres** (e.g. 1.88) |
| `maxHeight` | `number` | Maximum height in **metres** (e.g. 1.95) |

> **Height is in metres** (matching the `filterPlayers` tool's own parameter docs) — pass `1.88` for "above 1.88m", not centimetres. Pass a single height filter value once; do not retry the same query with different unit conventions.

##### Performance Metric Filters

| Parameter | Type | Description |
|-----------|------|-------------|
| `minGpm` | `number` | Minimum Gemini Player Metric score |
| `maxGpm` | `number` | Maximum Gemini Player Metric score |
| `minGpr` | `number` | Minimum Gemini Player Rating score |
| `maxGpr` | `number` | Maximum Gemini Player Rating score |
| `minVaep` | `number` | Minimum VAEP (Valuing Actions by Estimating Probabilities) score |
| `maxVaep` | `number` | Maximum VAEP score |
| `minMinutesPlayed` | `number` | Minimum minutes played |
| `maxMinutesPlayed` | `number` | Maximum minutes played |

##### Contract and Valuation Filters

| Parameter | Type | Description |
|-----------|------|-------------|
| `minMonthsRemaining` | `number` | Minimum contract months remaining |
| `maxMonthsRemaining` | `number` | Maximum contract months remaining |
| `minValuation` | `number` | Minimum player valuation |
| `maxValuation` | `number` | Maximum player valuation |

##### Fit Score Filters

| Parameter | Type | Description |
|-----------|------|-------------|
| `minTeamStyleFit` | `number` | Minimum team style fit score |
| `maxTeamStyleFit` | `number` | Maximum team style fit score |
| `minFitScore` | `number` | Minimum overall fit score |
| `maxFitScore` | `number` | Maximum overall fit score |

##### Outfield Player Categorical Scores

| Parameter | Type | Description |
|-----------|------|-------------|
| `minCarrying` | `number` | Minimum carrying categorical score |
| `maxCarrying` | `number` | Maximum carrying categorical score |
| `minDribbling` | `number` | Minimum dribbling categorical score |
| `maxDribbling` | `number` | Maximum dribbling categorical score |
| `minPassing` | `number` | Minimum passing categorical score |
| `maxPassing` | `number` | Maximum passing categorical score |
| `minShooting` | `number` | Minimum shooting categorical score |
| `maxShooting` | `number` | Maximum shooting categorical score |

##### Goalkeeper Categorical Scores

| Parameter | Type | Description |
|-----------|------|-------------|
| `minGkPassing` | `number` | Minimum goalkeeper passing score |
| `maxGkPassing` | `number` | Maximum goalkeeper passing score |
| `minGkPositioning` | `number` | Minimum goalkeeper positioning score |
| `maxGkPositioning` | `number` | Maximum goalkeeper positioning score |
| `minGkShotStopping` | `number` | Minimum goalkeeper shot stopping score |
| `maxGkShotStopping` | `number` | Maximum goalkeeper shot stopping score |

##### Custom Statistics

| Parameter | Type | Description |
|-----------|------|-------------|
| `customStats` | `CustomStatFilterInput[]` | Organization-specific custom statistics |

Each `CustomStatFilterInput` object has:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `statName` | `string` | Yes | Name of the custom statistic |
| `minValue` | `number` | No | Minimum value threshold |
| `maxValue` | `number` | No | Maximum value threshold |

#### Filter Examples

Find young centre backs in the Premier League with high GPR:

```json
{
  "positionIds": ["<centre-back-id-from-listPositions>"],
  "leagueIds": ["<premier-league-id-from-listMyOrganizationsLeagues>"],
  "minAge": 18,
  "maxAge": 23,
  "minGpr": 70
}
```

Find Brazilian players with expiring contracts:

```json
{
  "nationalities": ["Brazil"],
  "maxMonthsRemaining": 12,
  "minMinutesPlayed": 900
}
```

Find goalkeepers with strong shot stopping:

```json
{
  "positionIds": ["<goalkeeper-id-from-listPositions>"],
  "minGkShotStopping": 75
}
```

Find players with custom organization statistics:

```json
{
  "minAge": 21,
  "maxAge": 28,
  "customStats": [
    { "statName": "Expected Goals", "minValue": 5.0 },
    { "statName": "Pressures Per 90", "minValue": 15.0, "maxValue": 30.0 }
  ]
}
```

## Step 4: Enrich Results

For any players the user wants to know more about, call `getPlayerBioDataByPlayerId` with the player's ID to retrieve detailed biographical information including:

- Full name and date of birth
- Current club and contract details
- Physical attributes
- Nationality and citizenship

## Step 5: Get Performance Metrics (Optional)

If the user wants performance data for a found player, call `listPlayerSeasonWeightedAvgMetricsByPlayerId` to retrieve seasonal weighted average metrics. This provides statistical context beyond the basic search results.

## Response Formatting

- Never mention database table names, column names, SQL queries, joins, or any data retrieval methods in your answer
- Use human-friendly names for all metrics: say "GPR" not "TIME_DECAYED_GPR", "Fit Score" not "FIT_SCORE", "Valuation" not "PLAYER_VALUATION"
- GPR stands for "Gemini Player Rating" -- never say "General Performance Rating" or "Gemini Performance Rating"
- Focus on insights and results, not how data was retrieved

## Step 6: Present Results

After gathering results:

1. Present matching players in a clear table or list format.
2. Highlight the attributes most relevant to the user's search criteria.
3. Offer to drill deeper into any specific player (see the `player-summary` skill for comprehensive profiles).
4. If no results are found, suggest broadening the search criteria or checking filter values.

### Present EVERY player the filter returned — never collapse to one

When `filterPlayers` returns multiple players, your response MUST present **all** of them — never silently narrow a multi-result list down to a single player. This is especially important for category prompts ("left wingers in Serie A", "centre backs above 1.88m aged 21-25", "the most physically impressive wingers"): the user asked for the set, so enumerate the full set the tool returned.

- If the tool returns N players, list all N (table or bulleted list), and state the count: "12 players matched — here are all 12:".
- If N is large and you deliberately show only the top few, say so explicitly with both numbers: "24 players matched; here are the top 10 by GPR:". Never present a top-N as if it were the complete result.
- Do **not** answer a category query with a single player unless `filterPlayers` actually returned exactly one. Reporting one when several were returned is a defect, not a summary.
- Order the list by the metric most relevant to the user's phrasing (e.g. "most physically impressive" → carrying/physical scores; otherwise GPR descending) so the enumeration is still useful.

### Enrich and rank — never report "no data" when an aggregate score is null

The list/connection tools (`filterPlayers`, `searchPlayers`, `listSimilarPlayers`) return **scalars-only** Player nodes — they do **not** include inline `bioData` or categorical scores. To rank a matched set by an attribute you must enrich each player individually via `getPlayerBioDataByPlayerId` (or `getPlayerSeasonCategoricalScores`), both already in this tool set.

**This includes GPR.** GPR (Gemini Player Rating) is **not** on the scalar filter nodes either, but it **is** returned by `getPlayerBioDataByPlayerId`. So when a prompt needs ranking or comparison by GPR — including roster-upgrade questions like "center backs who would **upgrade my roster**" (compare candidates' GPR against your current players' GPR) — enrich each relevant player via `getPlayerBioDataByPlayerId` and read GPR from there. **Never tell the user "GPR is unavailable" or refuse to rank** just because GPR was absent from the filter results — that is a copout and a defect. Enrich first; only state GPR is unavailable if `getPlayerBioDataByPlayerId` returns a null GPR for every player.

When the aggregate score a prompt seems to ask for is **null or absent**, do **not** conclude the data is missing and offer alternatives. The aggregate `PlayerBioData.physicalScore` in particular is currently null for players, but the granular `*CategoricalScore` fields **are** populated (`carryingGlobalCategoricalScore`, `aerialsGlobalCategoricalScore`, `dribblingGlobalCategoricalScore`, `passingGlobalCategoricalScore`, each with `Global` and `Positional` variants). So:

- For "most physically impressive" (and similar physicality prompts), enrich the matched players and rank on the relevant granular scores — physical → carrying / aerials / dribbling categorical scores — rather than reporting that no physical data exists.
- Treat a null aggregate as a signal to fall back to the granular fields, not as "no data". Only say data is unavailable if the granular `*CategoricalScore` fields are also null for every matched player.
- Map the user's phrasing to the right granular fields (e.g. "strong in the air" → aerials, "good on the ball / carries it well" → carrying / dribbling), enrich, then present the full set ranked on that field per the ordering guidance above.
