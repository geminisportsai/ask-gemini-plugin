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
- "Compare X and Y" with two named players → that is `compare_players`. A similarity score between two named players stays here — see "Similarity score between two named players" below.

## Step 0: A price cap in another currency — ask first

Market valuations are recorded in euros, and nothing here converts a currency. Check this before **Step 1**: a cap in another currency is asked about before any tool call, even `searchPlayers` for the anchor.

- A cap in euros ("under €25m", "25m EUR") is used as given.
- A bare figure ("under 25m") is read as euros; say so: "I read 25m as €25m."
- A price cap in pounds, dollars or any other non-euro currency ("under £25m", "under $30m"): make no tool call and name no exchange rate. Reply with one short question that keeps the user's figure and currency: "Valuations are recorded in euros. What euro amount should I use for your £25m?" Give no list of players. Search only once the user gives a euro figure. Never convert a currency yourself, and never read pounds or dollars as euros.
- If the request also names a region ("in Europe"), ask only the euro question, with no sentence about the region: the region goes in the `filter` once the search runs (see **A club region: `clubContinents` and `clubRegions`**).

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
| "In Europe" / "from South American clubs" (a continent) | Restrict candidates to clubs on that continent: `filter.clubContinents` (see **A club region** under Step 4). |
| "In the UK & Ireland" (an FM24 region) / "Scandinavian clubs" (an area one FM24 region covers) | Restrict candidates to clubs in that FM24 region: `filter.clubRegions`, with the exact region name (see **A club region** under Step 4). |
| "Forwards similar to X" / "midfielders similar to X" | Restrict candidates to the named position group. |
| "Who plays like X" (no modifier) | No candidate filter — the user wants the raw similarity list. |
| "Replacement for X" (no modifier) | Treat as raw similarity unless other phrasing suggests a constraint. |

The anchor itself defines the "similarity target", **not** the candidate filter. Do not, for example, restrict candidates to the anchor's own team or league unless the user explicitly asked.

For league constraints, follow the empty-resolver halt rule below.

## Step 3: Resolve any named league to an ID

If the user names a specific league ("cheaper Premier League version of Pedri"), call `listMyOrganizationsLeagues` and pick the matching `league_id`. The list is paged (20 per page by default): call it with `first: 100` and, while `pageInfo.hasNextPage` is true, call again with `after: <pageInfo.endCursor>`. A league that is not on the first page is not out of scope.

**Empty-resolver halt** — if `listMyOrganizationsLeagues` returns an empty list, **stop immediately** and respond with this exact message (no preamble, no caveats, no partial answer):

> Your organization doesn't have any leagues configured, so I can't scope this comparison. Please add a league to your organization in the settings and try again.

If the resolver returns results but the named league isn't among them, respond:

> Your organization doesn't have the [league name] league configured. Please add it in your organization settings and try again.

Do not silently drop the league filter or substitute a different league.

## Step 4: Call `listSimilarPlayers` — put candidate constraints in `filter`

Call `listSimilarPlayers` with the resolved anchor `playerId` and put **all candidate-pool constraints in the `filter` argument** (a `SimilarPlayerFilterInput`). The tool applies them **server-side**, so you do **not** enrich candidates one by one. Supported `filter` fields:

- `minValuation` / `maxValuation` — market valuation caps in **whole euros** ("under €25m" → `maxValuation: 25000000`); a cap in another currency gets the **Step 0** question first.
- `minAge` / `maxAge` — age caps ("under 23" → `maxAge: 23`).
- `leagueIds` — restrict to named leagues (resolve via `listMyOrganizationsLeagues`; follow the empty-resolver halt rule above).
- `clubContinents` — restrict to clubs on a continent (`EUROPE`, `AFRICA`, `ASIA`, `NORTH_AMERICA`, `SOUTH_AMERICA`, `OCEANIA`); "in Europe" → `clubContinents: [EUROPE]`.
- `clubRegions` — restrict to clubs in FM24 scouting regions, each name exactly as written in the table under **A club region** (several names are combined with OR).
- `minMinutesPlayed` / `maxMinutesPlayed`, `minMonthsRemaining` / `maxMonthsRemaining`.
- `sortBy` / `sortOrder` (default: `similarityScore` DESC — leave as-is unless the user asks otherwise).

If the user asked for a specific number of results ("top 5 alternatives"), pass it as `first`. Otherwise let the tool return its default (20).

If the user names a constraint the `filter` genuinely cannot express, such as a position group, apply what the filter supports and state the unmet part in your prose. A region is not one of these: it goes in `clubContinents` or `clubRegions` (see **A club region: `clubContinents` and `clubRegions`** below). Do **not** fabricate an argument name and do **not** enrich every candidate to emulate it.

### A club region: `clubContinents` and `clubRegions`

A region means where the player's **club** plays: the backend maps each player's current league country to an FM24 scouting region and filters by it server-side. When the user asks for a region ("in Europe", "from South American clubs", "Scandinavian clubs"), put it in the `filter`:

- A continent goes in `clubContinents`, one value per continent: Europe → `EUROPE`, Africa → `AFRICA`, Asia → `ASIA`, North America → `NORTH_AMERICA`, South America → `SOUTH_AMERICA`, Oceania → `OCEANIA`. "In Europe" → `clubContinents: [EUROPE]`, which covers FM24's eight European regions (Turkey is in; Israel and Kazakhstan count as Asia). `NORTH_AMERICA` includes Central America and the Caribbean, and `SOUTH_AMERICA` is both South America regions.
- An FM24 region goes in `clubRegions`, with each name exactly as written in the table below; several names are combined with OR. "In the UK & Ireland" → `clubRegions: ["UK & Ireland"]`. An area the table does not name but one region covers ("Scandinavian clubs") uses that region (`clubRegions: ["Northern Europe"]`), and the answer says which FM24 region was used and the countries it covers.
- To add a region to a continent ("in Europe or the Middle East"), put every name in `clubRegions`: the continent's FM24 regions plus the other one. Europe's are Central Europe, Eastern Europe, North Eastern Europe, Northern Europe, South Eastern Europe, South Europe, UK & Ireland and Western Europe; Africa's are Central Africa, East Africa, North Africa, Southern Africa and Western Africa; Asia's are Central Asia, East Asia, Middle East, South Asia and Southeast Asia; North America's are Caribbean, Central America and North America; South America's are South America (North) and South America (South); Oceania's is Oceania. Set both fields only to narrow, when the region lies inside the continent ("Scandinavian clubs in Europe"): a player must then match both.
- A single country ("clubs in Spain") is not a region value, and the `filter` has no country field. Before searching, offer the FM24 region that contains it ("Western Europe") or, when the user means a league, the league itself (`leagueIds`, Step 3), and wait for the answer.
- A nationality ("Brazilian players", "players born in Norway") is not a club region. Say the search filters by where a player's club plays, not by nationality, and ask whether to search clubs in that area instead. When a phrase can be read either way ("South American players"), filter by the club region and say that is how you read it: "I've read this as players at South American clubs; the search can't filter by nationality."
- `BAD_USER_INPUT` from `listSimilarPlayers` comes in two kinds. "Unknown scouting region(s): …" lists the valid FM24 names: retry once with the matching valid name, and if none matches, say the region isn't one of the FM24 scouting regions and list them. "… have no region in common …" means `clubRegions` and `clubContinents` exclude each other: retry once with only one of them, the one that says what the user asked. Never run the search without the region instead.
- Any other error on a region field (for example a backend that rejects `clubContinents` or `clubRegions` as an unknown argument) means region filtering isn't available there: say so once, offer to search without the region, and wait for the answer.
- Never stand a league list (`leagueIds`) in for a region, and never infer or label a player's region or country from their club's name. Whatever the backend returns, never drop, reorder or flag a player because you think their club is outside the region, and show each club exactly as the tool returns it.
- The region counts as applied only when the response's `unknownClubRegionExcludedCount` is a number (0 included). When the region was applied, never say you couldn't restrict the results to it. When the field is null or missing, the backend did not apply the region: say once, before the list, that the region filter wasn't applied this time ("The region filter wasn't applied this time, so this list isn't limited to clubs in Europe."), show the list exactly as returned, never filter it yourself or by club names, and give no region confirmation in Step 6.
- `unknownClubRegionExcludedCount` counts players left out only because their club's region isn't recorded (free agents, or no league country). When it is more than 0, add one soft sentence after the list and give no number: "Some players whose club's region isn't recorded, such as free agents, aren't included." Never quote the count; it is taken over the whole player pool, not this list.

<!-- club-regions:start -->
<!-- Generated from lib/skills/scoutingRegions.ts by `bun run skills:render-regions`. Do not edit by hand. -->

> Source: GD-217 / Notion "Regions for Each Country", Football Manager 2024 (FM24) scouting regions. Pass region names to `filter.clubRegions` exactly as written in this table. A continent is not a row here: it goes in `filter.clubContinents`.

| Region | Countries (the club's current league country) |
|--------|-----------------------------------------------|
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
<!-- club-regions:end -->

### Value/age qualifiers: use the `filter`, don't enrich the candidate list

The `listSimilarPlayers` nodes are scalars-only (similarity score, no valuation/age), but that does **not** mean you must give up or enrich the whole list. Use the server-side `filter`:

1. **Absolute caps** ("under €25m", "under 23"): pass `filter.maxValuation` / `filter.maxAge` directly — one call, no enrichment.
2. **Relative caps** ("cheaper than Saka", "younger than De Bruyne"): call `getPlayerBioDataByPlayerId` on the **anchor only** (a single call) to read its valuation/age, then pass that number as `filter.maxValuation` / `filter.maxAge`.
3. Present the qualifier explicitly ("valued below Saka's €74M tag").

**Never** loop `getPlayerBioDataByPlayerId` over every returned candidate to filter by valuation/age — the `filter` does it server-side, and per-candidate enrichment is slow enough to **time the request out**. The only legitimate `getPlayerBioDataByPlayerId` calls here are the single anchor lookup for a relative threshold and the single lookup on Y for a similarity score between two named players (below). And **never** answer a "cheaper/younger" request with "valuation/age data isn't available" — pass the cap to `filter` instead.

## Step 5: Handle empty / null results

If `listSimilarPlayers` returns an empty array with a `filter` set, tell the user no similar players matched, naming every filter you applied, the region included ("No similar players at clubs in Europe valued under €29m came back."). Name the region as an applied filter only when it was applied, that is when `unknownClubRegionExcludedCount` is a number (see **A club region**); otherwise name the filters that were applied and say the region filter wasn't. Never say there is no similarity data when a filter was set; only an empty array with no `filter` means the anchor has none. Do not substitute an unfiltered list or fall back to SQL. Suggest broadening one constraint (e.g., "if I lift the cheaper-than-anchor restriction, I can show similar players at any price", or "if I widen Northern Europe to all of Europe, …"), then wait for confirmation before re-running.

## Step 6: Present the result

- Lead with the anchor: "Players similar to **Pedri** (Barcelona, 21yo, midfielder):" then list candidates.
- Include the candidate's club, age, and (when relevant to the user's filter) valuation in the row.
- Pass through whatever similarity score the tool returns — do not invent one and do not normalize it to a different scale.
- If the user named a constraint (cheaper, younger, in-league, a region), confirm it in one sentence: "All five are valued below Pedri's €100M tag" / "All from Premier League sides" / "All ten play for clubs in Europe." Confirm a region only when it was applied (see **A club region**).

## Similarity score between two named players

Use this path when the user names **two** players and asks how similar they are, e.g. "What is the similarity score of William Saliba and Ezri Konsa?" or "How similar are Saka and Madueke?". Call the first-named player X and the second Y.

The score belongs to the pair: it does not change with the filter, it is the same from X to Y as from Y to X, and it is the figure the Similar Players tab shows. But `listSimilarPlayers` returns at most 50 players, so Y is often not in X's unfiltered list. Narrow the candidate pool around Y until Y comes back:

1. Call `searchPlayers` for both players and verify each match as in Step 1.
2. Call `getPlayerBioDataByPlayerId` on Y (one call) to read Y's `age` and `minutesTotal`.
3. Call `listSimilarPlayers` with X's `playerId` and `filter`: `minAge: <Y's age>`, `maxAge: <Y's age + 1>`, `minMinutesPlayed: <Y's minutesTotal rounded down − 1>`, `maxMinutesPlayed: <Y's minutesTotal rounded up + 1>`. Each minimum must be strictly less than its maximum, or the call fails. Leave out a pair of fields when Y's bio has no value for it.
4. Look for Y's player id in the returned list (never match by name). If Y is not there, call again with only `minMinutesPlayed: <Y's minutesTotal rounded down − 50>`, `maxMinutesPlayed: <Y's minutesTotal rounded up + 50>`; if Y is still not there, call once with no `filter`.
5. When Y is returned, report its `similarityScore`. It is already a percentage (65.6 means 66%), not a 0–1 fraction: "Ezri Konsa is 66% similar to William Saliba." You may follow with a short profile comparison if it helps.
6. When none of those calls returns Y, say that the two players are not among each other's similar players, and offer a side-by-side profile comparison instead.

Never report a score unless `listSimilarPlayers` returned Y with it: never estimate, interpolate, or borrow another player's score, and never compute one yourself. Never say there is no similarity score — the product has one. Never `executeSqlQuery` for this.

## Common pitfalls

1. **Don't use `executeSqlQuery`** — there is no SQL fallback for similarity. If `listSimilarPlayers` doesn't satisfy the request, say so and stop; do not write SQL against player_vector or any similar table.
2. **Don't filter on the anchor's league/team unless asked** — "similar to Pedri" does not mean "must also be at Barcelona" or "must also be in La Liga". The anchor defines the target style, the candidate pool is global by default.
3. **Don't conflate with team-fit** — "find a player who fits Liverpool's style" is player-team-fit (a different intent). This skill is player-to-player similarity, not player-to-team fit.
4. **Don't skip the anchor lookup** — calling `listSimilarPlayers` with a player name instead of an ID will fail. Always `searchPlayers` first.
