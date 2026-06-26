---
name: sql-analytics
description: >
  Write and execute SQL queries against the sports data warehouse.
  Use when the user asks to rank, compare, or look up player statistics,
  or any question requiring aggregate data across players/teams/seasons.
---

# SQL Analytics

Use this skill whenever the user asks a question that requires querying the sports data warehouse -- player rankings, statistical comparisons, aggregate metrics, filtering by position or club, or any ad-hoc data lookup.

## Step 1: Discover Schema

Call `getSqlSchema` to retrieve the full database schema. This returns tables, columns with data types, foreign key relationships, unique values for filterable columns, and query construction guidelines.

You must call this tool before writing any SQL query to confirm available tables and column names.

## Step 2: Resolve "My Roster" / "My Team" References

If the user refers to their own team(s) using phrases like **"my roster"**, **"my team"**, **"my squad"**, **"our roster"**, **"our team"**, **"our squad"**, **"our players"**, or **"the roster"**, you must first resolve which teams they mean before writing SQL.

Call `listMyOrganizationsTeams` to get the list of teams belonging to the user's active organization. An organization can have one or more teams (e.g., a first team plus an under-23 or B side), so the result is always a list — never assume a single team.

Then filter your SQL with `WHERE p.team_id IN ('<id1>'::uuid, '<id2>'::uuid, ...)` using every team_id returned. Do not use `=`; even single-team orgs should use `IN` so the query continues to work if the org adds more teams later.

**CRITICAL — empty result handling**: If `listMyOrganizationsTeams` returns an empty list (`edges: []`), the organization has no teams configured. **You MUST stop and respond. Do NOT call any further tools** — no `getSqlSchema`, no `executeSqlQuery`.

This is a hard stop. Specifically, do NOT:

- Drop the team filter and run an unscoped query — that leaks data from other organizations
- Mention partial findings, global rankings, or "context" — these mislead the user
- Try to be "helpful" by showing top players globally — the only helpful response is to direct the user to configure their org

**Respond with this exact message and nothing else**:

> Your organization doesn't have any teams configured, so I can't identify your roster. Please add a team to your organization in the settings and try again.

The same applies if the user references a specific team by name (e.g., "rank Arsenal by GPR") and that team isn't in the resolver's result — tell the user the team is out of scope for this organization, don't silently broaden the query.

Examples:

- "Rank my roster by GPR" → call `listMyOrganizationsTeams` → filter by `p.team_id IN (...)` → order by `"TIME_DECAYED_GPR"` DESC
- "What is the average age of my roster?" → call `listMyOrganizationsTeams` → `SELECT AVG(s."AGE") FROM stat.player_stats_pivoted s JOIN public.player p ON p.id = s."PLAYER_ID" WHERE p.team_id IN (...)`
- "Compare Pedri to the midfielders on my roster" → call `listMyOrganizationsTeams` → filter midfielders on those teams, then SELECT comparison columns for both Pedri and the resolved midfielders

## Step 3: Key Tables Reference

The three most important tables for player analytics:

> **WARNING**: The table is `public.scout_report`, NOT `public.scouting_report`. Always use exactly the table names from `getSqlSchema`.

| Table | Schema | Type | Purpose |
|-------|--------|------|---------|
| `public.player` | `public` | Regular table | Player biographical data (names, team affiliation) |
| `stat.player_stats_pivoted` | `stat` | Materialized view | Aggregated player statistics and scores |
| `public.scout_report` | `public` | Regular table | Individual scouting reports with JSONB data column |

Player names live in `public.player`. Statistics live in `stat.player_stats_pivoted`. You almost always need to JOIN them.

## Step 4: Column Reference for `stat.player_stats_pivoted`

Available columns (all UPPERCASE, must be double-quoted):

| Column | Description |
|--------|-------------|
| `"PLAYER_ID"` | Foreign key to `public.player.id` |
| `"FIT_SCORE"` | Overall fitness/quality score (use ONLY when the user explicitly asks about "fit"/"fitness"/"quality"). Stored as a 0–1 decimal — always render as an integer 0–100 (multiply by 100, round), e.g. 0.63 → 63. (The `player_team_fit` tool score is already 0–100; do not multiply that one.) |
| `"TIME_DECAYED_GPR"` | Time-decayed Gemini Player Rating. **Default ranking metric for generic "top"/"best"/"worst players" requests** (best/top → DESC, worst → ASC). |
| `"AGE"` | Player age |
| `"GENERAL_POSITION"` | Broad position category (see Step 6) |
| `"PRIMARY_POSITION"` | Specific position role |
| `"CURRENT_CLUB"` | Current club name |
| `"CURRENT_LEAGUE"` | Current league name |
| `"PLAYER_VALUATION"` | Market valuation (use for "value" queries) |
| `"NATIONALITY"` | Player nationality |
| `"MINUTES_TOTAL"` | Total minutes played |

**Columns that do NOT exist:** `PLAYER_NAME`, `GOALS`, `ASSISTS`, `RATING`. Do not use these.

## Step 5: Column Naming Rules

- **Materialized view columns** (`stat.player_stats_pivoted`): UPPERCASE, MUST be double-quoted.
  Example: `SELECT "FIT_SCORE" FROM stat.player_stats_pivoted`
- **Regular table columns** (`public.player`, `public.team`): snake_case, no quoting needed.
  Example: `SELECT first_name, last_name FROM public.player`

## Step 6: Valid `GENERAL_POSITION` Values

Use these exact strings when filtering by position:

- `'Winger'`
- `'Forward'`
- `'Centre Back'`
- `'Full Back'`
- `'Centre Midfielder'`
- `'Defensive Midfielder'`
- `'Goalkeeper'`

## Step 7: Query Construction Patterns

Follow these rules for every query:

1. **Always use schema-qualified table names**: `public.player`, `stat.player_stats_pivoted`, `public.team`.
2. **Always JOIN for player names**: Player names are not in the stats view. Join like this:
   ```sql
   stat.player_stats_pivoted s JOIN public.player p ON p.id = s."PLAYER_ID"
   ```
3. **Default LIMIT 10**: Always add `LIMIT 10` unless the user requests a different count.
4. **Only SELECT queries**: No INSERT, UPDATE, DELETE, or DDL.
5. **Cast UUIDs explicitly**: `WHERE id = 'value'::uuid`.
6. **Use ILIKE for text matching**: `WHERE t.name ILIKE '%Arsenal%'`.
7. **Use foreignKeys from schema for JOIN paths** between regular tables.
8. **For JSONB columns**: Use `->` for object access and `->>` for text extraction. Check `uniqueValues` from the schema for filterable column values.

### Sorting Reference

| User intent | ORDER BY clause |
|-------------|----------------|
| "Top"/"Best" players (generic) | `ORDER BY "TIME_DECAYED_GPR" DESC` |
| "Worst" players (generic) | `ORDER BY "TIME_DECAYED_GPR" ASC` |
| Explicitly about "fit"/"fitness"/"quality" | `ORDER BY "FIT_SCORE" DESC` |
| "Most valuable" | `ORDER BY "PLAYER_VALUATION" DESC` |
| "Youngest" | `ORDER BY "AGE" ASC` |
| "Most experienced" | `ORDER BY "MINUTES_TOTAL" DESC` |

**Default ranking metric:** for generic "top"/"best"/"worst players" requests, rank by GPR (`"TIME_DECAYED_GPR"`) — best/top descending, worst ascending. Only use `"FIT_SCORE"` when the user explicitly asks about "fit", "fitness", or "quality".

## Step 8: Template Queries

### Player Rankings by Position

Generic "top/best/worst" rankings default to GPR (`"TIME_DECAYED_GPR"`). Only switch to `"FIT_SCORE"` when the user explicitly asks about "fit"/"fitness"/"quality".

```sql
SELECT p.first_name, p.last_name, s."TIME_DECAYED_GPR", s."CURRENT_CLUB", s."GENERAL_POSITION"
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
WHERE s."GENERAL_POSITION" = 'Winger'
ORDER BY s."TIME_DECAYED_GPR" DESC
LIMIT 10
```

### Players by Club

```sql
SELECT p.first_name, p.last_name, t.name AS team_name
FROM public.player p
JOIN public.team t ON t.id = p.team_id
WHERE t.name ILIKE '%Arsenal%'
LIMIT 20
```

### Player Comparison

```sql
SELECT p.first_name, p.last_name, s."FIT_SCORE", s."TIME_DECAYED_GPR",
       s."AGE", s."CURRENT_CLUB", s."GENERAL_POSITION"
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
WHERE p.last_name IN ('Salah', 'Saka', 'Foden')
```

## Step 8a: Team-level aggregates — team GPR / team strength

Some prompts ask for a **team-level** GPR (sometimes phrased as "team strength", "team rating", "team GPR", "team form") rather than a player-level metric. There is no pre-computed team_gpr column — every team-level GPR is an `AVG("TIME_DECAYED_GPR")` over the team's players. This is **not a new intent**; it's a SQL pattern under `lookup_stat` / `rank_players`.

> **Critical — return the AVG, not a per-player breakdown.** When the user asks for a **team's** GPR / strength / rating (e.g., "what is my team's GPR", "what's Arsenal's team strength"), the answer is **the AVG value** — a single number per team. Do **not** return a per-player breakdown unless the user explicitly asked for one ("show me my team's players ranked by GPR"). A 1-row answer for a single team, or N rows for a multi-team org / league ranking, is correct. Listing every player's individual GPR is the wrong shape for these prompts and contradicts the user's question.

Three prompt shapes, each with its own resolution path:

### Shape 1: "My team's GPR" / "What's our team strength?" (first-person)

The user is asking about the team(s) belonging to their active organization. Follow the Step 2 "My Roster" resolution first:

1. Call `listMyOrganizationsTeams`.
2. If the result is empty, **apply the Step 2 empty-resolver halt** — respond with the verbatim "Your organization doesn't have any teams configured…" message and stop. Do NOT broaden to "all teams" or "league averages".
3. Otherwise, build a query that averages `"TIME_DECAYED_GPR"` for players on those team IDs:

```sql
SELECT t.name AS team_name,
       ROUND(AVG(s."TIME_DECAYED_GPR")::numeric, 2) AS team_gpr,
       COUNT(*) AS player_count
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
JOIN public.team t   ON t.id = p.team_id
WHERE p.team_id IN ('<id1>'::uuid, '<id2>'::uuid)
  AND s."TIME_DECAYED_GPR" IS NOT NULL
GROUP BY t.name
```

If `listMyOrganizationsTeams` returns multiple teams (e.g., first team + B side), the `GROUP BY t.name` returns one row per team so the user sees each separately.

### Shape 2: Named team — "What's Arsenal's team strength?" / "Liverpool's GPR"

The user named a third-party club. Do **not** call `listMyOrganizationsTeams` — that resolver is scoped to the user's own organization. Resolve the team via SQL on `public.team`:

```sql
SELECT t.id, t.name
FROM public.team t
WHERE t.name ILIKE '%Arsenal%'
LIMIT 5
```

Pick the matching team_id, but **disambiguate first when the name resolves to more than one team** (see Step 8c-team below — "Chelsea", "Arsenal", "Rangers" etc. can each match several distinct clubs across countries/divisions). If the resolver returns one clear match, proceed; if it returns several, do NOT silently pick one — either ask the user which club they mean, or pick the most likely top-flight club and explicitly state the assumption in your answer. Then run the team GPR aggregate against that team_id:

```sql
SELECT t.name AS team_name,
       ROUND(AVG(s."TIME_DECAYED_GPR")::numeric, 2) AS team_gpr,
       COUNT(*) AS player_count
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
JOIN public.team t   ON t.id = p.team_id
WHERE t.id = '<resolved-team-id>'::uuid
  AND s."TIME_DECAYED_GPR" IS NOT NULL
GROUP BY t.name
```

If the team-resolver query returns zero rows, tell the user you couldn't find that team — do not silently average everyone or pick a similarly-named team.

### Shape 3: League-wide ranking — "Rank the Premier League teams by GPR"

The user wants every team in a league ranked by team GPR. The user may name the league directly ("Premier League", "Bundesliga"). Resolve via SQL with a join through `public.league`:

```sql
SELECT t.name AS team_name,
       l.name AS league_name,
       ROUND(AVG(s."TIME_DECAYED_GPR")::numeric, 2) AS team_gpr,
       COUNT(*) AS player_count
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
JOIN public.team   t ON t.id = p.team_id
JOIN public.league l ON l.id = t.league_id
WHERE l.name ILIKE '%Premier League%'
  AND s."TIME_DECAYED_GPR" IS NOT NULL
GROUP BY t.name, l.name
ORDER BY team_gpr DESC
LIMIT 20
```

`GROUP BY t.name, l.name` keeps the league name available for output without aggregating across leagues. `ORDER BY team_gpr DESC` produces the ranking. Use `LIMIT 20` so the answer fits a typical league size; cap higher if the league has more clubs.

If the league-resolver query returns zero rows, tell the user the named league isn't in scope — do not silently broaden to "all leagues" or substitute a different league.

### Notes that apply to all three shapes

- `AVG("TIME_DECAYED_GPR")` is the canonical team strength expression. If the user explicitly says "team fit score" or "team valuation" instead, swap `"TIME_DECAYED_GPR"` for the corresponding column from Step 4.
- Always filter out NULL GPR (`s."TIME_DECAYED_GPR" IS NOT NULL`) so the average isn't dragged toward zero by missing data.
- Always include `player_count` so the user can judge sample size — a "team GPR" computed over 6 players is much weaker than over 25.
- Round the average to 2 decimal places in the SQL (`ROUND(... ::numeric, 2)`) so the output is clean without post-processing.
- Speak about "team GPR", "team strength", or "team rating" in your response — never mention `TIME_DECAYED_GPR`, `AVG`, or the SQL.

## Step 8b: Scout reports for a SPECIFIC PLAYER — use `organizationScoutReports`, NOT SQL

**Do NOT count or list a specific player's scout reports with SQL over `public.scout_report`.** That table is **organization-scoped only** — it is not filtered by what the current user is permitted to see. A raw `COUNT(*)` therefore returns every report in the org (other scouts' private reports included), which is far more than the player's page shows the user. Even with the archived/parent/processed filters the count is still wrong, because per-user visibility cannot be expressed in SQL here.

Instead, use the permission-scoped **`organizationScoutReports`** query, which returns exactly the reports the user can see and supports a player-name search:

```graphql
organizationScoutReports(filter: { search: "<player full name>" }) { totalCount }
```

1. Pass the player's name as the user gave it (e.g. "Marcus Rashford") as `filter.search`.
2. **`totalCount`** is how many scout reports that player has — report it directly for "how many scout reports for X". It already matches the player page (e.g. Rashford → 4, Mbappé → 3).
3. To list or summarize the reports, read `edges { node { ... } }` from the same query. Never fall back to `executeSqlQuery` over `public.scout_report` for a per-player question.
4. **Comparing two or more players' scout reports** ("compare the scouting reports for X and Y", "what do our scouts disagree about between X and Y") — use **`compareOrganizationScoutReports`** with ALL the players' names in one call: `compareOrganizationScoutReports(playerNames: ["Haaland", "Mbappe"])`. It returns one group per player (in order), each with its own `search`, `totalCount`, and `reports` — including players with zero reports. This is the deterministic way to compare; it cannot collapse names together or drop a player.
   - **One name per array element** — `["Haaland", "Mbappe"]`, never one combined string like `["Haaland and Mbappe"]` (that matches nobody).
   - **Report each group independently.** A group with `totalCount: 0` means that player has no reports — say so for that player and still report the others. Never collapse to "neither has reports" when one group is non-empty.
   - Never read `public.scout_report` via SQL for this, and never tell the user you lack a tool for scout reports.

   **Worked example — "Compare the scouting reports for Haaland and Mbappe":**
   ```
   compareOrganizationScoutReports(playerNames: ["Haaland", "Mbappe"])
     → [ { search:"Haaland", totalCount:0, reports:[] },
         { search:"Mbappe",  totalCount:5, reports:[…] } ]
   ```
   Correct answer states both groups: *"Your scouts have **no reports on Erling Haaland**, so there's nothing to compare on his side. For **Kylian Mbappé** there are **5 reports** — [summarize them]."* Returning "no reports for either player" here is **wrong** — the Mbappe group has 5.

   (For a SINGLE player's scout reports — "how many reports for X", "what do our scouts say about X" — use `organizationScoutReports(filter:{search:"X"})` as in Step 8b above. `compareOrganizationScoutReports` is specifically the multi-player comparison tool.)

**Org-wide scout activity — prefer `organizationScoutReports` with date filters.** For overall scouting-activity counts — e.g. "how many reports did our scouts write this week / this month / this season" — use the scoped `organizationScoutReports` query with its `reportDateFrom` / `reportDateTo` filters and read `totalCount`:

```graphql
organizationScoutReports(filter: { reportDateFrom: "<ISO start>", reportDateTo: "<ISO end>" }) { totalCount }
```

This is permission-scoped and already excludes archived/duplicate/unprocessed rows, so the count is correct without hand-written guards. Resolve the date window from the current date (see the "this season" guidance — a football season spans two calendar years). Raw SQL over `public.scout_report` is a **last-resort fallback** only when the needed aggregate genuinely cannot be expressed through `organizationScoutReports`. If you must fall back to SQL for an org-wide aggregate, link by **id, never by name** (`data->>'playerName'` is frequently empty), and you MUST still filter to active rows or the count is inflated:

```text
WHERE sr.archived_at IS NULL          -- exclude archived
  AND sr.parent_report_id IS NULL     -- exclude child/duplicate rows, keep only top-level
  AND sr.processed_data IS NOT NULL   -- exclude unprocessed rows
  AND sr.organization_id = '<org-id>'::uuid   -- org scope (as already used elsewhere)
```

Never use SQL over `public.scout_report` to answer how many reports a single player has — that is the per-player case above and must go through `organizationScoutReports`.

**Scout-report honesty — only claim a report exists when `organizationScoutReports` actually returns one.** Only tell the user a player has a scouting report (or quote a "scout score") when `organizationScoutReports(filter:{search:<name>}).totalCount` is one or more for that player. If it is zero, say the player has **no** scout report — do not soften it, do not infer one. In particular, **never infer the existence of a scout report from the presence of a GPR, GPM, fit score, or any other metric.** A GPR is computed for almost every tracked player and says nothing about whether a human scout report exists. The two are unrelated data sources — having a GPR does not mean a scout report was written, and a player with a strong GPR routinely has zero scout reports. Report exactly what `organizationScoutReports` returns.

## Step 8b-rank: Ranking or FILTERING players by scout reports — use `organizationScoutReports`, NOT a `public.scout_report` SQL join

Some questions rank or filter players by whether **we** have scouted them, how many reports they have, or how our scouts rate them:

- "Who are the top 5 players we have scout reports on?"
- "Which players have our scouts written the most reports about / rated highest?"
- "Show me 5 moppers under 10M that our scouts recommend / that we've scouted."

The set of scouted players and their per-player report counts MUST come from the permission-scoped `organizationScoutReports` query — **never** from a SQL join against `public.scout_report`. That table is org-scoped, ignores per-user visibility, and over-counts: it surfaces players whose reports the user can't actually see and inflates counts (e.g. it would list players with zero *visible*/active reports). QA has flagged exactly this — a raw `public.scout_report` ranking returns players (and counts) the user shouldn't see.

How to do it scoped:

1. Call `organizationScoutReports` with **no `search` filter** (page with `first`/`after`) to get the reports the user can see. Each `edges { node }` carries `playerId`, `playerName`, `club`, `overallScore`, `reportTypeName`, `matchDate`, `scoutName`.
2. Group the nodes by `playerId` → the **distinct scouted players**, a **count per player**, and (if the user asked who is "recommended" / "rated highest") an average `overallScore` per player.
3. Rank / take the top-N by per-player count (or average score). Use `playerName`/`club` from the nodes for output — do not re-query `public.scout_report`.
4. If the user adds attribute constraints not present in scout-report data (position/role like "moppers", valuation "< 10M", age, league), first resolve the scouted `playerId` set via steps 1–2, then apply those attribute filters with SQL **restricted to that id set** (`WHERE p.id IN ('<id1>','<id2>', …)`) or via `filterPlayers`. The "we've scouted them" / "our scouts recommend" predicate always comes from `organizationScoutReports`; SQL only supplies the non-scout attributes for those ids. **Never** add `public.scout_report` to a `FROM`/`JOIN`.

A player with zero reports the user can see simply won't appear in `organizationScoutReports` — never re-introduce them from SQL.

## Step 8c: Ambiguous league names — disambiguate before answering

League names are not unique across countries. The same name maps to several distinct leagues:

- **"Championship"** → USL Championship (USA), Scottish Championship, EFL Championship (England), …
- **"Premier League"** → England, Scotland, Russia, …
- **"Serie A"** → Italy and Brazil, …

When the user names a league **without a country** and the name is one of these ambiguous ones, do **not** silently query all of them or pick one. First list the distinct matches:

```sql
SELECT DISTINCT "CURRENT_LEAGUE"
FROM stat.player_stats_pivoted
WHERE "CURRENT_LEAGUE" ILIKE '%Championship%'
ORDER BY "CURRENT_LEAGUE"
```

If more than one distinct league comes back, ask the user which one they mean (list them as numbered options) before running the ranking query. If only one matches, proceed. If the user already specified the country ("Italian Serie A", "English Championship"), use that and don't ask.

## Step 8c-team: Ambiguous team / club names — disambiguate before answering

Club names are **not** unique. The same name maps to several distinct teams across countries and divisions:

- **"Chelsea"** → Chelsea FC (England), and other clubs sharing the name in lower divisions / other countries.
- **"Arsenal"** → Arsenal FC (England), Arsenal de Sarandí (Argentina), Arsenal Tula (Russia), …
- **"Rangers"**, **"Athletic"**, **"Racing"**, etc. — all collide across leagues.

When the user names a club and the team resolver (`public.team` ILIKE, or `listMyOrganizationsEligibleTeams`) returns **more than one row**, do **NOT** silently pick one — that is exactly how "Who are Chelsea's top 5 players?" matched the wrong club. Instead, do one of:

1. **Ask the user to confirm** which club they mean — list the candidates with distinguishing detail (league / country):

   ```sql
   SELECT t.id, t.name, l.name AS league_name
   FROM public.team t
   LEFT JOIN public.league l ON l.id = t.league_id
   WHERE t.name ILIKE '%Chelsea%'
   ORDER BY t.name
   LIMIT 10
   ```

2. **Pick the most likely top-flight club and state the assumption explicitly** in your answer — e.g. "Assuming Chelsea FC in the Premier League…". Only take this path when there's a clear top-flight favourite; otherwise ask.

If exactly one team matches, proceed without asking. If zero match, tell the user you couldn't find that club — never average everyone or substitute a similarly-named team.

## Step 8d: Skill / ability comparisons — use categorical scores, not just scout reports

When the user asks how good a player is at a specific **skill** ("shooting ability", "dribbling", "passing", "aerial ability") or to **compare** players on a skill, the primary source is the numeric categorical-score columns in `stat.player_stats_pivoted`, not scout reports. Scout reports are qualitative colour, available for only a few players, and must not be the sole basis for a skill comparison.

The pivoted view has `*_CATEGORICAL_SCORE` columns per skill family — e.g. shooting (`"SHOT_LOCATION_..."`), dribbling (`"DRIBBLING_..."`), passing (`"PASSING_..."`), carrying (`"CARRYING_..."`), goalkeeping (`"GK_..."`), etc. Bring in scout-report notes only as a supplement when they exist.

**Categorical-score column naming — only two qualifiers exist.** Every categorical score column is `<FAMILY>_[TRANS_]<GLOBAL|POSITIONAL>_CATEGORICAL_SCORE`. The ONLY scope qualifiers are `GLOBAL` and `POSITIONAL` (each optionally prefixed with `TRANS_`). There is **no** `REGIONAL`, `LOCAL`, `NATIONAL`, or other variant — do not invent them. Examples that exist: `"DRIBBLING_TRANS_GLOBAL_CATEGORICAL_SCORE"`, `"PASSING_GLOBAL_CATEGORICAL_SCORE"`, `"PASSING_POSITIONAL_CATEGORICAL_SCORE"`, `"SHOT_LOCATION_GLOBAL_CATEGORICAL_SCORE"`. The full column list is large, so confirm exact names against `getSqlSchema` before querying.

**Contract data IS in the view.** Do not tell the user contract length isn't tracked — `"CONTRACT_EXPIRES"` (and `"CONTRACT_OPTION"`, `"CONTRACT_THERE_EXPIRES"`) are columns in `stat.player_stats_pivoted`. `"TEAM_STYLE_FIT"` is the team-style fit score column.

**Recover from a "column does not exist" error — never give up.** If `executeSqlQuery` returns `column ... does not exist`, you guessed a name wrong. Re-check `getSqlSchema` for the correct column (right family, `GLOBAL`/`POSITIONAL` qualifier, exact casing, double-quoted) and re-run. Do not abandon part of the answer or claim the data isn't available — the error means the column name was wrong, not that the data is missing.

## Step 8e: Goalscorer leaderboards — use listTopScorers, not SQL

Goals scored are **not** in `stat.player_stats_pivoted` (it has GPR, Fit Score, valuation, position — but no goals/assists). So do **not** try to rank goalscorers with SQL; you will either fail or return the wrong metric.

For prompts asking who scored the most goals — "top 5 goalscorers in the Premier League this season", "leading scorers in Serie A", "who has the most goals" — use the dedicated **`listTopScorers`** tool:

1. Resolve the league to its id with `listMyOrganizationsLeagues`. If the league name is ambiguous across countries (see Step 8c), ask the user which one first.
2. Call `listTopScorers` with `leagueId` (required), optional `seasonId` (omit for the most recent season with data), and `first` (default 20; use the number the user asked for, e.g. 5).
3. Present the returned players with their goal totals. If it returns an empty list, tell the user there's no goal data for that league/season — do **not** fall back to SQL or substitute a different metric (e.g. GPR).

This is the only correct path for goal counts. `listSeasonProviderMetrics` is single-player and cannot produce a leaderboard.

## Step 9: Common Pitfalls

1. **Do not reference columns that do not exist.** There are no `PLAYER_NAME`, `GOALS`, `ASSISTS`, or `RATING` columns. Check the schema first.
2. **Do not forget double quotes on materialized view columns.** `SELECT FIT_SCORE` will fail. Use `SELECT "FIT_SCORE"`.
3. **Do not omit the schema prefix.** `FROM player` will fail. Use `FROM public.player`.
4. **Do not skip the JOIN for names.** The stats view only has `"PLAYER_ID"`, not names.
5. **Do not use uncast UUID literals.** Use `'value'::uuid` for UUID comparisons.
6. **Do not rename tables.** The scouting table is `public.scout_report`, NOT `public.scouting_report`. Copy table names exactly from the schema.
7. **Never answer a per-player scout-report question with SQL — use `organizationScoutReports`.** Counting or listing one player's scout reports over `public.scout_report` over-reports what the user can see (the table is org-scoped, not visibility-scoped); use `organizationScoutReports(filter:{search:<player name>})` and read its `totalCount` (and `edges` to list) instead (see Step 8b). SQL over `public.scout_report` is only for org-wide scout-activity aggregates, and even then must join by id (never `data->>'playerName'`) and filter `WHERE sr.archived_at IS NULL AND sr.parent_report_id IS NULL AND sr.processed_data IS NOT NULL` (plus the org scope), or a raw `COUNT(*)` over-counts archived, child/duplicate, and unprocessed rows.
8. **Do not silently resolve ambiguous leagues.** "Championship", "Premier League", and "Serie A" map to multiple leagues across countries — disambiguate per Step 8c before answering.
9. **`SELECT DISTINCT` + `ORDER BY` must agree.** Postgres requires every `ORDER BY` expression to also appear in the `SELECT` list when `DISTINCT` is used (otherwise: "for SELECT DISTINCT, ORDER BY expressions must appear in select list"). Either add the ordering column to the `SELECT`, drop `DISTINCT`, or use `GROUP BY` — don't emit a `SELECT DISTINCT ... ORDER BY <unselected column>` query.

## Step 10: Execute Query

Call `executeSqlQuery` with your constructed query:

```json
{
  "query": "SELECT p.first_name, p.last_name, s.\"TIME_DECAYED_GPR\" FROM stat.player_stats_pivoted s JOIN public.player p ON p.id = s.\"PLAYER_ID\" ORDER BY s.\"TIME_DECAYED_GPR\" DESC LIMIT 10"
}
```

Constraints: 30-second timeout, maximum 10,000 rows returned. Add WHERE clauses and LIMIT to keep queries fast.

## Step 11: Response Formatting

- Never mention database table names, column names, SQL queries, joins, or any data retrieval methods in your answer
- Use human-friendly names for all metrics: say "GPR" not "TIME_DECAYED_GPR", "Fit Score" not "FIT_SCORE", "Valuation" not "PLAYER_VALUATION"
- Fit Score from player_stats_pivoted is stored as a 0–1 decimal — always render it as an integer 0–100 (multiply by 100, round), e.g. 0.63 → 63. (The player_team_fit score is already 0–100; do not multiply that one.)
- GPR stands for "Gemini Player Rating" -- never say "General Performance Rating" or "Gemini Performance Rating"
- Focus on insights and results, not how data was retrieved

## Step 12: Troubleshooting

- If a query returns 0 rows for a player filtering question, try the `filterPlayers` tool instead -- it supports roleArchetypes, valuation, position, and other structured filters that may match when SQL does not
- If 0 rows are expected to be a data issue, broaden your SQL filters (remove constraints one at a time) rather than repeating the same query
- If you've retrieved the schema already, do not call getSqlSchema again -- write and execute a query
- If you've been reasoning for 2+ steps without calling a tool, either execute a query or provide a final answer with what you know
- If a tool returns an error, try a different approach: simpler parameters, a different tool, or a modified query

## Step 13: Present Results

After receiving query results:

1. Format the data as a readable table or list.
2. Highlight key insights (top performers, outliers, trends).
3. Offer follow-up queries the user might find useful (e.g., "Want to see this filtered by league?" or "Should I compare these players in detail?").
