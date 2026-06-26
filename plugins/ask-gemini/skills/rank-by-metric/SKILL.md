---
name: rank-by-metric
description: >
  Rank a filtered cohort of players by a raw per-90 (or season-total) StatsBomb
  provider metric — npxG/90, successful take-ons/90, crosses/90, pressures/90,
  progressive passes or carries/90, OBV, etc. — optionally with min/max
  thresholds. Use when the user wants the BEST players in a cohort by a specific
  provider stat, not a single player's number and not a Gemini score.
---

# Rank Players By Provider Metric

Use this skill when the user wants to **rank a cohort of players by a raw third-party (StatsBomb) per-90 metric** — e.g. "which Serie A wingers average the most take-ons per 90", "top U23 attacking midfielders in the Belgian Pro League by npxG", "rank Championship centre-backs by pressures per 90". The single tool is **`rankPlayersByMetric`**.

Use a different intent when:

- The user wants **one named player's** provider metric → `get_season_provider_metric` (e.g. "what's Saka's npxG?").
- The user ranks by a **Gemini-computed score** (GPR, GPM, VAEP, fit score), by **goals**, or by **scout-report** data → `rank_players` (SQL / `listTopScorers`).
- There is **no metric ranking** — only attribute filters ("show me left-backs in Serie A") → `filter_players`.
- The user wants players **similar to a named anchor** → `find_similar_players`.

## What `rankPlayersByMetric` does

It builds a cohort from structured attributes (the same filter as `filterPlayers`: position, league/competition, team, nationality, age, valuation, minutes, role archetype) and then **ranks that cohort by a provider metric**, optionally filtering by per-metric `min`/`max` thresholds. Each result carries the metric value(s) it was ranked on. It is org-scoped and season-scoped (latest season with data by default).

Arguments:

- `filter` — a `PlayerAdvancedFilterInput` (same shape and unit conventions as `filterPlayers`: `minHeight`/`maxHeight` in **metres** e.g. `1.89`, valuation in **full units** e.g. `20000000`, IDs resolved via the lookup tools). **Must contain at least one criterion** — an empty filter is rejected.
- `sortByMetric` — the metric to rank by (a `ProviderMetricField` enum value, see table below). **Required.**
- `metricFilters` — optional list of `{ metric, min?, max? }` thresholds (each `metric` is a `ProviderMetricField`).
- `sortOrder` — `DESC` (default) for "most/best", `ASC` for "least".
- `seasonId` — optional; **omit for the latest season with data**. Resolve a named / "this/last" season via `listSeasons`.
- `first` — page size (default 20).

## Step 1: Build the cohort filter (resolve IDs first)

Resolve any named attributes to IDs before calling the tool, exactly as for `filterPlayers`:

- positions → `listPositions`
- leagues → `listMyOrganizationsLeagues`
- teams → `listMyOrganizationsTeams`
- nationalities → `listNationalities`
- role archetypes → `listRoleArchetypes`
- preferred foot → no lookup; pass `filter.feet` directly as a `PlayerFoot` array (`LEFT` / `RIGHT` / `BOTH`)

Put these into `filter` (`positionIds`, `leagueIds`/`competitionIds`, `teamIds`, `nationalities`, `roleArchetypes`, `feet`, plus `minAge`/`maxAge`, `minValuation`/`maxValuation`, etc.). A cohort with at least one criterion is required.

For **footedness**, include `BOTH` alongside the side — a two-footed player can play either foot: **left-footed** → `feet: [LEFT, BOTH]`, **right-footed** → `feet: [RIGHT, BOTH]`.

## Step 2: Map the user's metric to a `ProviderMetricField`

Only these metrics are supported. Map the user's phrasing to the enum value; pass it as `sortByMetric` (and inside `metricFilters`).

| User phrasing | `ProviderMetricField` |
|---|---|
| npxG / non-penalty xG (per 90) | `NP_XG_P90` |
| npxG (season total) | `NP_XG_TOTAL` |
| take-ons / successful dribbles (per 90) | `SUCCESSFUL_DRIBBLES_P90` |
| crosses (per 90) | `CROSSES_P90` |
| expected assists from crosses / cross xA | `CROSS_XA_P90` |
| pressures (per 90) | `PRESSURES_P90` |
| pressure regains / counterpressure regains | `PRESSURE_REGAINS_P90` |
| on-ball value (for) net | `OBV_FOR_NET_P90` |
| on-ball value (total) net | `OBV_TOTAL_NET_P90` |
| aerial duels (per 90) | `AERIALS_P90` |
| tackles (per 90) | `TACKLES_P90` |
| interceptions (per 90) | `INTERCEPTIONS_P90` |
| progressive passes (per 90) | `PROGRESSIVE_PASSES_P90` |
| progressive carries (per 90) | `PROGRESSIVE_CARRIES_P90` |
| passes into the box (per 90) | `PASSES_INTO_BOX_P90` |
| carry OBV gain | `CARRY_OBV_GAIN_P90` |
| dribble OBV gain | `DRIBBLE_OBV_GAIN_P90` |
| penalty goals (per 90) | `PENALTY_GOALS_P90` |
| **total xA / expected assists (per 90)** | `XA_90` (open-play only: `OP_XA_90`) |
| **npxG + xA (per 90)** | `NPXGXA_90` |
| **chances created / key passes (per 90)** | `KEY_PASSES_90` (open-play only: `OP_KEY_PASSES_90`) |
| **crossing accuracy / cross-completion ratio** | `CROSSING_RATIO` |
| **errors (leading to shot/goal) (per 90)** | `ERRORS_90` |
| **assists (per 90)** | `ASSISTS_90` (open-play: `OP_ASSISTS_90`) |
| deep progressions (per 90) | `DEEP_PROGRESSIONS_90` |
| non-penalty shots (per 90) | `NP_SHOTS_90` |
| on-ball value, total (per 90) | `OBV_90` |
| possession-adjusted tackles (per 90) | `PADJ_TACKLES_90` |
| possession-adjusted interceptions (per 90) | `PADJ_INTERCEPTIONS_90` |
| poss-adjusted tackles + interceptions (per 90) | `PADJ_TACKLES_AND_INTERCEPTIONS_90` |
| ball recoveries (per 90) | `BALL_RECOVERIES_90` |
| aggressive actions (per 90) | `AGGRESSIVE_ACTIONS_90` |
| passing accuracy / completion ratio | `PASSING_RATIO` |

The advanced metrics (from `XA_90` down) are StatsBomb-360 stats scoped to the player's current league for the season; the per-90 ones end in `_90`, ratios (`CROSSING_RATIO`, `PASSING_RATIO`) are 0–1 proportions.

**Do NOT invent or substitute metrics.** Only the metrics in the table above are supported. If the user names something outside it, say it's not available rather than silently ranking by a different metric. The one category genuinely **NOT in the data**:

- high-intensity **sprints**, **high-speed running**, top speed, distance covered, or any physical / GPS tracking metric — we do not ingest these.

(xA, chances created / key passes, crossing accuracy, and errors **are** available now — use the table above; don't tell the user they're missing.)

If the only metric the user named is unavailable, tell them which metrics *are* available rather than guessing.

## Step 3: Thresholds and direction

- "averaging over 4 take-ons per 90" → `metricFilters: [{ metric: SUCCESSFUL_DRIBBLES_P90, min: 4 }]`.
- "the most / best / top" → `sortOrder: DESC`; "least / fewest" → `ASC`.
- You can threshold on one metric and sort by another (e.g. threshold crosses ≥ 2/90, sort by `CROSS_XA_P90`).

## Step 4: Present results

Report the ranked players with the metric value(s) they ranked on (the tool returns them per player). State the season used and that values are per-90 unless the metric is a total (`NP_XG_TOTAL`). If a threshold filtered the cohort to few/no players, say so plainly.
