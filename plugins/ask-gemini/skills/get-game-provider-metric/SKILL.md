---
name: get-game-provider-metric
description: >
  Look up a third-party provider metric (xG, xA, progressive passes, etc.) for a
  specific match or a small window of matches for a single player (key passes
  are answered with the season value).
  Use when the user names an opponent, a date, or "last N games".
---

# Get Game Provider Metric

Use this skill when the user wants a **per-match** provider metric for a specific named player — phrased around a named opponent ("xG vs Arsenal"), a specific date ("last weekend"), or a small recent window ("last 5 matches"). The result is per-match numbers, not a season aggregate.

If the user asks for season totals or per-90 averages, use `get_season_provider_metric` instead.

## KEY PASSES — season value only

Key passes (also "chances created") exist only as a **season** figure — see Step 2 for how to fetch `playerSeasonKeyPasses90`. When the user asks for key passes, even "in his last 5 matches":

- Answer with the season value (per 90, with `playerSeasonMinutes` when present) and one line saying key passes are recorded per season, not per match.
- Do not name, list or count any match in the key-passes line — not "in his last 5", not "including his matches against …". The season figure is not a summary of particular matches, and a match list risks including fixtures that have not been played.
- Say "this season" unless a tool result gave the season's name, and name the competition only if a tool result gave it. If the value is from an earlier season that no tool named, say "the most recent season with data".
- If key passes are the only metric asked for, skip Step 4 — no `listGameProviderMetrics` call is needed. If other metrics are asked alongside (e.g. xG), still show their per-match table per Step 4, and put key passes on a separate season line beneath it that names no matches.

## TOP-LEVEL RULE — read this before anything else

You are **forbidden** from reporting any per-match metric value (xG, xA, passes, minutes, goals, anything specific to one fixture) **unless every single number in your response is sourced from a `listGameProviderMetrics` row returned this turn**.

This rule has no exceptions:

- **If you did not call `listGameProviderMetrics`**, you may not report a per-match value. State that you don't have per-match data and stop.
- **If `listPlayerMatches` does not include a match against the user's named opponent**, you may not report a per-match value against that opponent. State plainly "I don't see a match against [opponent] in [player]'s available fixtures" and stop. **Do not fabricate a match date from your training knowledge.** Even if you "know" that Erling Haaland played Arsenal on a specific date, that knowledge is not in this turn's tool results, so it is forbidden.
- **If `listGameProviderMetrics` returns no row for the requested match**, you may not synthesise a value from `bioData`, from the season aggregate, or from your prior knowledge. State that per-match data isn't tracked for that fixture and stop. In a "last N" window, skip it instead and continue with the next played match (Step 4).

Violating this rule produces hallucinated answers and is the worst possible failure mode for this skill. When in doubt, refuse with a plain "I don't have that per-match data" sentence — that is always preferable to a fabricated number.

## Step 1: Resolve the player to an ID — and verify it's the right one

Call `searchPlayers` with the player's name. **Then check the top result before using it.**

`searchPlayers` returns the first lexicographic match for a name fragment, which is often **not** the famous player the user means. Single-name queries are especially risky:

- "Haaland" → frequently a Norwegian lower-division player, not Erling Haaland.
- "Saka" → frequently a German amateur, not Bukayo Saka.
- "Rodri" → frequently a Qatari-league player, not Manchester City's Rodrigo Hernández.
- "Mbappe" → frequently a Montpellier B player, not Kylian Mbappé.

**Always sanity-check the top result against the user's implied context.** Re-search with the player's full name when the user's prompt implies a top-five-league context (the user names an EPL/La Liga/Bundesliga/Serie A/Ligue 1 opponent, or a major club) but the top result is:

- At a club outside that context (a 3rd-tier German amateur side, a Qatari club, a USL Championship side, etc.), or
- Has empty / all-null `bioData` categorical scores (a strong signal the player isn't tracked by the metric system the user is asking about).

Re-search with the full name ("Erling Haaland", "Bukayo Saka", "Rodrigo Hernández", "Kylian Mbappé"). If the re-search still doesn't return a plausible match, **stop and ask the user to confirm the player** — do not proceed with the wrong player's `player_id` and do not fabricate per-match values for a player who isn't in the data.

## Step 2: Resolve the metric the user named

Map the user's phrasing to the canonical fields returned by `listGameProviderMetrics`. The tool returns flat per-game rows with fields like `npXgTotal`, `headerXgTotal`, `cornerXaTotal`, `crossXaTotal`, `passesTotal`, `successfulPassesTotal`, `progressivePassesTotal`, `progressiveCarriesTotal`, `intoF3PassesTotal`, `passesIntoBoxTotal`, `pressuresTotal`, `pressureRegainsTotal`, `interceptionsTotal`, `tacklesTotal`, `successfulTacklesTotal`, `aerialsTotal`, `successfulAerialsTotal`, etc.

Common synonym mappings:

| User phrasing | Field on the returned row |
|---|---|
| "xG", "expected goals" | `npXgTotal` (+ note penalty xG if asked); for headed xG specifically use `headerXgTotal` |
| "npxG", "non-penalty xG" | `npXgTotal` |
| "xA", "expected assists" | The data splits xA by source: report `crossXaTotal + cornerXaTotal` as the headline (and break out the components). There is no single combined `xaTotal` field. |
| "key passes", "chances created" | **Key passes are recorded per season, not per match.** Do not label anything else as key passes — a key pass means a pass leading to a shot, which is not what these fields measure. Instead: call `listAdvancedCompetitionStats({ playerId })` and show `playerSeasonKeyPasses90` (key passes per 90) from the row whose `seasonId` and `leagueId` match the `seasonKey` and `competitionKey` of the most recent played match from `listPlayerMatches` (rows are not in date order; if no row matches, use the most recent season available and say which). Name the season/competition only from a tool result (`listSeasons` / `listMyOrganizationsLeagues`); otherwise say "this season". Include `playerSeasonMinutes` when present. Do not say the season figure covers particular matches — it is one league-season value, not a summary of the user's last N. Say in one line that key passes are available per season rather than per match. Do not build a "last N" table of stand-in metrics for key passes; if you also show `intoF3PassesTotal` / `passesIntoBoxTotal`, label them by their own names. Never answer that key passes are not tracked — they are, per season |
| "progressive passes" | `progressivePassesTotal` |
| "progressive carries" | `progressiveCarriesTotal` |
| "pass completion %" | `successfulPassesTotal / passesTotal × 100` (compute from returned fields) |
| "pressures" | `pressuresTotal` |
| "tackles + interceptions" | `tacklesTotal + interceptionsTotal` |
| "aerial duels won" | `successfulAerialsTotal` |

When the user names a metric not in the table, look at the returned row and surface the closest matching field — do not invent a value. If no field maps, tell the user the per-match data doesn't track that metric and list 2–3 related fields that are available.

## Step 3: Match resolution — pick the right matches

The match-resolution path depends on how the user phrased the question.

| User phrasing | How to resolve |
|---|---|
| "vs [opponent]", "against [opponent]" | Call `listPlayerMatches` for the player, filter to matches where the opponent matches the named club, keep the most recent **played** one — dated before the current date, never an upcoming fixture (or all played matches against that opponent if the user said "matches against"). |
| "in the [date] match", "on [date]" | Call `listPlayerMatches` and filter to the matching date. |
| "last [N] games / matches" | Take the N most recent **played** matches — see "Last N means played matches" below. |
| "last weekend", "last game" | Take the most recent **played** match (dated before the current date) — see below. |
| "in the [league] this season" without a specific match | This is a season-scope question — switch to `get_season_provider_metric`. |
| "[opponent] in the [league]" | Resolve the league via `listMyOrganizationsLeagues` (see empty-resolver halt below), then filter `listPlayerMatches` to that league + opponent, keeping played matches only. |

### Last N means played matches

`listPlayerMatches` returns the player's fixtures newest-first **including upcoming fixtures that have not been played yet**, with no played/date filter. A match counts toward "last N" only when its `matchDate` is before the current date given in your instructions. Never list an upcoming fixture in a "last N" table or count it toward N, and never present a future-dated match as played.

- Call `listPlayerMatches({ playerId, first: 30 })` once — enough to cover upcoming fixtures plus N played matches. Do not page `listPlayerMatches` with `after`: its cursor repeats the previous page's last row. If that leaves fewer than N played matches with data and `pageInfo.hasNextPage` is true, make one larger call with `first: 100` instead.
- Drop every match dated on or after the current date, then walk the rest newest-first.

`listMyOrganizationsLeagues` is paged (20 per page by default): call it with `first: 100` and, while `pageInfo.hasNextPage` is true, call again with `after: <pageInfo.endCursor>`. A league that is not on the first page is not out of scope.

If the user names a league, follow the empty-resolver halt rule: if `listMyOrganizationsLeagues` returns empty, stop and respond:

> Your organization doesn't have any leagues configured. Please add a league in your organization settings and try again.

If the named league isn't in the resolver's results, tell the user that league isn't in scope for their organization — do not silently drop the filter.

**Named-opponent halt**: when the user names an opponent and the filtered match list from `listPlayerMatches` is empty (no match against that opponent in the player's data), **stop here**. Respond:

> I don't see a match between [player] and [opponent] in the available fixtures for [player]'s current season. I can show you their performance against other recent opponents if that helps.

**Do not** call `listGameProviderMetrics` with a guessed `gameId`. Do not assert a match took place on a date drawn from your training knowledge. The only source of truth for whether the match exists in this data is the `listPlayerMatches` result you just received.

## Step 4: Call `listGameProviderMetrics` — this step is mandatory

After resolving the player and any specific match IDs you need, you **MUST** call `listGameProviderMetrics` (exception: a key-passes-only question skips this step — see KEY PASSES). Reporting a per-match metric value (xG, xA, etc.) without a corresponding `listGameProviderMetrics` call is **hallucination** — every number you surface to the user must come from a returned row of this tool, not from your prior knowledge of the player.

Two call shapes:

- **Single-match scope** (named opponent / specific date): call `listGameProviderMetrics({ playerId, gameId })` using the gameId resolved from `listPlayerMatches`. This returns one row.
- **Window scope** ("last N games"): call `listGameProviderMetrics({ playerId, gameId })` for the newest **N + 1** played matches (or every played match, if there are fewer), one call each, newest first, using each match's `matchId` from `listPlayerMatches`. The extra match is a spare, because some matches have no per-match data. Make all N + 1 calls before writing anything. Show the newest N matches that returned a row; leave out the extra one. Never put a "No data" row in a last-N table — a match without data is skipped and replaced, not shown; name it in one line under the table.
  - **Count the matches that returned a row, not the calls you made.** An empty result does not count toward N. A row with zero minutes counts as empty: it does not count toward N, and it is named as skipped (Step 5). Example: for "last 5" make 6 calls up front; if one comes back empty, the other 5 are your table, and the empty one is named as skipped.
  - Only if more than one of the N + 1 came back empty, call the next older played match, one at a time, until you have N rows or run out of played matches.
  - Before writing the table, count its rows. Fewer than N while played matches remain untried is not an answer: call the next one. Only say you found fewer than N when every played match has been tried or you have no steps left — and then say which of those it was.
  - If 3 played matches in a row return no row, stop calling per match: make the unfiltered call once, and if it is empty too, say per-match data isn't available for this player (Step 4, rule 3).
  - Why one call per match: the unfiltered call (`listGameProviderMetrics({ playerId })`) returns every game the player has data for, which for a regular starter is larger than the tool-result limit, so the recent rows you need can be cut off. A result that ends with `[truncated from` … chars] is incomplete — a match missing from it is not evidence that a match has no data.
  - For a larger window (N above 6) there are not enough steps for one call per match: make the unfiltered call, use only the rows it actually returned, and if it was truncated say that the per-match data was too large to show in full rather than reporting matches as having no data.

Strict rules:

1. **Never report a per-match metric without first calling `listGameProviderMetrics`.** If you find yourself writing a number like "Haaland's xG was 0.50 vs Arsenal" and you haven't called this tool yet, stop — go call it.
2. **Never copy a value from `bioData` and call it a per-match stat.** `bioData.*GlobalCategoricalScore` fields are season-aggregate categorical scores (0–100ish), not per-match values. They are not xG, xA, etc.
3. **If the unfiltered `listGameProviderMetrics({ playerId })` call returns an empty array**, tell the user the per-match provider data isn't available for that player. An empty result from a match-scoped call (`gameId`) only means that one match has no data: in a last-N window, skip it and continue to the next played match. Do not fall back to season-aggregate fields and present them as if they were per-match values.

For multi-match runs up to 6 matches, call once per played match with `gameId` (above); only a larger window uses the single unfiltered call.

## Step 5: DNP and substitute-appearance handling — don't return zeros without context

A 0 for xG can mean three very different things:

1. **The player played and genuinely had no shot value** — that's a real 0.
2. **The player was a late sub with only 5–10 minutes** — the 0 is real but the sample is meaningless.
3. **The player was unused / suspended / injured** — there's no row at all (DNP).

Handle each:

- **DNP** (no row in `listPlayerMatches` for that fixture, or zero minutes): say so explicitly — "Saka didn't feature against Liverpool on 2026-02-08" — and **exclude the match from per-match metric output**. Don't report a metric value for a match the player didn't play. In a "last N" window, a DNP or zero-minute match does not count toward N; list it in the skipped note under the table and take the next played match.
- **Sub appearance with low minutes** (<25 min): show the metric but include minutes in the same row so the reader can weight it. Example: "vs Chelsea (sub, 14 min): xG 0.02 / xA 0.10".
- **Started and played meaningful minutes**: surface the value with minutes alongside.

Always show minutes (or "DNP") next to every metric value.

## Step 6: Multi-match output format

For a window of matches ("last N games"), return a per-match table — not a single average:

| Match | Date | Minutes | xG | xA |
|---|---|---|---|---|
| vs Liverpool (H) | 2026-02-08 | 90 | 0.42 | 0.31 |
| vs Brighton (A) | 2026-02-01 | 78 | 0.18 | 0.04 |
| vs Chelsea (H) | 2026-01-25 | 14 | 0.02 | 0.10 |
| vs Forest (H) | 2026-01-11 | 90 | 0.66 | 0.22 |
| vs Everton (A) | 2026-01-04 | 85 | 0.31 | 0.12 |

Skipped: vs West Ham (A), 2026-01-18 — didn't feature / no data.

If the user explicitly asks for an average ("average xG over his last 5"), compute it from the per-match values, but **only over matches where the player played meaningful minutes** (low-minute sub appearances may be excluded; explicitly note the denominator). Always show the per-match table alongside the average so the user can see what's in the bucket.

For a single match, a one-line answer is enough: "Saka vs Arsenal on 2026-02-08: xG **0.42**, xA **0.31**, passes into the box **4** (90 minutes)."

## Step 7: Empty / null handling

- `listPlayerMatches` returns no match against the named opponent: tell the user the player hasn't faced that opponent in the data range. Do not return season totals instead.
- `listGameProviderMetrics` returns nothing for a resolved match: state that the per-match data isn't available for that fixture — do not synthesize a value from the season average. In a "last N" window, take the next played match instead so N matches with data are shown where they exist; say how many you found if fewer than N.

## Common pitfalls

1. **Don't report metrics without calling `listGameProviderMetrics`** (except key passes — see KEY PASSES). This is the single biggest failure mode. Every per-match number you surface must trace to a row this tool returned. If you haven't called it, you must not report a value — say "I couldn't retrieve per-match data" and stop.
2. **Don't proceed with the wrong player.** Single-name `searchPlayers` queries frequently return a lower-tier player with the same surname. If the result's club is inconsistent with the user's implied context, re-search with the full name; if still wrong, ask the user.
3. **Don't return season totals when asked for a match.** If the user names an opponent or "last N games", you must resolve specific matches before calling the metric tool. The one exception is key passes, which exist only per season — show the season value as described in Step 2.
4. **Don't average a series silently.** "Last 5 games xG" should show all 5 values; if you only show the mean, the user can't tell whether one outlier match drove it.
5. **Don't omit minutes.** A 0.0 xG over 12 minutes is not the same as a 0.0 xG over 90 minutes.
6. **Don't fabricate DNPs.** If `listPlayerMatches` doesn't return a row for the named fixture, say so — don't invent the match.
7. **Don't use `executeSqlQuery`.** No SQL fallback for per-match provider data.
8. **Don't confuse `bioData` categorical scores with provider metrics.** Fields like `assistingPositionalCategoricalScore` are season-aggregate categorical scores on a 0–100 scale, not xG / xA / per-match values.
