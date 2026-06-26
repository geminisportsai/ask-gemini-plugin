---
name: player-summary
description: >
  Build a comprehensive player profile including bio data, performance scores,
  and match history. Use when the user asks about a specific player's details or overview.
---

# Player Summary

Use this skill when the user asks about a specific player's profile, details, stats overview, or performance history.

## Step 1: Get Basic Player Information

Call `getPlayer` or `getPlayerBioDataByPlayerId` with the player's ID to retrieve biographical data:

- Full name, date of birth, age
- Current club and contract status
- Physical attributes (height, weight, preferred foot)
- Nationality and citizenship
- Position information

If you only have the player's name (not their ID), use `searchPlayers` from the `player-search` skill first to find the player ID.

## Step 2: Get Performance Summary

Call `getPlayerDataSummary` with the player's ID to retrieve aggregated performance data. This provides an overall view of the player's quality and contribution metrics.

## Step 3: Get Seasonal Scores

Call `getPlayerSeasonCategoricalScores` with the player's ID to retrieve season-level categorical scores (aka attributes). This shows how the player's performance profile has trended across seasons and across different skill areas.

## Step 4: Get Match History

Call `listPlayerMatches` with the player's ID to retrieve recent match appearances. This provides context on playing time, match results, and competition level.

## Step 4b: Read Scout Reports (required for "what do my/our scouts think")

The organization's scouts write their own reports on players. When the user's
question is about **scout opinion** — e.g. "what do my scouts think of X",
"what do our scouts say about X", "how did we scout X", "our scouts' view on X" —
those reports are the **primary source**, not bio data or categorical scores.

- Call `organizationScoutReports(filter: { search: "<player name>" })` to retrieve
  the org's reports for the player. Use the returned report content — the scouts'
  ratings (offensive/defensive/athleticism/game-intelligence), notes, and
  recommendations — to answer the question.
- If reports exist (`totalCount > 0`), base the scout-opinion answer on what the
  reports actually say, and make clear it comes from your scouts' reports.
- If `totalCount` is 0, say plainly that your scouts have not written any reports
  on this player yet — then optionally offer the data-driven profile instead.
- **Never fabricate scout opinion.** Do not relabel bio data, GPR, or categorical
  scores as "Scout Assessment" / "Scout Recommendation" / "our scouts rated…" when
  no scout report was read. Presenting model-derived data as a scout's view is a
  defect. Scout language is only warranted when it is backed by an actual report.

For a general profile request (no explicit scout-opinion ask), reading scout
reports is optional enrichment; include a brief scout note only if reports exist.

## Step 5: Response Formatting

- Never mention database table names, column names, SQL queries, joins, or any data retrieval methods in your answer
- Use human-friendly names for all metrics: say "GPR" not "TIME_DECAYED_GPR", "Fit Score" not "FIT_SCORE", "Valuation" not "PLAYER_VALUATION"
- GPR stands for "Gemini Player Rating" -- never say "General Performance Rating" or "Gemini Performance Rating"
- Focus on insights and results, not how data was retrieved

## Step 6: Present Comprehensive Profile

Combine the data from all previous steps into a structured profile:

1. **Header**: Player name, age, nationality, current club, and position.
2. **Performance Overview**: Key scores from the data summary (FIT_SCORE, GPR, or other available metrics).
3. **Season Trends**: Highlight improvement or decline across seasons using the categorical scores.
4. **Recent Matches**: List recent appearances with key details (competition, result, minutes played).
5. **Insights**: Note any standout observations -- career trajectory, strengths, or areas the data highlights.

Offer follow-up actions such as comparing the player with others (via `sql-analytics`) or adding them to a watchlist (via `watchlist-management`).
