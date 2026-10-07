---
name: get-season-provider-metric
description: >
  Look up a season-aggregate third-party provider metric (xG, npxG, xA, key
  passes, pass completion %, PPDA, etc.) or a player's minutes in a season, for
  one or more named players. Use when the user asks for a season total or per-90
  average of a provider stat, or how many minutes a player played in a season.
---

# Get Season Provider Metric

Use this skill when the user wants a **season-level** third-party provider metric for a specific named player — xG, expected goals, npxG, xA, expected assists, key passes, progressive passes, pass completion %, PPDA, deep completions, etc. It also answers how many minutes a named player played in a season ("How many minutes did Walker, Balogun and Frimpong play in the 2025/26 season?"). The result is one player + one metric, or several named players' minutes, scoped to a season (or to "this season" / "last season").

Use a different intent when:

- The user names a specific match or opponent → `get_game_provider_metric`.
- The user asks for a Gemini-computed score (GPR, GPM, VAEP, fit score) → `lookup_stat`.
- The user wants to *rank a cohort* of players by a provider metric ("which wingers average the most take-ons/90", "top U23 AMs by npxG") → `rank_players_by_metric`.

## Step 1: Resolve the player to an ID — and verify it's the right one

Call `searchPlayers` with the player's name. **Then check the top result before using it.**

`searchPlayers` returns the first lexicographic match for a name fragment. Single-name queries for famous players often return the wrong person:

- "Haaland" → frequently a Norwegian lower-division player, not Erling Haaland.
- "Saka" → frequently a German amateur, not Bukayo Saka.
- "Rodri" → frequently a Qatari-league player, not Manchester City's Rodrigo Hernández.
- "Mbappe" → frequently a Montpellier B player, not Kylian Mbappé.

**Sanity-check the top result.** Re-search with the full name ("Erling Haaland", "Bukayo Saka", "Rodrigo Hernández", "Kylian Mbappé") when:

- The top result's `currentClub` / `currentLeague` is inconsistent with the user's implied context (the user mentioned a top-five league, or the prompt is about a globally-known player), or
- The top result's `bioData.*GlobalCategoricalScore` fields are all `null` (a strong signal the player is not tracked by the metric system the user is asking about).

If the re-search still doesn't return a plausible match, ask the user to confirm the player. Do not call `listSeasonProviderMetrics` with the wrong `player_id` and do not surface metric values for a player who isn't the one the user meant.

## Step 2: Identify the metric the user asked for

Users phrase provider metrics inconsistently. Map the user's phrasing to a canonical metric name before calling the tool. Common synonyms:

| User phrasing | Canonical metric |
|---|---|
| "xG", "expected goals", "xG (expected goals)" | xG |
| "npxG", "non-penalty xG", "expected goals excluding penalties" | npxG |
| "xA", "expected assists", "xG assisted" | xA (note: in some rows the data splits xA by source — `crossXa`, `cornerXa`, etc. If the tool returns split components rather than a single xA, sum them for the headline and break out the components in your answer) |
| "key passes", "chances created" (when "created" = passes leading to shots) | key_passes |
| "shot-creating actions", "SCA" | shot_creating_actions |
| "goal-creating actions", "GCA" | goal_creating_actions |
| "progressive passes", "progressive distance" (passes) | progressive_passes |
| "progressive carries", "ball carries forward", "progressive ball carries" | progressive_carries |
| "pass completion %", "pass accuracy", "passing percentage" | pass_completion_pct |
| "PPDA", "passes per defensive action", "pressing intensity" | ppda |
| "deep completions", "completions in the final third (passes)" | deep_completions |
| "tackles + interceptions", "T+I" | tackles_plus_interceptions |
| "pressures", "pressing events" | pressures |
| "aerial duels won", "aerial wins" | aerial_duels_won |
| "minutes", "minutes played", "how many minutes did X play" | `minutesTotal` (see **Minutes** below) |

If the user names a metric not in this table, pass through their phrasing to `listSeasonProviderMetrics` — the tool's filter parameter will surface whether that metric exists. Do not silently substitute a different metric.

### Minutes

"How many minutes did X play in <season>" is answered from the season row of `listSeasonProviderMetrics` (Steps 3 and 5). Read it like this:

- `minutesTotal` is the player's minutes in that season summed across all competitions (league, cups, Europe). It is the answer to a minutes question.
- `minutesSource` is `PROVIDER`, `TRANSFERMARKT` or `MIXED`. Say where the minutes come from in plain words: `PROVIDER` → "from match data", `TRANSFERMARKT` → "from Transfermarkt", `MIXED` → "from match data and Transfermarkt". Never name the match-data provider, a table or a field in the answer.
- Per-90 values and the other metrics come from one competition, `leagueId`, and `leagueMinutes` is that sample. When `leagueMinutes` differs from `minutesTotal`, say the per-90 is from that competition, e.g. "per 90 in the Premier League (1,178 of his 1,733 minutes this season)".
- A `TRANSFERMARKT`-only row has minutes and no provider metrics: give the minutes, and say the per-90 metrics are not available for that season.

For several named players, resolve each one (Step 1) and call `listSeasonProviderMetrics` once per player with the same `seasonId` — except for "latest season", where each player is read with no `seasonId` (see **Latest season**). Call `listSeasons` once for all of them, then per player one `searchPlayers` and one `listSeasonProviderMetrics` — no `getPlayer`, and no second `listSeasons`. If you run out of steps before every player is read, name each player you did not look up. Answer with one line per player — the player, the season's minutes and the source — so every player the user named is in the answer, including one with no row (say so for that player).

### Advanced StatsBomb-360 metrics live in a different tool

A set of advanced per-player season metrics are **not** in `listSeasonProviderMetrics`. For these, call **`listAdvancedCompetitionStats`** by `playerId`, optionally scoped to a league and season.

**Scoping rule — a requested season must never be silently dropped.** League and season go together or not at all, so asking for a season alone leaves you with a league to find:

1. If the user named both a league and a season, resolve the league via `listMyOrganizationsLeagues` and pass both IDs. That list is paged (20 per page by default): call it with `first: 100` and follow `pageInfo.hasNextPage` / `after: <pageInfo.endCursor>` to the last page — a league not on the first page is not out of scope.
2. If the user named **only a season**, resolve the player's own league first — via `listMyOrganizationsLeagues`, or the player's `currentLeague` — and pass that league with the requested season. Dropping to an unscoped call would answer about a different span than the one asked for.
3. If exactly one league cannot be identified, **ask the user which competition they mean.** Do not make an unscoped call and present the result as if it covered their season.

The metrics available through it:

- **total xA / expected assists** (open-play `OP_XA_90`), **npxG + xA** (`NPXGXA_90`)
- **key passes / chances created** (`KEY_PASSES_90`, open-play `OP_KEY_PASSES_90`)
- **crossing accuracy / cross-completion ratio** (`CROSSING_RATIO`), **passing accuracy** (`PASSING_RATIO`)
- **errors** leading to shot/goal (`ERRORS_90`), **assists** (`ASSISTS_90`)
- **deep progressions** (`DEEP_PROGRESSIONS_90`), **non-penalty shots** (`NP_SHOTS_90`), **on-ball value** (`OBV_90`)
- **possession-adjusted tackles / interceptions** (`PADJ_TACKLES_90`, `PADJ_INTERCEPTIONS_90`, combined `PADJ_TACKLES_AND_INTERCEPTIONS_90`), **ball recoveries** (`BALL_RECOVERIES_90`), **aggressive actions** (`AGGRESSIVE_ACTIONS_90`)

These are scoped to the player's current league for the season; `_90` values are per-90, ratios are 0–1 proportions. The only metrics genuinely **not** in the data are GPS tracking metrics (sprints, high-speed running, PSV-99 / top speed, distance covered) — say the metric the user named is unavailable and offer the player's Physical Score instead (Gemini's overall physical rating, which is in the data), rather than substituting a provider metric.

## Step 3: Resolve the season scope

**To scope a metric to a season you must resolve the season to its `id` (a UUID) first.** `listSeasonProviderMetrics`'s `seasonId` argument is a UUID, **not** a year string. Never pass a display string like `"2024/25"`, `"2024"`, or `"2023-2024"` as `seasonId` — that is an invalid (non-UUID) value and the call will fail or return nothing.

Resolve the season with the dedicated `listSeasons` tool:

1. Call `listSeasons`. Each row is shaped `{ id, displayYear, startYear, endYear }`. Two kinds of rows exist:
   - **Split (European-football) seasons** span two calendar years: `endYear === startYear + 1` (e.g. `{ displayYear: "2025", startYear: 2025, endYear: 2026 }` is the **2025/26** season).
   - **Single calendar-year seasons** have `startYear === endYear` (e.g. `{ displayYear: "2026", startYear: 2026, endYear: 2026 }`), for calendar-year leagues (MLS, Brazil, the Nordic leagues).
   - A split season and the calendar year it ends in are the same season: the 2025/26 row (`startYear 2025, endYear 2026`) and the calendar `2026` row (`startYear 2026, endYear 2026`) return the same `listSeasonProviderMetrics` row. Prefer the split row, and label the season by its split span ("2025/26"), never by the calendar year.
   - **`displayYear` is a single year string (e.g. `"2025"`), NOT `"2024/25"`.** Note two different rows can share the same `displayYear` (one split, one calendar) — disambiguate by `startYear`/`endYear`, never by `displayYear` alone.

2. Match the user's phrasing to one season and take its `id`:

| User phrasing | Which season to pick from `listSeasons` |
|---|---|
| "this season", "this year", no temporal phrase | The split season in progress on the current date (a season starts in July): from July, the split row whose `startYear` is the current year; before July, the one whose `endYear` is the current year. |
| "last season", "last year", "previous season" | The split season before that one. |
| "latest season", "most recent season", "latest season on record" | The most recent season that has a row for this player — not necessarily the season in progress. See **Latest season** below. |
| Named season ("2024/25", "24/25", "2024-25", "2023/24 season") | The split row whose `startYear`/`endYear` matches the named span (e.g. "2024/25" → `startYear 2024, endYear 2025`). |
| Named single year only ("in 2026", "the 2026 season") | The split season ending in that year (2025/26 for 2026) — the same season as the calendar `2026` row. |
| "career", "all time", "over his career" | Multiple seasons — don't pin one `seasonId`; return per-season rows, not a single aggregate (the tool may not pre-aggregate career totals). |

3. Pass that resolved `id` (UUID) as `seasonId` to `listSeasonProviderMetrics` in Step 5.

**Latest season.** For "latest season", "most recent season" or "latest season on record", call `listSeasonProviderMetrics` for the player with **no** `seasonId`: it returns one row per season the player has data for. Call `listSeasons` too. Match each row's `seasonId` to a `listSeasons` row. The most recent row is the one whose matched `listSeasons` row has the greatest `endYear`, then the greatest `startYear`. Take that row and its matched `listSeasons` row and name the season by that row's `label` when it has one, otherwise by its split span ("2025/26"); a `seasonId` that matches a calendar-year row names the split season ending in that year. Never resolve "latest season" to "this season" and stop at an empty in-progress row. If that season is the one in progress, answer with it, label it "(in progress)", say in one sentence that the season is still under way, and offer the last complete season — never run it unasked. A player whose most recent row is an older season (for example 2023/24) is answered with that season, called "the latest season on record". For several named players, each player's latest season is their own; name it on each player's line.

**Calendar-year leagues.** The data does not say whether a league plays a split season or a calendar year, so these rules assume a split season. When the season came from a bare year ("2026") or from "this season", "last season" or "latest season", state the assumption once in the answer, with the label built from the resolved row: "<startYear>/<last two digits of endYear> season (for calendar-year leagues such as MLS, that's the <endYear> season)". For example, the row `startYear 2026, endYear 2027` gives "2026/27 season (for calendar-year leagues such as MLS, that's the 2027 season)". A season the user named ("2025/26", "25/26", "2026") is used as named, under the one-season rule — never re-interpreted, and no fallback to another season. When "this season" resolved to the split season in progress and the player has no row for it, do not stop at "no data": say that season has no data for the player yet, say a calendar-year league's current season is the previous season here — the split row whose `endYear` is the resolved row's `startYear` — and offer it. For a single-player minutes question, also read that previous season (`listSeasonProviderMetrics` with its `seasonId`) and show it, labelled from its own row: "<startYear>/<last two digits of endYear> season (the <endYear> season in a calendar-year league)".

**Report the season you actually used, honestly.** State the resolved season in your answer by its split span (e.g. "in the 2025/26 season"). Never claim you used a season you didn't resolve. If there is no row for the season you resolved, say so. Never report another season's number in its place — not the latest season's, and not a career total. "Latest season" names no fixed season, so the most recent season with a row is the season asked for, not a substitute.

If you cannot call `listSeasons`, or the user's named season doesn't match any returned row, tell the user you can't resolve that season rather than guessing — do **not** fall back to calling `listSeasonProviderMetrics` with no season and then report whatever comes back as if it were the requested season. The no-`seasonId` call is right only for "latest season" (see **Latest season**), and only because that season is then named from its own row.

## Step 4: Per-90 vs total

Default behaviour:

- "How many [X] does Y have" → **season total**.
- "What's Y's [X] per 90" / "average [X] per game" / "per match" → **per-90** (or per-match if that's the granularity the tool exposes).
- A bare "What's Y's xG?" with no qualifier → return both the season total and the per-90 in your response so the user has context for sample size.

The tool returns whatever fields it supports — don't manufacture per-90 by dividing yourself if it isn't returned; surface what came back and note the absent field.

## Step 5: Call `listSeasonProviderMetrics`

Call with the resolved `playerId`, the resolved metric name, and the resolved season `seasonId` from Step 3. If the tool supports a multi-metric query and the user named several ("xG and xA"), include both.

**The `seasonId` you pass MUST be the UUID resolved via `listSeasons` in Step 3** — never a year string like `"2024/25"` or `"2024"`. If you intended a specific season, you must pass its resolved `seasonId`. Do **not** "recover" from a rejected non-UUID value by re-calling `listSeasonProviderMetrics` with no season param and then presenting the number as though it were the requested season — that produces an answer scoped to the wrong season. If the season can't be resolved, follow the Step 3 fallback and tell the user. Exception: for "latest season", call with no `seasonId` and name the season from the row (Step 3, **Latest season**).

## Step 6: Sample-size context — always show minutes

Provider metrics are misleading without minutes played. **Always** include the player's minutes for that season (`minutesTotal`) in the response and, when it differs, the minutes behind the per-90 (`leagueMinutes`, see **Minutes** in Step 2). One-line example:

> Saka's xG this season: **8.4** (per 90: **0.42** over **1,797 minutes**).

For a minutes question, the minutes are the answer: give `minutesTotal` with its source (see **Minutes** in Step 2).

If minutes are very low (under ~500 in a top-five league season), prepend a one-line caveat: "Small sample — Saka has only played 410 minutes this season, so per-90 figures are noisy."

## Step 7: Empty / null handling

- The tool returns no row for that (player, metric, season): tell the user the metric isn't available for that player in that season. For "latest season" with no rows at all, say the player has no season data on record. Do not substitute a similar metric, do not fall back to SQL, do not guess.
- The metric name isn't recognised by the tool: tell the user the metric isn't tracked, list 2–3 close synonyms from the table above as suggestions, and stop.

## Step 8: Present the result

- One sentence with the headline number, the per-90 in parentheses if relevant, and the minutes for context.
- If the user asked for multiple metrics on the same player, render as a 2-column table (metric, value).
- Don't mention table names, the match-data provider's name, or implementation details. Speak about the metric in plain terms. Naming the source class of minutes ("from match data", "from Transfermarkt") is fine — see **Minutes** in Step 2.

## Common pitfalls

1. **Don't substitute a different metric.** If the user asked for xA and the tool only has xG, say so — do not return xG and label it as xA.
2. **Don't aggregate seasons silently.** If the user asked for "this season" and you return a career total, the number is wrong.
3. **Don't divide by minutes yourself.** Per-90 is a specific computation; only report it if the tool returned it.
4. **Don't drop minutes context.** A 0.4 xG/90 means very different things at 200 minutes vs 2,500 minutes — surface the sample size.
5. **Don't use `executeSqlQuery`.** No SQL fallback. If the tool can't answer the question, surface that to the user.
6. **Don't report a career total as a season's minutes.** `getPlayer`'s `bioData.minutesTotal` is a **career** total. Never present it as one season's minutes. A season's minutes are the season row's `minutesTotal`.
