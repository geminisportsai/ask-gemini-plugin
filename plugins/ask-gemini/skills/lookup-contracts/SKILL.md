---
name: lookup-contracts
description: >
  Look up contract clauses or contract terms for a named player or for players
  on the user's roster — release/buyout clauses, buyback options, sell-on
  percentages, ROFR, no-trade, injury wage protection, commercial obligations.
---

# Lookup Contracts

Use this skill when the user wants to know what's in a player's **contract** — specific clauses, terms, conditions, or obligations. The result is one or more clause entries with names and bodies extracted from the player's stored contract document.

Use a different intent when:

- The user wants the contract **expiry date** or **months remaining** → that's surfaced by `summarize_player` / `filter_players` (bioData fields), not this skill.
- The user wants **transfer history** ("where has X played", "what did Arsenal sign") → `get_recent_transfers`.
- The user wants the actual **transfer fee** / **payment schedule** for a specific move → still this skill, but with `documentType: TRANSFER_AGREEMENT` (see Step 4 below).

## Step 1: Resolve the player to an ID — and verify it's the right one

Call `searchPlayers` with the player's name. **Then check the top result before using it.**

`searchPlayers` returns the first lexicographic match for a name fragment, which is often **not** the famous player the user means. Single-name queries are especially risky:

- "Saka" → frequently a German amateur, not Bukayo Saka.
- "Rice" → frequently a lower-tier player, not Declan Rice.
- "Bellingham" → could match Jude or Jobe; re-search with the full name when context implies one.

Re-search with the player's full name ("Bukayo Saka", "Declan Rice", "Jude Bellingham") when the top result's `currentClub` is inconsistent with where the user implies the player plays, or when `bioData.*GlobalCategoricalScore` fields are all `null`. If re-search still doesn't find a plausible match, **stop and ask the user to confirm the player** — do not call `getPlayerContractClauses` with the wrong player_id.

## Step 2: Identify the clause(s) the user named — synonym table

Users phrase clauses inconsistently. Map their phrasing to the canonical clause name **before** filtering the resolver's results:

| User phrasing | Canonical clause name (substring match against `name`) |
|---|---|
| "release clause", "buyout clause", "minimum fee release", "exit clause" | `Release Clause` / `Buyout Clause` |
| "buyback", "buyback option", "buy-back", "right to repurchase" | `Buyback Clause` / `Buyback Option` |
| "sell-on", "sell-on %", "sell-on percentage", "sell-on fee", "future-sale share" | `Sell-On Percentage` / `Sell-On Clause` |
| "ROFR", "first right of refusal", "right of first refusal", "matching right" | `First Right of Refusal` / `Right of First Refusal` |
| "no-trade", "no-transfer", "veto on transfer", "transfer veto" | `No-Trade Clause` / `No-Transfer Clause` |
| "injury clause", "injury wage protection", "wage protection on injury", "long-term injury reduction" | `Injury Wage Protection` |
| "commercial obligations", "image rights", "media days", "appearance obligations" | `Commercial Obligations` / `Image Rights` |
| "loyalty bonus", "signing-on fee", "signing bonus" | `Loyalty Bonus` / `Signing-On Fee` |
| "performance bonus", "appearance bonus", "goal bonus", "clean sheet bonus" | `Performance Bonus` / `Appearance Bonus` |

If the user names a clause not in this table, fetch the full list anyway (see Step 4) and surface any clause whose `name` is a plausible match — do not invent a canonical name and do not pass synonym strings as filter arguments to the resolver (it doesn't support that).

## Step 3: Pick the `documentType`

`getPlayerContractClauses` accepts an optional `documentType` argument:

| `documentType` | Meaning | When to use |
|---|---|---|
| `CONTRACT` (default) | The playing contract between the player and their current club. Contains wage terms, bonuses, release/buyout clauses, no-trade clauses, image rights, commercial obligations. | **Default — use this unless the user's phrasing clearly references the transfer fee.** |
| `TRANSFER_AGREEMENT` | The agreement between buying and selling clubs from a transfer. Contains the headline fee, payment schedule, add-ons, performance triggers, sell-on % owed to a former club, buyback rights for the selling club. | Use only when the user explicitly references **transfer-fee terms** — "the transfer fee terms", "the payment schedule for the transfer", "add-ons in the transfer agreement", "performance bonuses tied to the transfer", "what did Real Madrid agree to pay for X". |

When in doubt, default to `CONTRACT`. The two document types do not overlap — a buyback clause that lives in the transfer agreement (selling club's repurchase right) is a different clause from a release clause that lives in the playing contract (player-side exit).

## Step 4: Call `getPlayerContractClauses` and filter client-side

```
getPlayerContractClauses(playerId: <resolved-id>, documentType: CONTRACT)
getPlayerContractClauses(playerId: <resolved-id>, documentType: TRANSFER_AGREEMENT)
```

A player can have **two** contract documents — the playing `CONTRACT` and the `TRANSFER_AGREEMENT` — and a given clause may live in either (sell-on %, buyback, and ROFR frequently live in the `TRANSFER_AGREEMENT`, not the playing contract). Unless the user's phrasing clearly pins the answer to one document, **call the resolver for BOTH document types and merge the results**, then filter the combined clause list client-side. When the same canonical clause appears in both documents, surface both and label which document each came from.

The resolver returns the full list of clauses extracted from the document, each with `{ name, body, status, savedAt }`. **The parser uses canonical names**, so synonym strings ("buyout", "ROFR") are not valid filter arguments — there is no name-filter parameter on the resolver. Always fetch the full list, then filter client-side by checking each returned `name` against the canonical name(s) from your Step 2 synonym mapping (case-insensitive substring match is the safe pattern).

If the user named one clause, find the matching row and surface it. If the user asked for "all clauses" or didn't name a specific one, render the full list.

## Step 5: Empty result — verbatim halt

If `getPlayerContractClauses` returns an empty array (`[]`) for **both** `CONTRACT` and `TRANSFER_AGREEMENT`, the player has no extracted contract documents. **Stop immediately** and respond with this exact message (no preamble, no caveats, no fabricated clauses):

> No contract has been extracted for this player. If you expected to see one, contact your organization admin to confirm the document has been uploaded and processed.

Do not:

- Fabricate clause names or values
- Imply that "no clauses found" means the player has a clause-free contract (it almost always means the document hasn't been processed)
- Fall back to SQL or to other tools — there is no other source for clause data

Note: querying BOTH document types (Step 4) is the expected default, not a prohibited "silent switch." Only emit the verbatim halt above when **both** `CONTRACT` and `TRANSFER_AGREEMENT` return `[]`. If either document returns clauses, answer from the merged set.

## Step 6: "Not applicable" results are meaningful — surface them verbatim

When a clause exists with a body of `"Not applicable"`, `"N/A"`, `"None"`, or similar, that is a **deliberately negotiated absence** — the clause was discussed and excluded. Surface it explicitly:

> Bukayo Saka's contract has a "Release Clause" entry, but the body reads "Not applicable" — meaning a release clause was considered and deliberately excluded from the agreement.

Do **not**:

- Treat "Not applicable" as missing data and respond "I don't have that information"
- Hide the result because it's nominally empty
- Conflate "Not applicable" with the empty-array case from Step 5 (those are two different states with two different responses)

The distinction matters: "Not applicable" tells the user the clause was negotiated out (a positive datum); the empty array from Step 5 tells the user the document hasn't been processed (an unknown).

## Step 7: "My roster" scope — fan-out across team_ids' players

When the user phrases the question in terms of their own roster ("does anyone on my roster have a no-trade clause?", "anyone on my team…", "any of my players…", "which of my players have buyback clauses?"), the answer must come from checking the **whole roster**, not a single player. Resolving one player and answering from that alone is the exact defect to avoid — "does anyone on my roster have a no-trade clause?" requires checking **every** player on the roster, then aggregating. The answer must **name the specific players who satisfy the constraint** (and say "none" when none do) — do NOT answer generically (e.g., "some players may have…", "I can't tell without more info") when the roster can be enumerated, and do NOT answer from one player. Follow the standard My-Roster fan-out pattern:

1. Call `listMyOrganizationsTeams`.
2. **Empty-resolver halt** — if the result is empty, stop and respond with:

   > Your organization doesn't have any teams configured, so I can't identify your roster. Please add a team to your organization in the settings and try again.

   Do not broaden to "all players" and do not fall back to SQL.

3. **Enumerate the roster.** For each `team_id` returned, enumerate the team's players using whatever player-listing path the resolved tool set provides (a team-roster / team-players listing tool if present). Then fan out `getPlayerContractClauses` over **each** resolved player ID — not just the first — check each player's contract against the user's constraint, and **name the players who satisfy it** (and state when none do). **State how many players you checked** so the user knows the answer covered the whole roster (e.g. "I checked all 26 players on your roster; 2 have a no-trade clause: …"). If the user's prompt already lists specific player names ("does Rice or Saka have a no-trade clause?"), resolve each via `searchPlayers` and fan out over those IDs instead.
4. **If the resolved tool set genuinely lacks any roster-enumeration path** (only `listMyOrganizationsTeams` + name-keyed `searchPlayers`, no team-players listing), then list and check the roster players you CAN resolve, name who satisfies the constraint among them, and explicitly state the limitation — do not answer generically. Use:

   > Based on the players I can resolve on your roster, the following have a no-trade clause: [names]. Note: I may not be able to enumerate every player on your roster in one shot — if you tell me which players to check (e.g., "check Rice, Saka, and Ødegaard"), I can confirm the rest.

   Do not fabricate a roster list. Do not partially answer based on the few players you can resolve without explicitly stating that scope.

## Step 8: Present the result

For a single clause, lead with the player and clause name, then the body verbatim:

> **Bukayo Saka — Release Clause**: "Triggered if Premier League final position is below 8th in any season; £75m fixed amount, payable in two equal instalments." (Saved 2024-08-12)

For multiple clauses on one player, render a table:

| Clause | Body | Status |
|---|---|---|
| Release Clause | £75m, conditional on... | Active |
| Buyback Clause | Not applicable | — |
| Sell-On Percentage | 15% to Hale End (former training club) | Active |

For a fan-out across multiple players (Step 7), group by player:

> **Bukayo Saka**
> - Release Clause: £75m...
> - Buyback Clause: Not applicable
>
> **Declan Rice**
> - Release Clause: None recorded
> - Sell-On Percentage: 10% to West Ham

Pass through `body` text verbatim — these are extracted from real contract documents, not paraphrased. Don't summarise, don't soften, don't translate "Not applicable" into "no data".

Never mention `documentType` enum values, GraphQL field names, or the resolver name in the user-facing answer.

## Common pitfalls

1. **Don't pass synonym strings as resolver args.** `getPlayerContractClauses` doesn't accept a clause-name filter. Always fetch the full list and filter client-side via the synonym table.
2. **Don't treat empty results as zero clauses.** Empty array = document not extracted = the verbatim Step 5 response. Be honest about the difference.
3. **Don't conflate "Not applicable" with missing data.** A clause body of "Not applicable" is meaningful — surface it explicitly.
4. **Always check both documents before halting.** Query both `CONTRACT` and `TRANSFER_AGREEMENT` and merge; only emit the empty-result halt when BOTH are empty. A clause absent from the playing contract (e.g. sell-on %, buyback) is frequently present in the transfer agreement.
5. **Don't use `executeSqlQuery`.** No SQL fallback for clause data.
6. **Don't fabricate clause names.** If the user names a clause not in the synonym table and the returned list has no match, say so plainly — don't synthesize a plausible-sounding clause body.
