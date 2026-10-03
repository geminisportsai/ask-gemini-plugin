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

You must call this tool before writing any SQL query to confirm available tables and column names — except in Steps 8b, 8b-rank and 8b-region, which never call `getSqlSchema`, not as the first call and not after a column error: its sample rows carry raw scout-report data outside the user's permissions, and those steps list every column they need.

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

## Step 2a: Identify a specific named player by ID — never match a player by name in SQL

When a prompt names a **specific player** (e.g. "compare Alexander Isak and Darwin Nunez", "what is Saka's fit score", "Pedri's contract"), do **not** identify them by matching `first_name` / `last_name` in SQL — `ILIKE`, `IN (...)`, and `=` are all **accent-sensitive**, so a query for "Darwin Nunez" silently misses the player stored as "Darwin **Núñez**". QA flagged exactly this regression.

Resolve each named player to their **id** first, then filter the stats query by id:

1. Call **`searchPlayers`** with the name exactly as the user gave it (e.g. `"Darwin Nunez"`). It folds diacritics **server-side** (via `f_unaccent`), so unaccented input matches accented records every time.

   **Do not take the top result on trust — result order is not a resolver.** `searchPlayers` returns the first lexicographic match for a name fragment, which is often not the player the user means; a surname query like "Haaland" can match a lower-tier player ahead of Erling Haaland. Check the candidate's `currentClub` or `currentLeague` against the player they plainly mean, resolving the club with `getPlayer` if `searchPlayers` omits it. If more than one plausible player remains, **ask which one they mean** and list the candidates with distinguishing detail. Only use an `id` once exactly one player is identified. A query run against the wrong player returns confident, wrong numbers, which is worse than a clarifying question.
2. Filter your query by that id — **never** by name:
   ```sql
   SELECT p.first_name, p.last_name, s."TEAM_STYLE_FIT", s."CONTRACT_EXPIRES"
   FROM stat.player_stats_pivoted s
   JOIN public.player p ON p.id = s."PLAYER_ID"
   WHERE s."PLAYER_ID" IN ('<isak-uuid>'::uuid, '<nunez-uuid>'::uuid)
   ```
3. If `searchPlayers` returns no match for a name, tell the user that player wasn't found — do **not** fall back to an `ILIKE` / `IN` name query (it will keep missing accented names).

This applies only to **player identification**. `ILIKE` on **team** and **league** names (Steps 8b–8c) is fine — those resolvers/queries already handle their own disambiguation.

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
| `"LEAGUE_COUNTRY"` | Country of the player's **current** league (from `public.league.country`). Spellings are inconsistent (`England`/`england`, `Korea, South`/`Korea  South`, `Türkiye`/`Turkiye`). Never use it for a scout-report region question — every report already carries its `region` (Step 8b-region). |
| `"PLAYER_VALUATION"` | Market valuation (use for "value" queries) |
| `"NATIONALITY"` | Player nationality |
| `"MINUTES_TOTAL"` | Total minutes played |

**Columns that do NOT exist:** `PLAYER_NAME`, `GOALS`, `ASSISTS`, `RATING`. Do not use these. This is about SQL columns; it does not apply to the `PLAYER_NAME` sort of `organizationScoutReports`.

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
6. **Use ILIKE for text matching of TEAM / LEAGUE names only**: `WHERE t.name ILIKE '%Arsenal%'`. For a **specific named player**, do NOT match `first_name`/`last_name` with `ILIKE`/`IN`/`=` — they're accent-sensitive and miss diacritics (e.g. "Nunez" ≠ "Núñez"). Resolve the player via `searchPlayers` → id and filter on `s."PLAYER_ID"` (Step 2a).
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

Resolve each named player to their id via `searchPlayers` first (Step 2a), then filter by id — **not** by `last_name` (accent-sensitive, misses names like "Núñez"):

```sql
SELECT p.first_name, p.last_name, s."FIT_SCORE", s."TIME_DECAYED_GPR",
       s."AGE", s."CURRENT_CLUB", s."GENERAL_POSITION"
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
WHERE s."PLAYER_ID" IN ('<salah-uuid>'::uuid, '<saka-uuid>'::uuid, '<foden-uuid>'::uuid)
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

Never use SQL over `public.scout_report` to answer how many reports a single player has — that is the per-player case above and must go through `organizationScoutReports`. The fallback also never applies to a question asked **by region** ("reports from South America", "players in Scandinavia") — that always goes through Step 8b-region.

**Scout-report honesty — only claim a report exists when `organizationScoutReports` actually returns one.** Only tell the user a player has a scouting report (or quote a "scout score") when `organizationScoutReports(filter:{search:<name>}).totalCount` is one or more for that player. If it is zero, say the player has **no** scout report — do not soften it, do not infer one. In particular, **never infer the existence of a scout report from the presence of a GPR, GPM, fit score, or any other metric.** A GPR is computed for almost every tracked player and says nothing about whether a human scout report exists. The two are unrelated data sources — having a GPR does not mean a scout report was written, and a player with a strong GPR routinely has zero scout reports. Report exactly what `organizationScoutReports` returns.

## Step 8b-rank: Ranking or FILTERING players by scout reports — permission-scoped sources, never a `public.scout_report` join

Some questions rank or filter players by whether **we** have scouted them, how many reports they have, or how our scouts rate them:

- "Who are the top 5 players we have scout reports on?"
- "Which players have our scouts written the most reports about?"
- "Show me 5 moppers under 10M that our scouts recommend / that we've scouted."

The set of scouted players MUST come from a permission-scoped source — `filter.scoutReport` on `filterPlayers` (below) or the `organizationScoutReports` query — **never** from a SQL join against `public.scout_report`. That table is org-scoped, ignores per-user visibility, and over-counts: it surfaces players whose reports the user can't actually see and inflates counts (e.g. it would list players with zero *visible*/active reports). QA has flagged exactly this — a raw `public.scout_report` ranking returns players (and counts) the user shouldn't see.

**What "top" means.** "Top players we have scout reports on" asks for the best players among those we have scouted — rank them by GPR (`"TIME_DECAYED_GPR"` DESC), not by how many reports they have. Rank by report count only when the user asks about the number of reports ("most reports", "most scouted"). You may show each player's report count next to the GPR.

**No `getSqlSchema` anywhere in this step** (see Step 1), on either path.

**Criteria ledger — before you answer, on either path.** Write every criterion phrase in the question as a list in your reasoning, in the user's words — each position, price, age, league, skill, role, scout phrase and any other condition — and mark each one `applied (<the field or tool that applied it>)` or `not applied`. A phrase counts as applied only when a call that succeeded this turn applied it. Every phrase marked not applied goes into the answer's Not applied list; leave none out.

Worked example with `filter.scoutReport` — "Show me 5 moppers who cost less than 10M that are recommended by our scouts":

- "moppers" → not applied: mopper (scouts' positional profile)
- "cost less than 10M" → applied (`maxValuation: 10000000`)
- "recommended by our scouts" → applied (`scoutReport: { wouldSignPlayer: true }`)

So the answer lists the players under 10M our scouts would sign, and says: "Not applied: mopper (scouts' positional profile) — I can't filter on that yet."

Worked example on the fallback — "Show me 5 moppers who cost less than 10M that are recommended by our scouts":

- "moppers" → not applied: mopper (scouts' positional profile)
- "cost less than 10M" → applied (`s."PLAYER_VALUATION" < 10000000`)
- "recommended by our scouts" → not applied (no report read this turn carries a recommendation)

So the answer ends: "Not applied: mopper (scouts' positional profile), recommended by our scouts — that isn't available from our scout reports yet." followed by the Coverage sentence.

### Scout criteria with `filter.scoutReport`

`filterPlayers` filters players by our scouts' verdicts itself, through `filter.scoutReport`. It counts only the reports this user may see, and all criteria hold on the same report. Use it only when the `filterPlayers` tool description mentions `filter.scoutReport`. A backend without it rejects or ignores the argument — ignored, it returns players our scouts never rated — so if the description does not mention it, never send it; follow the fallback below instead. If a call with `scoutReport` errors, never retry without it; use the fallback below.

When it is available, make one `filterPlayers` call with `filter.scoutReport` and every other criterion the user named in the same `filter` — no `organizationScoutReports` reading, no copied ids, no SQL:

| The user asks for players… | `filter.scoutReport` |
|---|---|
| we have scout reports on / we've scouted | `{ hasReport: true }` |
| our scouts rated a first-11 (Starting XI) player | `{ startingXI: true }` |
| our scouts rated an investment | `{ investmentPlayer: true }` |
| our scouts rated a first-11 investment | `{ startingXI: true, investmentPlayer: true }` |
| our scouts recommend / would sign | `{ wouldSignPlayer: true }` |

Do not send `positionalProfiles` yet: older reports have no positional profile filled in, so the filter would miss players our scouts did profile. A role name tied to a scout verdict ("moppers our scouts recommend", "stoppers our scouts rated a first-11 player") is the scouts' positional profile, which can't be filtered yet: name it not applied — "mopper (scouts' positional profile)" — never put it in `roleArchetypes`, and still apply the other criteria, scout verdicts included, through `filter.scoutReport`. A role tied only to having a report ("moppers we've scouted") is the statistical archetype: send `roleArchetypes` with the archetype name `listRoleArchetypes` returns (e.g. `["MOPPER"]`) and `scoutReport: { hasReport: true }`, and label it in the answer as "the statistical <ARCHETYPE> role archetype, not a scout's view".

- `hasReport: false` cannot be combined with a verdict. `hasReport: false` alone is rejected, so pair it with at least one other criterion the user named: "centre-backs we have not scouted" is `{ positionIds: [...], scoutReport: { hasReport: false } }`. If the user named nothing else, ask which position or league to search.
- Only some report forms record verdicts and positional profiles. You may say that, but never name which organizations', clubs' or report forms record a verdict or profile — not even when a tool description names them.
- Resolve the other criteria first. Positions → `listPositions` (`positionIds`). Page `listPositions` before using any position id: call it with `first: 100`; while `pageInfo.hasNextPage` is true, call again with `after: pageInfo.endCursor`. Never rank or filter on a partial position list — a position missing from the first page still exists. Do not pass `isGeneral`: it returns only general positions, without the abbreviations the role groups use. A league → call `listMyOrganizationsLeagues` with `first: 100`; while `pageInfo.hasNextPage` is true, call again with `after: pageInfo.endCursor` — a league not on the first page is not out of scope (`leagueIds`). Valuation in full units (`maxValuation: 10000000` for 10M). Ages with `minAge` / `maxAge`. Footedness → `feet`: left-footed → `feet: [LEFT, BOTH]`, right-footed → `feet: [RIGHT, BOTH]` (a two-footed player can play either side).
- Sort: "top" / "best" → `sortBy: "GPR", sortOrder: "DESC"`. A skill the user ranks by → its `SortField` (carrying → `CARRYING`). `first` = the number the user asked for (10 if none).

Worked example — "left wingers under 10M our scouts rated as a first-11 investment, best carriers first":

```
filterPlayers(filter: { positionIds: ["<LW id>"], maxValuation: 10000000, scoutReport: { startingXI: true, investmentPlayer: true } }, sortBy: "CARRYING", sortOrder: "DESC", first: 10)
```

Answering:

- Only describe a player as scouted, rated or recommended by our scouts for a criterion that was in a `filter.scoutReport` call that succeeded. A scout judgement with no `scoutReport` field (e.g. "good attitude") is named as not applied — "I can't filter on <criterion> yet, so I didn't apply it." — and never stood in for by GPR, `overallScore` or a role archetype.
- Every phrase your criteria ledger marks not applied is named in the answer, in the user's words — leave none out.
- `totalCount` is a server count of the matching players, so you may state it ("12 left wingers match; here are the top 10").
- Zero results is the answer: none of the players our scouts rated that way match the other criteria. Never answer with a bare "none" — add that only some report forms record verdicts and positional profiles, so players reported on other forms can't appear. Never rerun without `scoutReport` to fill the list; you may offer an unscouted search the user can ask for.
- `filterPlayers` cannot count reports — a question about the number of reports ("most reports", "most scouted") uses the fallback below, which reads and counts the reports.

### Fallback — only when the `filterPlayers` description does not mention `filter.scoutReport`, a `scoutReport` call errored, or the question is about the number of reports

On the fallback, a role name tied to a scout verdict is never the statistical archetype: name it not applied — "mopper (scouts' positional profile)" — never put it in `roleArchetypes` and never call `listRoleArchetypes` for it. If you offer the statistical archetype as a follow-up, label it in the answer as "the statistical <ARCHETYPE> role archetype, not a scout's view". In SQL, never filter `s."ROLE_ARCHETYPE"` for a role tied to a scout verdict either.

On the fallback, a role tied only to having a report ("moppers we've scouted") reads the primary archetype: filter on `s."ROLE_ARCHETYPE"` and say "players whose primary statistical role archetype is <ARCHETYPE> (not a scout's view)" — never present it as every <role> we have scouted.

How to do it scoped — one retrieval, ranked within what it returned:

1. Make exactly one `organizationScoutReports` call: **no `search` filter** and `first: 50` (a 100-report page is larger than the tool-result size limit and gets cut, losing reports). Each `edges { node }` carries `playerId`, `playerName`, `club`, `overallScore`, `reportTypeName`, `matchDate`, `scoutName`, and — when the report type exposes them — `numericRatings` / `categoricalRatings` (`key`, `label`, `value`). Note `totalCount`. If the result carries a `TRUNCATED` note, only the edges shown count as read. Do not page further to widen coverage — even when no player in it meets the criteria; answer from this one page and say so. A full ranking across every scouted player needs `filter.scoutReport`, which this backend does not have.
2. Group the nodes by `playerId` → the **distinct scouted players** in what you read and a **count per player**. Before writing any SQL, write the distinct `playerId`s as a numbered list in your reasoning, each with its report count. Check it: the per-player report counts must add up to the number of edges you read. If they don't, you missed or merged a player, so redo the list. The last number in the list is the distinct-player count, and you copy the ids into the SQL from that numbered list. Use `playerName`/`club` from the nodes for output — do not re-query `public.scout_report`.
3. Read GPR — and any non-scout attribute the user named (position, valuation "< 10M", age, league, a skill score such as carrying — column families in Step 8d) — with one SQL query over `stat.player_stats_pivoted` **restricted to those player ids**. The query counts the ids it was sent, so you can check none were dropped while copying them:

   ```sql
   WITH ids AS (SELECT unnest(ARRAY['<id1>', '<id2>', …]::uuid[]) AS id)
   SELECT (SELECT count(DISTINCT id) FROM ids) AS ids_sent,
          count(*) FILTER (WHERE s."TIME_DECAYED_GPR" IS NULL) OVER () AS no_gpr,
          i.id AS player_id, p.first_name, p.last_name,
          s."TIME_DECAYED_GPR", s."CURRENT_CLUB", s."GENERAL_POSITION"
   FROM ids i
   LEFT JOIN public.player p ON p.id = i.id
   LEFT JOIN stat.player_stats_pivoted s ON s."PLAYER_ID" = i.id
   ORDER BY s."TIME_DECAYED_GPR" DESC NULLS LAST
   ```

   Add the columns and `WHERE` conditions for any attribute the user named. Do not add a `LIMIT` to this query: it returns at most one row per id, and you need every row. Take the top N yourself in step 4.

   **`no_gpr` check — a hard rule.** Every row carries `no_gpr`, the number of returned players with no GPR. If `no_gpr` is greater than 0, the answer must contain "Some reported players have no GPR yet and aren't ranked." — whatever else you leave out.

   **`ids_sent` check — a hard stop.** Before using any row, compare `ids_sent` with the distinct-player count from step 2 (the last number in your numbered list) — write both numbers in your reasoning as `ids_sent = <x>, list = <y>`. If they differ, you MUST NOT answer yet:
   1. Compare the ids in the query you ran with your numbered list, and find every id that is missing.
   2. Re-run the same query with every id from the numbered list — the full list, not only the missing ids.
   3. Check `ids_sent` again. Answer only once `ids_sent` equals the step-2 count.

   **Zero rows is the answer.** If no row meets the user's criteria, the answer is that none of the scouted players in the reports I could read match — say exactly that, and never say the players lack position, valuation or score data. Keep the scouted-id restriction on every query: never run a second query without the restriction, never drop it to get more rows, and never list a player who is not in the reports you read. You may offer an unscouted search as something the user can ask for — never run it and never list its players. If fewer scouted players match than the user asked for, answer with those.

   A player with no stats row (or a NULL GPR) still comes back from the `LEFT JOIN` — treat them as "no GPR yet": they are not ranked, and the final paragraph says so (step 5). **Never** add `public.scout_report` to a `FROM`/`JOIN`.

   **Do not call `getSqlSchema` in this procedure** — not even after a column error. Its sample rows contain raw scout-report data outside the user's permissions, and nothing in it is a fact about these players. Use only these columns: from `public.player p` — `p.id`, `p.first_name`, `p.last_name`; from `stat.player_stats_pivoted s` — `s."PLAYER_ID"`, `s."TIME_DECAYED_GPR"` (GPR), `s."AGE"`, `s."GENERAL_POSITION"`, `s."PRIMARY_POSITION"` (e.g. left winger), `s."CURRENT_CLUB"`, `s."CURRENT_LEAGUE"`, `s."PLAYER_VALUATION"` (full units, e.g. 10000000 for 10M), `s."NATIONALITY"`, `s."FOOT"` (left-footed → `lower(s."FOOT") IN ('left', 'both')`, right-footed → `lower(s."FOOT") IN ('right', 'both')`), `s."ROLE_ARCHETYPE"` (the player's PRIMARY statistical role archetype only, uppercase, e.g. `'MOPPER'`) — only for a role tied to having a report, never for a scout verdict — and the skill scores `s."<FAMILY>_[TRANS_]GLOBAL_CATEGORICAL_SCORE"` from Step 8d (carrying is `s."CARRYING_TRANS_GLOBAL_CATEGORICAL_SCORE"`). Never guess a column: a criterion with no column listed here (a release clause, say) is not applied — name it under Not applied without running a query for it.
4. Rank those players by GPR (or by the metric the user asked for) and take the top N. This is a ranking within the reports you read: never present it as a ranking of every scouted player. Say it in the answer's first sentence: "Among the players in the first <N> of your <T> scout reports, the top <K> by <metric> are …" — the Coverage line alone is not enough. Never write "among those your scouts have reported on", "of all our scouted players" or any wording that reads as complete unless <N> equals <T>.
5. **End the answer with this final paragraph, in exactly this shape:**

   > Not applied: <each criterion you did not apply, in the user's words> — that isn't available from our scout reports yet. Coverage: the <N> scout reports I could read (you have <T>).

   The Coverage sentence is the last line of the answer. The whole answer is laid out like this:

   ```text
   <answer, then any offer — "If you'd like, I can …">

   Not applied: … — that isn't available from our scout reports yet. Coverage: the <N> scout reports I could read (you have <T>).
   ```

   - Put any offer or follow-up before this paragraph — nothing comes after it.
   - List every criterion the user named that you did not apply, each in the user's own words — e.g. "rated as a worthwhile first-11 investment by our scouts", "recommended by our scouts", a scout positional profile such as "mopper", or an attribute with no column above. Leave out the "Not applied:" sentence only when every criterion was applied.
   - <N> is the number of edges you actually read and <T> is `totalCount` — never a fixed page size, and never `totalCount` as the number you read.
   - Show a valuation in the currency the tool returned; if it returned none, show the number with no currency symbol ("valued at 1.5M").
   - If any reported players had no GPR, put this sentence just before "Coverage:": "Some reported players have no GPR yet and aren't ranked."
   - Never state a count of players anywhere in the answer, even when you read every report — not "covering 23 distinct players", not "the 8 scouted players under that price", not "these three players", not how many players the reports cover, not how many matched, not how many lack a GPR. Player counts come from ids copied by hand and can be wrong. (Listing the top N the user asked for is fine.)
   - Forbidden wording: never say our scout reports "don't contain", "don't include", "don't record" or "don't have" something, and never describe what our scout reports contain instead. The data exists; it just isn't available here yet — say "isn't available from our scout reports yet" and nothing more about it.

**Before you answer, check your draft:**

- No number, in digits or in words in any language ("3", "three", "três"), is followed by "players" anywhere, except the top N the user asked for.
- The last line starts with "Not applied:" or "Coverage:", and no offer comes after it.
- Every Not applied item ends with exactly "— that isn't available from our scout reports yet", with no other reason.
- Every criterion the user named is either applied or listed under Not applied.
- Every phrase your criteria ledger marks not applied is in the Not applied list.
- The first sentence says "the first <N> of your <T> scout reports" unless <N> equals <T>.
- If `no_gpr` is greater than 0, the final paragraph contains "Some reported players have no GPR yet and aren't ranked."

If any check fails, fix the draft before you answer.

**Scout judgements the reports may not carry.** Some questions ask for a specific scout verdict — "rated as a worthwhile first-11 investment", "recommended", or a scout positional profile such as "mopper", "stopper". Apply it only when the report data you retrieved in this conversation actually contains it (a `categoricalRatings` / `numericRatings` entry with that meaning). If it does not, do not guess and do not replace it with GPR, `overallScore`, or a role archetype: answer with the scouted players that meet the other criteria — the judgement isn't available from our scout reports yet — list it under Not applied (step 5). `overallScore` may be shown, labelled as the report's overall score — never called a recommendation.

**"Moppers" and other role names "recommended by our scouts".** A stats-derived role archetype (`MOPPER` and the others from `listRoleArchetypes`) exists in player data, but it is a statistical profile, not a scout's view. When the user asks for a role that our scouts recommend, list the scout positional profile under Not applied; you may offer the statistical role-archetype filter as a follow-up the user can ask for — e.g. "Show me moppers under 10M" — clearly labelled as statistical rather than scout-based. Never say the filter is unavailable.

A player with zero reports the user can see simply won't appear in `organizationScoutReports` — never re-introduce them from SQL.

## Step 8b-region: Scout reports by REGION — `organizationScoutReports` with `filter.regions` and `regionCounts`

Some questions ask about scout reports by **region** rather than by player:

- "Show me scout reports from South America."
- "How many reports did we file on players in Scandinavia?"
- "Which players have we scouted in Scandinavia?"
- "Which region have our scouts covered most?"

**Region or nationality?** "Reports from / in <region>", "players in <region>" and "scouted in <region>" mean the **league region** — this step. A nationality or demonym ("Brazilian players", "South American players", "players born in Norway") is not something this step can answer: the region filter is the league the player was playing in when the report was written, not their nationality. Say so, and offer the league-region answer instead. If a question could be read either way, use the league region and say that is the interpretation you used.

Every report `organizationScoutReports` returns carries a `region`: the **FM24 scouting region of the league the player was playing in when the report was written** — the league of the report's match, or the player's current league when the report has no resolvable match — set by the backend (null only when neither gives a country inside the FM24 table). The query filters by it (`filter.regions`, several names combined with OR) and counts by it (`regionCounts`: `{ region, count }` per region over the same filters and visibility, ignoring pagination; the counts add up to `totalCount`). It is permission-scoped to what this user may see. There is no SQL in this step.

### The region names

Read the status instruction at the top of this block first and follow it. Map the user's words onto the region names in this table.

<!-- scouting-regions:start -->
<!-- Generated from lib/skills/scoutingRegions.ts by `bun run skills:render-regions`. Do not edit by hand. -->

> Source: GD-217 / Notion "Regions for Each Country", Football Manager 2024 (FM24) scouting regions. These are the FM24 scouting regions: a name that is neither a region in this table nor a stem listed below is not one of the FM24 scouting regions — say so and list the regions. Pass region names to `filter.regions` exactly as written in this table.

> The only multi-region names are: South America (= South America (North) + South America (South)). Each is not a region on its own but means all of those regions: answer for all of them together and name the FM24 regions used. Any other area name, including continents such as Africa, Asia or Europe, is not an FM24 scouting region — say so and list the regions.

| Region | Countries (league country when the report was written) |
|--------|--------------------------------------------------------|
| Central Africa | Cameroon; Central African Republic; Chad; Congo; DR Congo; Equatorial Guinea; Gabon; São Tomé & Príncipe |
| East Africa | Burundi; Djibouti; Eritrea; Ethiopia; Kenya; Mayotte; Réunion; Rwanda; Somalia; South Sudan; Tanzania; Uganda; Zanzibar |
| North Africa | Algeria; Egypt; Libya; Morocco; Sudan; Tunisia |
| Southern Africa | Angola; Botswana; Comoros; Eswatini; Lesotho; Madagascar; Malawi; Mauritius; Mozambique; Namibia; Seychelles; South Africa; Zambia; Zimbabwe |
| Western Africa | Benin; Burkina Faso; Cape Verde; Côte d'Ivoire; Gambia; Ghana; Guinea; Guinea-Bissau; Liberia; Mali; Mauritania; Niger; Nigeria; Senegal; Sierra Leone; Togo |
| Central Asia | Kazakhstan; Kyrgyzstan; Tajikistan; Turkmenistan; Uzbekistan |
| East Asia | China; Chinese Taipei; Guam; Hong Kong; Japan; Macau; Mongolia; North Korea; Northern Marianas; South Korea |
| Middle East | Bahrain; Iran; Iraq; Israel; Jordan; Kuwait; Lebanon; Oman; Palestine; Qatar; Saudi Arabia; Syria; United Arab Emirates; Yemen |
| South Asia | Afghanistan; Bangladesh; Bhutan; India; Maldives; Nepal; Pakistan; Sri Lanka |
| Southeast Asia | Brunei; Cambodia; Indonesia; Laos; Malaysia; Myanmar; Philippines; Singapore; Thailand; Timor-Leste; Vietnam |
| Central Europe | Austria; Belgium; Czech Republic; Germany; Liechtenstein; Luxembourg; Netherlands; Poland; Slovakia; Switzerland |
| Eastern Europe | Bulgaria; Hungary; Moldova; Romania; Serbia |
| North Eastern Europe | Belarus; Estonia; Latvia; Lithuania; Russia; Ukraine |
| Northern Europe | Denmark; Faroe Islands; Finland; Iceland; Norway; Sweden |
| South Eastern Europe | Armenia; Azerbaijan; Cyprus; Georgia; Greece; North Macedonia; Turkey |
| South Europe | Albania; Bosnia and Herzegovina; Croatia; Italy; Kosovo; Malta; Montenegro; San Marino; Slovenia |
| UK & Ireland | England; Ireland; Northern Ireland; Scotland; Wales |
| Western Europe | Andorra; France; Gibraltar; Portugal; Spain |
| Caribbean | Anguilla; Antigua and Barbuda; Aruba; Bahamas; Bermuda; Bonaire; British Virgin Islands; Cayman Islands; Cuba; Curaçao; Dominica; Dominican Republic; Grenada; Guadeloupe; Haiti; Jamaica; Martinique; Montserrat; Puerto Rico; Saint Barthélemy; Saint Kitts and Nevis; Saint Lucia; Saint-Martin; Sint Maarten; St. Vincent & the Grenadines; Trinidad & Tobago; Turks & Caicos Islands; US Virgin Islands |
| Central America | Belize; Costa Rica; El Salvador; Guatemala; Honduras; Nicaragua; Panama |
| North America | Canada; Mexico; St. Pierre & Miquelon; United States |
| Oceania | American Samoa; Australia; Cook Islands; Fiji; Kiribati; Micronesia; New Caledonia; New Zealand; Papua New Guinea; Samoa; Solomon Islands; Tahiti; Tonga; Tuvalu; Vanuatu; Wallis & Futuna Islands |
| South America (North) | Bolivia; Colombia; Ecuador; French Guiana; Guyana; Peru; Suriname; Venezuela |
| South America (South) | Argentina; Brazil; Chile; Paraguay; Uruguay |
<!-- scouting-regions:end -->

### Procedure

1. **Map the region.** Turn the user's phrase into region names from the table above: an exact region, or both halves of a multi-region name — for "South America" pass `regions: ["South America (North)", "South America (South)"]`. If it maps to no region (see the rules below), do not call any tool for it.
2. **Count — one call.** For "how many reports …" make ONE `organizationScoutReports` call with `filter: { regions: [...] }` and `first: 1`. Its `totalCount` is the exact number of matching reports across all pages; for a multi-region name also give each region's count from `regionCounts`. For "which region have our scouts covered most?" make ONE call with no `regions` filter and `first: 1`, and rank `regionCounts` (a null `region` is "no region": players with no league or a country outside the FM24 table). Add any other filter the user gave (dates, clubs, scouts, `search`) to the same call. Read the number from `totalCount` / `regionCounts` — never page through edges to count reports, and never add up nodes to get a report count.
3. **List.** For "show me reports from …" / "which players have we scouted in …" call `organizationScoutReports` with `filter: { regions: [...] }`, `first: 50`. Fetch further pages with `after: pageInfo.endCursor` only while the user asked for all of them and `pageInfo.hasNextPage` is true — a players question ("which players", "how many players") counts as asking for all of them: keep paging to the cap — at most 4 pages (200 reports) in one turn. Whenever fewer reports are shown than `totalCount` — past the cap, after a truncated page, or when the user did not ask for all of them — say "showing [N] of [totalCount]" ([N] is the number of reports actually shown, not 50 per page) and that there are more. Offer to continue only when there is a way to: from `truncation.resumeAfterCursor` when the last page was truncated and has one, otherwise from `pageInfo.endCursor` when `pageInfo.hasNextPage` is true. If every report is already shown ([N] equals `totalCount`), list them all and do not say there are more or offer to continue. If `totalCount` is 0, say this user has no scout reports on players in that region.
   - **Reports** ("show me reports"): present each node — `playerName`, `club`, `region`, `overallScore`, `reportTypeName`, `matchDate`, `scoutName` — and say how many reports there are from `totalCount`. A reports listing gives no per-player report count either: never write "(3 reports)", "[k] reports" or a count in a heading next to a player's name — tallying repeated reports is not reliable enough to state. If you group the listed reports by player, head each group with the player's name and club only. The only report count you state is `totalCount` (or "showing [N] of [totalCount]").
   - **Players** ("which players have we scouted", "how many players"): `totalCount` counts reports, not players, and a player can have several reports. For these questions add `sort: { field: PLAYER_NAME, direction: ASC }` so each player's reports come back next to each other. Then group the nodes by `playerId` and list each player once by name (with their club), never once per report. Do not give a per-player report count such as "(3 reports)" — tallying repeated reports is not reliable enough to state. That holds anywhere in the answer, including a summary: never say how many reports one player has or who was scouted most. If asked who was scouted most, say per-player report counts are not available and give the list of players. The player count covers only the pages you fetched. Use "[N] reports covering [M] players" only when every page was fetched (`hasNextPage` false on the last page) and no page carried a `truncation` note. If a page came back with a `truncation` note, some of its edges were dropped to fit the result size limit: continue with `after: truncation.resumeAfterCursor` (this counts as a page toward the cap) instead of `pageInfo.endCursor`, so the dropped edges are read; if the `truncation` note has no `resumeAfterCursor`, use the at-least sentence. Otherwise give "at least [M] players …" whenever pages remain — after the cap, or because a page could not be fetched or the turn ran out of tool calls: say "at least [M] players in the first [N] of [totalCount] reports" and offer to continue. For a players answer use that sentence instead of "showing [N] of [totalCount]". Never present a partial player count as the total, and never give `totalCount` as a number of players.
4. **A rejected name.** If the query rejects a region name with an error listing the valid names, retry once with the matching name from that list; if none matches, apply the unknown-region rule below.

### Rules

- Never use SQL, `public.scout_report` or `stat.player_stats_pivoted` for a region question. The report table is org-scoped, not scoped to what this user may see, so it over-counts; the org-wide last-resort SQL fallback in Step 8b does **not** apply to region questions. Never call `getSqlSchema` for a region question — its sample rows are other organizations' data. The reports, their regions and their counts all come from `organizationScoutReports`, and you never copy player or report ids between calls.
- Every region answer must say that region reflects the league each player was playing in **when the report was written** (the league of the report's match), not their current club — a player who has since transferred stays under the region they were scouted in; a report with no resolvable match (none linked, or none whose league has a country in the FM24 table) uses the player's current league instead.
- While the status instruction says region data is not available yet, it overrides everything in this step: do none of the above and do not list any region names.
- Otherwise, if the user names one of the multi-region names listed above the table (e.g. "South America" → South America (North) + South America (South)), answer for the union of those regions and name the FM24 regions used.
- If the name matches no FM24 region and no listed multi-region name (e.g. "Scandinavia", or a continent such as "Africa"), say it is not one of the FM24 scouting regions and list all 24 region names from the table. After that list you may add which region contains the countries they meant, but ask before answering for it. Never guess which countries a region contains, and never substitute a similar region.

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

When the user asks how good a player is at a specific **skill** ("shooting ability", "dribbling", "passing", "aerial ability") or to **compare** players on a skill, the primary source is the numeric categorical scores, not scout reports. Scout reports are qualitative colour, available for only a few players, and must not be the sole basis for a skill comparison.

**Read the categorical scores from `stat.player_stats_pivoted` with `executeSqlQuery`.** Resolve each player's id with `searchPlayers` first (accent-safe — see Step 2a), then `SELECT` the relevant `*_CATEGORICAL_SCORE` columns for those ids. These are the numeric skill ratings (roughly 0–110; values above 100 are normal). Bring in scout-report notes only as a supplement when they exist.

**Map the skill the user named to the REAL column family — never invent a column from the skill word.** The categorical families that exist in the pivoted view are:

- **shooting ability** → `"SHOT_LOCATION_GLOBAL_CATEGORICAL_SCORE"` and `"SHOT_LOCATION_POSITIONAL_CATEGORICAL_SCORE"`. This **is** the shooting score — present it to the user as their "Shooting Score" (Global / Positional).
- **passing** → `"PASSING_GLOBAL_CATEGORICAL_SCORE"` / `"PASSING_POSITIONAL_CATEGORICAL_SCORE"`
- **dribbling** → `"DRIBBLING_TRANS_GLOBAL_CATEGORICAL_SCORE"`
- **carrying** → `"CARRYING_TRANS_GLOBAL_CATEGORICAL_SCORE"`
- **goalkeeping** → the `"GK_*_CATEGORICAL_SCORE"` columns

There is **NO** `SHOOTING_*`, `FINISHING_*`, or `SHOT_PLACEMENT_*` column — "shooting ability" is measured by the **`SHOT_LOCATION`** categorical score. Do **not** synthesize a `SHOOTING_GLOBAL_CATEGORICAL_SCORE` (or any `<skill-word>_…_CATEGORICAL_SCORE`) from the name of the skill; that exact mistake — querying a non-existent `SHOOTING_GLOBAL_CATEGORICAL_SCORE` — is what made QA prompt #11 fail. Confirm the exact column name against `getSqlSchema` before querying. Example shooting comparison:

```sql
SELECT p.first_name, p.last_name,
       s."SHOT_LOCATION_GLOBAL_CATEGORICAL_SCORE"     AS shooting_global,
       s."SHOT_LOCATION_POSITIONAL_CATEGORICAL_SCORE" AS shooting_positional
FROM stat.player_stats_pivoted s
JOIN public.player p ON p.id = s."PLAYER_ID"
WHERE s."PLAYER_ID" IN ('<rashford-uuid>'::uuid, '<lewandowski-uuid>'::uuid)
```

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

1. **Do not reference columns that do not exist.** There are no `PLAYER_NAME`, `GOALS`, `ASSISTS`, or `RATING` SQL columns. Check the schema first.
2. **Do not forget double quotes on materialized view columns.** `SELECT FIT_SCORE` will fail. Use `SELECT "FIT_SCORE"`.
3. **Do not omit the schema prefix.** `FROM player` will fail. Use `FROM public.player`.
4. **Do not skip the JOIN for names.** The stats view only has `"PLAYER_ID"`, not names.
5. **Do not use uncast UUID literals.** Use `'value'::uuid` for UUID comparisons.
6. **Do not rename tables.** The scouting table is `public.scout_report`, NOT `public.scouting_report`. Copy table names exactly from the schema.
7. **Never answer a per-player scout-report question with SQL — use `organizationScoutReports`.** Counting or listing one player's scout reports over `public.scout_report` over-reports what the user can see (the table is org-scoped, not visibility-scoped); use `organizationScoutReports(filter:{search:<player name>})` and read its `totalCount` (and `edges` to list) instead (see Step 8b). SQL over `public.scout_report` is only for org-wide scout-activity aggregates (never for region questions — Step 8b-region), and even then must join by id (never `data->>'playerName'`) and filter `WHERE sr.archived_at IS NULL AND sr.parent_report_id IS NULL AND sr.processed_data IS NOT NULL` (plus the org scope), or a raw `COUNT(*)` over-counts archived, child/duplicate, and unprocessed rows.
8. **Do not silently resolve ambiguous leagues.** "Championship", "Premier League", and "Serie A" map to multiple leagues across countries — disambiguate per Step 8c before answering.
9. **Never identify a specific named player by name in SQL.** `WHERE p.last_name ILIKE '%Nunez%'` (or `= 'Nunez'`, or `IN ('Nunez')`) is accent-sensitive and silently misses the stored "Núñez" — this is exactly the regression QA caught. Resolve the player with `searchPlayers` (diacritic-folding) and filter on `s."PLAYER_ID"` instead (Step 2a).
10. **`SELECT DISTINCT` + `ORDER BY` must agree.** Postgres requires every `ORDER BY` expression to also appear in the `SELECT` list when `DISTINCT` is used (otherwise: "for SELECT DISTINCT, ORDER BY expressions must appear in select list"). Either add the ordering column to the `SELECT`, drop `DISTINCT`, or use `GROUP BY` — don't emit a `SELECT DISTINCT ... ORDER BY <unselected column>` query.

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

- If a query returns 0 rows for a player filtering question, try the `filterPlayers` tool instead — only for an attribute filter — never for a GPR or metric ranking, and never in Step 8b-rank to replace a scout criterion. It supports roleArchetypes, valuation, position, and other structured filters that may match when SQL does not
- If 0 rows are expected to be a data issue, broaden your SQL filters (remove constraints one at a time) rather than repeating the same query
- If you've retrieved the schema already, do not call getSqlSchema again -- write and execute a query
- If you've been reasoning for 2+ steps without calling a tool, either execute a query or provide a final answer with what you know
- If a tool returns an error, try a different approach: simpler parameters, a different tool, or a modified query

## Step 13: Present Results

After receiving query results:

1. Format the data as a readable table or list.
2. Highlight key insights (top performers, outliers, trends).
3. Offer follow-up queries the user might find useful (e.g., "Want to see this filtered by league?" or "Should I compare these players in detail?").
