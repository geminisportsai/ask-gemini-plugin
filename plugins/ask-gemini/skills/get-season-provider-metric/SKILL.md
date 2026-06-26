---
name: get-season-provider-metric
description: >
  Look up a season-aggregate third-party provider metric (xG, npxG, xA, key
  passes, pass completion %, PPDA, etc.) for a specific player. Use when the
  user asks for a season total or per-90 average of a provider stat.
---

# Get Season Provider Metric

Use this skill when the user wants a **season-level** third-party provider metric for a specific named player — xG, expected goals, npxG, xA, expected assists, key passes, progressive passes, pass completion %, PPDA, deep completions, etc. The result is one player + one metric, scoped to a season (or to "this season" / "last season").

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

If the user names a metric not in this table, pass through their phrasing to `listSeasonProviderMetrics` — the tool's filter parameter will surface whether that metric exists. Do not silently substitute a different metric.

### Advanced StatsBomb-360 metrics live in a different tool

A set of advanced per-player season metrics are **not** in `listSeasonProviderMetrics`. For these, call **`listAdvancedCompetitionStats`** (by `playerId`, optionally scoped to a league+season — resolve a named league via `listMyOrganizationsLeagues`; pass league and season together or neither):

- **total xA / expected assists** (open-play `OP_XA_90`), **npxG + xA** (`NPXGXA_90`)
- **key passes / chances created** (`KEY_PASSES_90`, open-play `OP_KEY_PASSES_90`)
- **crossing accuracy / cross-completion ratio** (`CROSSING_RATIO`), **passing accuracy** (`PASSING_RATIO`)
- **errors** leading to shot/goal (`ERRORS_90`), **assists** (`ASSISTS_90`)
- **deep progressions** (`DEEP_PROGRESSIONS_90`), **non-penalty shots** (`NP_SHOTS_90`), **on-ball value** (`OBV_90`)
- **possession-adjusted tackles / interceptions** (`PADJ_TACKLES_90`, `PADJ_INTERCEPTIONS_90`, combined `PADJ_TACKLES_AND_INTERCEPTIONS_90`), **ball recoveries** (`BALL_RECOVERIES_90`), **aggressive actions** (`AGGRESSIVE_ACTIONS_90`)

These are scoped to the player's current league for the season; `_90` values are per-90, ratios are 0–1 proportions. The only metrics genuinely **not** in the data are physical/GPS tracking (sprints, high-speed running, top speed, distance covered) — say those are unavailable rather than substituting.

## Step 3: Resolve the season scope

**To scope a metric to a season you must resolve the season to its `id` (a UUID) first.** `listSeasonProviderMetrics`'s `seasonId` argument is a UUID, **not** a year string. Never pass a display string like `"2024/25"`, `"2024"`, or `"2023-2024"` as `seasonId` — that is an invalid (non-UUID) value and the call will fail or return nothing.

Resolve the season with the dedicated `listSeasons` tool:

1. Call `listSeasons`. Each row is shaped `{ id, displayYear, startYear, endYear }`. **Two kinds of rows exist and they are NOT interchangeable:**
   - **Split (European-football) seasons** span two calendar years: `endYear === startYear + 1` (e.g. `{ displayYear: "2025", startYear: 2025, endYear: 2026 }` is the **2025/26** season). These are the seasons player provider metrics (xG, xA, passing, etc.) are reported against.
   - **Single calendar-year seasons** have `startYear === endYear` (e.g. `{ displayYear: "2026", startYear: 2026, endYear: 2026 }`). These are calendar-year competitions/aggregates and usually have **no** European-football provider data — picking one for "this season" returns an empty result and a wrong answer.
   - **`displayYear` is a single year string (e.g. `"2025"`), NOT `"2024/25"`.** Note two different rows can share the same `displayYear` (one split, one calendar) — disambiguate by `startYear`/`endYear`, never by `displayYear` alone.

2. Match the user's phrasing to one season and take its `id`:

| User phrasing | Which season to pick from `listSeasons` |
|---|---|
| "this season", "this year", no temporal phrase | The **most recent split season** — the row with the greatest `endYear` among rows where `endYear === startYear + 1`. (As of 2026 that is `startYear 2025, endYear 2026` = the 2025/26 season — **not** the calendar-year `2026` row.) |
| "last season", "last year", "previous season" | The **next-most-recent split season** (the split season with the second-greatest `endYear`). |
| Named season ("2024/25", "2023/24 season") | The split row whose `startYear`/`endYear` matches the named span (e.g. "2024/25" → `startYear 2024, endYear 2025`). |
| Named single year only ("in 2026", "the 2026 season") | Only then consider the calendar-year row (`startYear === endYear`) for that year, if one exists. |
| "career", "all time", "over his career" | Multiple seasons — don't pin one `seasonId`; return per-season rows, not a single aggregate (the tool may not pre-aggregate career totals). |

3. Pass that resolved `id` (UUID) as `seasonId` to `listSeasonProviderMetrics` in Step 5.

**Report the season you actually used, honestly.** State the resolved season in your answer using its real span (e.g. "in the 2025/26 season"). Never claim you used a season you didn't resolve, and never label the calendar-year `2026` row as "2025/26". If `listSeasonProviderMetrics` returns no row for the season you resolved, say the metric isn't available for that player in that season (Step 7) — do **not** silently report a different season's number, and do **not** claim "I don't have that data in the system" if you simply picked the wrong (calendar-year) season: re-resolve to the most recent **split** season first.

If you cannot call `listSeasons`, or the user's named season doesn't match any returned row, tell the user you can't resolve that season rather than guessing — do **not** fall back to calling `listSeasonProviderMetrics` with no season and then report whatever comes back as if it were the requested season.

## Step 4: Per-90 vs total

Default behaviour:

- "How many [X] does Y have" → **season total**.
- "What's Y's [X] per 90" / "average [X] per game" / "per match" → **per-90** (or per-match if that's the granularity the tool exposes).
- A bare "What's Y's xG?" with no qualifier → return both the season total and the per-90 in your response so the user has context for sample size.

The tool returns whatever fields it supports — don't manufacture per-90 by dividing yourself if it isn't returned; surface what came back and note the absent field.

## Step 5: Call `listSeasonProviderMetrics`

Call with the resolved `playerId`, the resolved metric name, and the resolved season `seasonId` from Step 3. If the tool supports a multi-metric query and the user named several ("xG and xA"), include both.

**The `seasonId` you pass MUST be the UUID resolved via `listSeasons` in Step 3** — never a year string like `"2024/25"` or `"2024"`. If you intended a specific season, you must pass its resolved `seasonId`. Do **not** "recover" from a rejected non-UUID value by re-calling `listSeasonProviderMetrics` with no season param and then presenting the number as though it were the requested season — that produces an answer scoped to the wrong season. If the season can't be resolved, follow the Step 3 fallback and tell the user.

## Step 6: Sample-size context — always show minutes

Provider metrics are misleading without minutes played. **Always** include the player's minutes for that season in the response (if the tool returns it in the same row, surface it; if not, mention that the metric is over the player's available minutes for that season). One-line example:

> Saka's xG this season: **8.4** (per 90: **0.42** over **1,797 minutes**).

If minutes are very low (under ~500 in a top-five league season), prepend a one-line caveat: "Small sample — Saka has only played 410 minutes this season, so per-90 figures are noisy."

## Step 7: Empty / null handling

- The tool returns no row for that (player, metric, season): tell the user the metric isn't available for that player in that season. Do not substitute a similar metric, do not fall back to SQL, do not guess.
- The metric name isn't recognised by the tool: tell the user the metric isn't tracked, list 2–3 close synonyms from the table above as suggestions, and stop.

## Step 8: Present the result

- One sentence with the headline number, the per-90 in parentheses if relevant, and the minutes for context.
- If the user asked for multiple metrics on the same player, render as a 2-column table (metric, value).
- Don't mention table names, provider names, or implementation details. Speak about the metric in plain terms.

## Common pitfalls

1. **Don't substitute a different metric.** If the user asked for xA and the tool only has xG, say so — do not return xG and label it as xA.
2. **Don't aggregate seasons silently.** If the user asked for "this season" and you return a career total, the number is wrong.
3. **Don't divide by minutes yourself.** Per-90 is a specific computation; only report it if the tool returned it.
4. **Don't drop minutes context.** A 0.4 xG/90 means very different things at 200 minutes vs 2,500 minutes — surface the sample size.
5. **Don't use `executeSqlQuery`.** No SQL fallback. If the tool can't answer the question, surface that to the user.
