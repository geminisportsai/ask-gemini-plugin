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

It builds a cohort from structured attributes (the same filter as `filterPlayers`: position, league/competition, team, nationality, age, valuation, role archetype) and then **ranks that cohort by a provider metric**, optionally filtering by per-metric `min`/`max` thresholds. Each result carries the metric value(s) it was ranked on. It is org-scoped and season-scoped (latest season with data by default).

Arguments:

- `filter` — a `PlayerAdvancedFilterInput` (same shape and unit conventions as `filterPlayers`: `minHeight`/`maxHeight` in **metres** e.g. `1.89`, valuation in **full units** e.g. `20000000`, IDs resolved via the lookup tools). **Must contain at least one criterion** — an empty filter is rejected.
- `sortByMetric` — the metric to rank by (a `ProviderMetricField` enum value, see table below). **Required.**
- `metricFilters` — optional list of `{ metric, min?, max? }` thresholds (each `metric` is a `ProviderMetricField`).
- `sortOrder` — `DESC` (default) for "most/best", `ASC` for "least".
- `seasonId` — optional; **omit for the latest season with data** — "this season" means omit it. Resolve a named or "last" season via `listSeasons`.
- `first` — page size (default 20).
- `minutesFloorLadder` — on newer backends only: season-minutes floors the server steps down through; the response's `appliedMinutesFloor` is the floor it used (see **Minutes**).

## Step 1: Build the cohort filter (resolve IDs first)

Resolve any named attributes to IDs before calling the tool, exactly as for `filterPlayers`:

- positions → `listPositions`. Page `listPositions` before using any position id: call it with `first: 100`; while `pageInfo.hasNextPage` is true, call again with `after: pageInfo.endCursor`. Never rank or filter on a partial position list — a position missing from the first page still exists. Do not pass `isGeneral`: it returns only general positions, without the abbreviations the role groups use. Finish paging `listPositions` before the first `rankPlayersByMetric` call, so the role group's ids are complete on that one call.
- leagues → `listMyOrganizationsLeagues`. A league cohort goes in `filter.playedInLeagueIds` when the `rankPlayersByMetric` tool description mentions `playedInLeagueIds`; otherwise in `filter.leagueIds`.
- teams → `listMyOrganizationsTeams`
- nationalities → `listNationalities`
- role archetypes → `listRoleArchetypes`
- preferred foot → no lookup; pass `filter.feet` directly as a `PlayerFoot` array (`LEFT` / `RIGHT` / `BOTH`)

Put these into `filter` — any position cohort goes in `primaryPositionIds` (see the role-group table below; players without a recorded primary position are left out, which is expected) — plus the league field from the leagues line above, `teamIds`, `nationalities`, `roleArchetypes`, `feet`, plus `minAge`/`maxAge`, `minValuation`/`maxValuation`, etc.). A cohort with at least one criterion is required.

**League cohort.** If the `rankPlayersByMetric` tool description mentions `playedInLeagueIds`, put a league cohort's IDs in `filter.playedInLeagueIds` instead of `leagueIds` (and instead of `competitionIds` for that league): it keeps the players who had minutes in that league in the ranked season, including players who have since moved club or league, while `leagueIds` is each player's current league. Never send `leagueIds` or `competitionIds` together with `playedInLeagueIds` for the same league. A team cohort (`teamIds`) stays each player's current team. When the description does not mention `playedInLeagueIds`, use `leagueIds` (the player's current league) — an older backend drops the unknown argument silently.

### Role groups → positions (fixed)

When the user names a role group, set `primaryPositionIds` (not `positionIds`) to the IDs (from `listPositions`) of exactly these abbreviations, so only players whose **primary** position is in the group are ranked. **Always use exactly the positions in this table for a role group** — never widen or narrow it from run to run, so the same question ranks the same cohort every time. Use only abbreviations that appear anywhere in the fully paged `listPositions` result.

| Role group the user names | Positions |
|---|---|
| Attacking midfielders ("AMs", "number 10s") | CAM, LAM, RAM |
| Defensive midfielders ("DMs", "holding midfielders") | CDM, LDM, RDM |
| Midfielders (general) | CDM, LDM, RDM, CM, LCM, RCM, CAM, LAM, RAM |
| Centre-backs ("CBs", "central defenders") | CB, LCB, RCB |
| Full-backs ("FBs", "full-backs", "wing-backs") | LB, RB, LWB, RWB |
| Wingers | LW, RW |
| Strikers ("forwards", "centre-forwards", "number 9s") | CF, LCF, RCF, SS |

A side applies to full-backs and wingers only: "left-backs" → LB, LWB; "right-backs" → RB, RWB; "left wingers" → LW; "right wingers" → RW. Wide midfielders (LM, RM) are not in the Wingers group until that is confirmed — don't add them. (`positionIds` matches any position a player lists, so a striker who also lists CAM would join an AM ranking — that is why role groups go in `primaryPositionIds`.)

**Ages:** "U<N>" means under N, so `maxAge` is N−1: "U23" → `maxAge: 22`, "U21" → `maxAge: 20`. "23 or younger" → `maxAge: 23`.

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

The advanced metrics (from `XA_90` down) are StatsBomb-360 stats scoped to the player's current league for the season (with `playedInLeagueIds`, the played-in league); the per-90 ones end in `_90`, ratios (`CROSSING_RATIO`, `PASSING_RATIO`) are 0–1 proportions.

**Do NOT invent or substitute metrics.** Only the metrics in the table above are supported. If the user names something outside it, say it's not available rather than silently ranking by a different metric. The one category genuinely **NOT in the data**:

- high-intensity **sprints**, **high-speed running**, top speed, distance covered, or any physical / GPS tracking metric — we do not ingest these.

(xA, chances created / key passes, crossing accuracy, and errors **are** available now — use the table above; don't tell the user they're missing.)

If the only metric the user named is unavailable, tell them which metrics *are* available rather than guessing.

## Step 3: Thresholds and direction

- "averaging over 4 take-ons per 90" → `metricFilters: [{ metric: SUCCESSFUL_DRIBBLES_P90, min: 4 }]`.
- "the most / best / top" → `sortOrder: DESC`; "least / fewest" → `ASC`.
- You can threshold on one metric and sort by another (e.g. threshold crosses ≥ 2/90, sort by `CROSS_XA_P90`).

**Same question, same call.** Build the call the same way every time:

- `first` = the number the user asked for (10 if none). Always pass `first` explicitly — never leave it to the tool default of 20 — and keep it the same on every call for the question.
- Omit `seasonId` unless the user named a season.
- Add no filter the user did not ask for — no extra positions, ages, leagues or thresholds — except the default minutes floor under **Minutes** below, which is always applied the same way. If the user named no cohort at all (the filter would be empty, which the tool rejects), ask which league or position to rank rather than inventing one.

**Minutes.** For any ranking except `NP_XG_TOTAL`, add a season-minutes floor so small-sample players don't top the list: append `{ metric: MINUTES_TOTAL, min: 900 }` to `metricFilters`, alongside any thresholds the user gave — unless the **Server minutes ladder** below applies, which replaces this filter — `MINUTES_TOTAL` is minutes in the **ranked season** (all competitions, also with `playedInLeagueIds`), and it comes back in each player's metric values. Use the user's number instead when they give one ("min 600 minutes" → `min: 600`). Do not pass `filter.minMinutesPlayed` for this: it is a career total across all seasons.

**Server minutes ladder.** For any ranking except `NP_XG_TOTAL`, if the `rankPlayersByMetric` tool description mentions `minutesFloorLadder` and the question gives no minutes number, send `minutesFloorLadder: [900, 600, 450, 300]` and no `MINUTES_TOTAL` metric filter, in one call: the server applies the first floor that leaves at least `first` players (else 300), and the floor used is `appliedMinutesFloor` from the response — state it (see **Step 4**). Never send `minutesFloorLadder` when the user gave a minutes number — including 900; use their number as a `MINUTES_TOTAL` floor below. When the description does not mention `minutesFloorLadder` (an older backend drops it silently), use the floor rule below.

**One call at the floor (without the server ladder).** Without the server ladder — the user gave a minutes number, or the description lacks `minutesFloorLadder` — the floor is 900, or the user's number. A minutes number in the question — including 900 — is the user's own: use it as the floor. Make exactly one `rankPlayersByMetric` call at that floor — never lower the floor automatically and never retry at a lower one on your own, even when few players qualify. The scout-report fallback in **Step 4** is the only exception to the one-call rule, and it never lowers the floor either. If that call returns fewer players than `first`, present them and offer to lower the floor (see **Step 4**). Only after the user says yes, make one follow-up call at the floor they accepted — same `filter` (including `filter.scoutReport`), `sortByMetric`, `sortOrder`, `first` and `seasonId`, and the user's other thresholds — and present it the same way. A bare yes means <LOWER> from **Step 4**.

## Step 4: Present results

**Scout criteria.** If the user also asked for players our scouts rated, recommended or think highly of: If the `rankPlayersByMetric` tool description mentions `filter.scoutReport`, add `filter.scoutReport` to the cohort filter in the same call — `{ wouldSignPlayer: true }` for "recommended / would sign", `{ startingXI: true }` for a first-11 rating, `{ investmentPlayer: true }` for an investment rating, `{ hasReport: true }` for "players we've scouted". It counts only the reports this user may see, and all criteria hold on the same report. A role named as a scout positional profile with any tie to our scouts — a verdict ("recommend", "would sign", "rated"), a profile ("profiled as", "see as"), or simply having a report ("we've scouted", "we have reports on") — is the scouts' positional profile: send it as `positionalProfiles: ["<Canonical>"]` in the same `filter.scoutReport` as the matching scout criteria — "moppers our scouts recommend" → `scoutReport: { positionalProfiles: ["Mopper"], wouldSignPlayer: true }`, "moppers we've scouted" → `scoutReport: { positionalProfiles: ["Mopper"], hasReport: true }` — and never in `roleArchetypes`. Use the canonical profile name, mapping plurals and case: Goalkeeper, Defensive Right Back, Defensive Left Back, Inverted Right Back, Inverted Left Back, Mopper, Stopper, #6, #8, #10, Right Inside Forward, Left Inside Forward, Right Winger, Left Winger, False 9, Target Man, Pure 9, Rocket ("moppers" → "Mopper", "number 6" → "#6"). If that call returns zero players, give the honest zero and offer: "I can show players whose statistical <ARCHETYPE> role archetype matches (not a scout's view) — want that?" — never run it unasked. A role with no tie to our scouts ("top 5 moppers in the Premier League") is the statistical archetype: send `roleArchetypes` with the archetype name `listRoleArchetypes` returns (e.g. `["MOPPER"]`) and label it in the answer with the exact words "the statistical <ARCHETYPE> role archetype, not a scout's view" — "not a scout's view" is required even when the user never mentioned scouts. A plain position the user names ("left wingers", "goalkeepers") stays in `positionIds`; it is a profile only when the user ties it to how our scouts profiled the player. Before answering, map every criterion phrase in the question to applied (with the field that applied it) or not applied, and name every not-applied one. Describe the ranked players as scout-rated only for criteria in a `filter.scoutReport` call that succeeded; a scout judgement with no `scoutReport` field is named as not applied. On zero results, say that only some report forms record verdicts, and never name which organizations', clubs' or report forms record a verdict or profile. A backend whose description does not mention `filter.scoutReport` rejects or ignores it, so never send it there. If a call with `scoutReport` errors, never retry without it; use the fallback: the next rule. A `scoutReport` call that returns zero players did not error: answer from it, and never fall back to reading reports or list players it did not return. If `organizationScoutReports` is in your tools, follow the scout-report procedure (read the reports through it and restrict the ranking to the players it returned). Otherwise say that criterion was not applied, and never describe the ranked players as rated or recommended by our scouts. A ranking by a provider metric or GPR is not a scout rating.

Report the ranked players with the metric value(s) they ranked on (the tool returns them per player). Always state the season — by name if you resolved it with `listSeasons`, otherwise "the latest season with data" — and that values are per-90 — except `NP_XG_TOTAL` (a season total) and `CROSSING_RATIO` / `PASSING_RATIO` (0–1 ratios). If a threshold filtered the cohort to few/no players, say so plainly.

**Answer header.** Open every ranking answer with this header, in this order, before the player list — never skip a line that applies:

1. Step-down line — only when the call used `minutesFloorLadder` and `appliedMinutesFloor` is below 900. It is the FIRST line of the answer, before anything else: "Fewer than <first> players in this cohort had 900+ minutes in <season>, so I used <A>+ minutes." (<A> = `appliedMinutesFloor`). Only when `metricFilters` carries a threshold the user asked for — not age, position, league or the minutes floor — say instead: "With your thresholds, fewer than <first> players had 900+ minutes in <season>, so I used <A>+ minutes."
2. Cohort line — pick exactly one by the call you made:
   - the call used `playedInLeagueIds`: "Players who played in <league> in <season> with at least <N> minutes; clubs shown are their current clubs." — with no floor (`NP_XG_TOTAL`): "Players who played in <league> in <season>; clubs shown are their current clubs."
   - otherwise, with a minutes floor: "Players with at least <N> minutes in <season>." (the ranked season, named as above)

<N> is the floor of the call you are presenting — with the server ladder, `appliedMinutesFloor`. Then, for a short list, say "Here are the <K> players who…", then list the players, showing each player's minutes from the metric values when a minutes floor was applied. Name any not-applied criteria after the header, never before it.

When a call returns fewer players than `first`, never call it a top <first>; say "Here are the <K> players who…" (<K> = the number it returned), and never imply more players qualified. If it returns none, say no players qualified — never "Here are the 0 players". Then, with a minutes floor, end with: "Only <K> players in this cohort had <N>+ minutes in <season>. Want me to lower the minimum (e.g. to <LOWER>)?" <LOWER> is 600 when <N> is above 600, otherwise 300; when <N> is 300 or less, make no offer. Offer to lower the minimum only when `metricFilters` carries no threshold the user asked for; otherwise say plainly that their thresholds left <K> players, without blaming the minutes. Write "player" for a count of 1. Show a player's club only when the tool returned it for that player (`clubName`) — never from memory. Rank results often carry no club; then show none and don't look one up.
