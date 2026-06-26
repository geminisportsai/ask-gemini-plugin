---
name: find-similar-players
description: >
  Find players who are statistically or stylistically similar to a single
  named anchor player. Use when the user asks for alternatives, replacements,
  cheaper/younger versions of a player, or "who plays like X".
---

# Find Similar Players

Use this skill when the user names **one anchor player** and wants other players who resemble them — alternatives, replacements, "cheaper versions", "younger versions", or "plays like X". The result is a ranked list of candidate players similar to the anchor.

Do **not** use this skill for:

- "Which teams fit X" → that is `player_team_fit`.
- "Rank players by metric Y" with no anchor → that is `rank_players`.
- "Compare X and Y" with two named players → that is `compare_players`.

## Step 1: Resolve the anchor player to an ID — and verify it's the right one

Call `searchPlayers` with the anchor's name (e.g., "Pedri", "Kevin De Bruyne"). **Then check the top result before using it.**

`searchPlayers` returns the first lexicographic match for a name fragment, which is often **not** the famous player the user means. Single-name queries are especially risky:

- "Haaland" → frequently a Norwegian lower-division player, not Erling Haaland.
- "Saka" → frequently a German amateur, not Bukayo Saka.
- "Rodri" → frequently a Qatari-league player, not Manchester City's Rodrigo Hernández.
- "Mbappe" → frequently a Montpellier B player, not Kylian Mbappé.

Re-search with the full name when the top result's `currentClub` / `currentLeague` doesn't match the player's well-known club, or when the result's `bioData.*GlobalCategoricalScore` fields are all `null`. If the re-search still doesn't return a plausible match, ask the user to confirm — do not call `listSimilarPlayers` with the wrong anchor `player_id`.

## Step 2: Identify the candidate-pool filters from the user's phrasing

This is the most important step. "Similar to X" almost always implies a filter on the **candidate pool** (not the anchor). Common modifiers and what they map to:

| Phrasing | Filter on candidates |
|---|---|
| "Cheaper alternative to X" | Candidate valuation strictly less than the anchor's valuation. |
| "Younger replacement for X" | Candidate age strictly less than the anchor's age. |
| "Older / more experienced version of X" | Candidate age greater than the anchor's age. |
| "[League] alternative to X" / "Premier League version of X" | Restrict candidates to the named league (resolve via `listMyOrganizationsLeagues`). |
| "Forwards similar to X" / "midfielders similar to X" | Restrict candidates to the named position group. |
| "Who plays like X" (no modifier) | No candidate filter — the user wants the raw similarity list. |
| "Replacement for X" (no modifier) | Treat as raw similarity unless other phrasing suggests a constraint. |

The anchor itself defines the "similarity target", **not** the candidate filter. Do not, for example, restrict candidates to the anchor's own team or league unless the user explicitly asked.

For league constraints, follow the empty-resolver halt rule below.

## Step 3: Resolve any named league to an ID

If the user names a specific league ("cheaper Premier League version of Pedri"), call `listMyOrganizationsLeagues` and pick the matching `league_id`.

**Empty-resolver halt** — if `listMyOrganizationsLeagues` returns an empty list, **stop immediately** and respond with this exact message (no preamble, no caveats, no partial answer):

> Your organization doesn't have any leagues configured, so I can't scope this comparison. Please add a league to your organization in the settings and try again.

If the resolver returns results but the named league isn't among them, respond:

> Your organization doesn't have the [league name] league configured. Please add it in your organization settings and try again.

Do not silently drop the league filter or substitute a different league.

## Step 4: Call `listSimilarPlayers` — put candidate constraints in `filter`

Call `listSimilarPlayers` with the resolved anchor `playerId` and put **all candidate-pool constraints in the `filter` argument** (a `SimilarPlayerFilterInput`). The tool applies them **server-side**, so you do **not** enrich candidates one by one. Supported `filter` fields:

- `minValuation` / `maxValuation` — price caps in full units (e.g. "under £25m / €25m" → `maxValuation: 25000000`).
- `minAge` / `maxAge` — age caps ("under 23" → `maxAge: 23`).
- `leagueIds` — restrict to named leagues (resolve via `listMyOrganizationsLeagues`; follow the empty-resolver halt rule above).
- `minMinutesPlayed` / `maxMinutesPlayed`, `minMonthsRemaining` / `maxMonthsRemaining`.
- `sortBy` / `sortOrder` (default: `similarityScore` DESC — leave as-is unless the user asks otherwise).

If the user asked for a specific number of results ("top 5 alternatives"), pass it as `first`. Otherwise let the tool return its default (20).

If the user names a constraint the `filter` genuinely cannot express — a **region** like "in Europe" (no single league list) or a position group — apply what the filter supports and state the unmet part in your prose. Do **not** fabricate an argument name and do **not** enrich every candidate to emulate it.

### Value/age qualifiers: use the `filter`, don't enrich the candidate list

The `listSimilarPlayers` nodes are scalars-only (similarity score, no valuation/age), but that does **not** mean you must give up or enrich the whole list. Use the server-side `filter`:

1. **Absolute caps** ("under £25m", "under 23"): pass `filter.maxValuation` / `filter.maxAge` directly — one call, no enrichment.
2. **Relative caps** ("cheaper than Saka", "younger than De Bruyne"): call `getPlayerBioDataByPlayerId` on the **anchor only** (a single call) to read its valuation/age, then pass that number as `filter.maxValuation` / `filter.maxAge`.
3. Present the qualifier explicitly ("valued below Saka's €74M tag").

**Never** loop `getPlayerBioDataByPlayerId` over every returned candidate to filter by valuation/age — the `filter` does it server-side, and per-candidate enrichment is slow enough to **time the request out**. The only legitimate `getPlayerBioDataByPlayerId` call here is the single anchor lookup for a relative threshold. And **never** answer a "cheaper/younger" request with "valuation/age data isn't available" — pass the cap to `filter` instead.

## Step 5: Handle empty / null results

If `listSimilarPlayers` returns an empty array, tell the user no similar players were found under the requested constraints — do not substitute an unfiltered list or fall back to SQL. Suggest broadening one constraint (e.g., "if I lift the cheaper-than-anchor restriction, I can show similar players at any price"), then wait for confirmation before re-running.

## Step 6: Present the result

- Lead with the anchor: "Players similar to **Pedri** (Barcelona, 21yo, midfielder):" then list candidates.
- Include the candidate's club, age, and (when relevant to the user's filter) valuation in the row.
- Pass through whatever similarity score the tool returns — do not invent one and do not normalize it to a different scale.
- If the user named a constraint (cheaper, younger, in-league), confirm it in one sentence: "All five are valued below Pedri's €100M tag" / "All from Premier League sides".

## Common pitfalls

1. **Don't use `executeSqlQuery`** — there is no SQL fallback for similarity. If `listSimilarPlayers` doesn't satisfy the request, say so and stop; do not write SQL against player_vector or any similar table.
2. **Don't filter on the anchor's league/team unless asked** — "similar to Pedri" does not mean "must also be at Barcelona" or "must also be in La Liga". The anchor defines the target style, the candidate pool is global by default.
3. **Don't conflate with team-fit** — "find a player who fits Liverpool's style" is player-team-fit (a different intent). This skill is player-to-player similarity, not player-to-team fit.
4. **Don't skip the anchor lookup** — calling `listSimilarPlayers` with a player name instead of an ID will fail. Always `searchPlayers` first.
