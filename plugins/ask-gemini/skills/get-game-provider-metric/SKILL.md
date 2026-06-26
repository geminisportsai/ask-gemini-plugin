---
name: get-game-provider-metric
description: >
  Look up a third-party provider metric (xG, xA, key passes, progressive passes,
  etc.) for a specific match or a small window of matches for a single player.
  Use when the user names an opponent, a date, or "last N games".
---

# Get Game Provider Metric

Use this skill when the user wants a **per-match** provider metric for a specific named player — phrased around a named opponent ("xG vs Arsenal"), a specific date ("last weekend"), or a small recent window ("last 5 matches"). The result is per-match numbers, not a season aggregate.

If the user asks for season totals or per-90 averages, use `get_season_provider_metric` instead.

## TOP-LEVEL RULE — read this before anything else

You are **forbidden** from reporting any per-match metric value (xG, xA, key passes, passes, minutes, goals, anything specific to one fixture) **unless every single number in your response is sourced from a `listGameProviderMetrics` row returned this turn**.

This rule has no exceptions:

- **If you did not call `listGameProviderMetrics`**, you may not report a per-match value. State that you don't have per-match data and stop.
- **If `listPlayerMatches` does not include a match against the user's named opponent**, you may not report a per-match value against that opponent. State plainly "I don't see a match against [opponent] in [player]'s available fixtures" and stop. **Do not fabricate a match date from your training knowledge.** Even if you "know" that Erling Haaland played Arsenal on a specific date, that knowledge is not in this turn's tool results, so it is forbidden.
- **If `listGameProviderMetrics` returns no row for the requested match**, you may not synthesise a value from `bioData`, from the season aggregate, or from your prior knowledge. State that per-match data isn't tracked for that fixture and stop.

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
| "key passes" | Not a single field — surface `intoF3PassesTotal` + `passesIntoBoxTotal` and explain those are the per-match passing-into-danger fields available |
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
| "vs [opponent]", "against [opponent]" | Call `listPlayerMatches` for the player, filter to matches where the opponent matches the named club, keep the most recent (or all matches against that opponent if the user said "matches against"). |
| "in the [date] match", "on [date]" | Call `listPlayerMatches` and filter to the matching date. |
| "last [N] games / matches" | Call `listPlayerMatches` ordered most-recent-first, take the first N matches. |
| "last weekend", "last game" | Call `listPlayerMatches` and take the most recent match. |
| "in the [league] this season" without a specific match | This is a season-scope question — switch to `get_season_provider_metric`. |
| "[opponent] in the [league]" | Resolve the league via `listMyOrganizationsLeagues` (see empty-resolver halt below), then filter `listPlayerMatches` to that league + opponent. |

If the user names a league, follow the empty-resolver halt rule: if `listMyOrganizationsLeagues` returns empty, stop and respond:

> Your organization doesn't have any leagues configured. Please add a league in your organization settings and try again.

If the named league isn't in the resolver's results, tell the user that league isn't in scope for their organization — do not silently drop the filter.

**Named-opponent halt**: when the user names an opponent and the filtered match list from `listPlayerMatches` is empty (no match against that opponent in the player's data), **stop here**. Respond:

> I don't see a match between [player] and [opponent] in the available fixtures for [player]'s current season. I can show you their performance against other recent opponents if that helps.

**Do not** call `listGameProviderMetrics` with a guessed `gameId`. Do not assert a match took place on a date drawn from your training knowledge. The only source of truth for whether the match exists in this data is the `listPlayerMatches` result you just received.

## Step 4: Call `listGameProviderMetrics` — this step is mandatory

After resolving the player and any specific match IDs you need, you **MUST** call `listGameProviderMetrics`. Reporting a per-match metric value (xG, xA, key passes, etc.) without a corresponding `listGameProviderMetrics` call is **hallucination** — every number you surface to the user must come from a returned row of this tool, not from your prior knowledge of the player.

Two call shapes:

- **Single-match scope** (named opponent / specific date): call `listGameProviderMetrics({ playerId, gameId })` using the gameId resolved from `listPlayerMatches`. This returns one row.
- **Window scope** ("last N games" or open-ended): call `listGameProviderMetrics({ playerId })` with no `gameId`. The tool returns all per-game rows the player has data for; intersect this with the most-recent N matches from `listPlayerMatches` and present that subset.

Strict rules:

1. **Never report a per-match metric without first calling `listGameProviderMetrics`.** If you find yourself writing a number like "Haaland's xG was 0.50 vs Arsenal" and you haven't called this tool yet, stop — go call it.
2. **Never copy a value from `bioData` and call it a per-match stat.** `bioData.*GlobalCategoricalScore` fields are season-aggregate categorical scores (0–100ish), not per-match values. They are not xG, xA, etc.
3. **If `listGameProviderMetrics` returns an empty array**, tell the user the per-match provider data isn't available for that player. Do not fall back to season-aggregate fields and present them as if they were per-match values.

For multi-match runs, prefer one call without `gameId` and post-filter, rather than fanning out one call per match.

## Step 5: DNP and substitute-appearance handling — don't return zeros without context

A 0 for xG can mean three very different things:

1. **The player played and genuinely had no shot value** — that's a real 0.
2. **The player was a late sub with only 5–10 minutes** — the 0 is real but the sample is meaningless.
3. **The player was unused / suspended / injured** — there's no row at all (DNP).

Handle each:

- **DNP** (no row in `listPlayerMatches` for that fixture, or zero minutes): say so explicitly — "Saka didn't feature against Liverpool on 2026-02-08" — and **exclude the match from per-match metric output**. Don't report a metric value for a match the player didn't play.
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
| vs West Ham (A) | 2026-01-18 | DNP | — | — |
| vs Forest (H) | 2026-01-11 | 90 | 0.66 | 0.22 |

If the user explicitly asks for an average ("average xG over his last 5"), compute it from the per-match values, but **only over matches where the player played meaningful minutes** (skip DNPs and explicitly note the denominator). Always show the per-match table alongside the average so the user can see what's in the bucket.

For a single match, a one-line answer is enough: "Saka vs Arsenal on 2026-02-08: xG **0.42**, xA **0.31**, key passes **4** (90 minutes)."

## Step 7: Empty / null handling

- `listPlayerMatches` returns no match against the named opponent: tell the user the player hasn't faced that opponent in the data range. Do not return season totals instead.
- `listGameProviderMetrics` returns nothing for a resolved match: state that the per-match data isn't available for that fixture — do not synthesize a value from the season average.

## Common pitfalls

1. **Don't report metrics without calling `listGameProviderMetrics`.** This is the single biggest failure mode. Every per-match number you surface must trace to a row this tool returned. If you haven't called it, you must not report a value — say "I couldn't retrieve per-match data" and stop.
2. **Don't proceed with the wrong player.** Single-name `searchPlayers` queries frequently return a lower-tier player with the same surname. If the result's club is inconsistent with the user's implied context, re-search with the full name; if still wrong, ask the user.
3. **Don't return season totals when asked for a match.** If the user names an opponent or "last N games", you must resolve specific matches before calling the metric tool.
4. **Don't average a series silently.** "Last 5 games xG" should show all 5 values; if you only show the mean, the user can't tell whether one outlier match drove it.
5. **Don't omit minutes.** A 0.0 xG over 12 minutes is not the same as a 0.0 xG over 90 minutes.
6. **Don't fabricate DNPs.** If `listPlayerMatches` doesn't return a row for the named fixture, say so — don't invent the match.
7. **Don't use `executeSqlQuery`.** No SQL fallback for per-match provider data.
8. **Don't confuse `bioData` categorical scores with provider metrics.** Fields like `assistingPositionalCategoricalScore` are season-aggregate categorical scores on a 0–100 scale, not xG / xA / per-match values.
