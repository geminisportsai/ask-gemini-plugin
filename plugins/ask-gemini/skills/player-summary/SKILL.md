---
name: player-summary
description: >
  Build a comprehensive player profile or scouting verdict: bio data, GPR with
  squad-role projection, fit scores, valuation range, contract status, pipeline
  status, scout reports, and internal notes. Use when the user asks about a
  specific player's details, overview, or what the organization knows/thinks
  about a player.
---

# Player Summary

Use this skill when the user asks about a specific player's profile, details, stats overview, performance history, or for a scouting verdict on a player.

There are two response shapes, chosen by what the user asked:

- **General profile** ("tell me about X", "full profile on X") — data-led: bio, GPR, fit scores, valuation, contract, recent matches, plus a brief scout/notes flavor when they exist. Steps 1–6.
- **Scouting verdict** ("what do my scouts think of X", "scouting verdict on X", "should we pursue X", "what do our notes say about X") — intelligence-led: scout reports and internal notes are the primary sources. Step 7.

## Step 1: Resolve the player to an ID

Every prompt for this intent names a player. Call `searchPlayers` with the player's name first.

### Wrong-player re-search rule

Common surnames ("Saka", "Haaland", "Rodri", "Mbappe", "Bellingham") often return obscure homonyms (lower-division or amateur players). Before proceeding:

- Prefer the candidate whose `clubName`/`gpr` matches a recognizable profile; check `doesBelongToCurrentUsersOrganization` where relevant.
- If bioData comes back all-null, you likely have the wrong player — re-check the search results before answering.
- Note that prominent players are often stored mononymously — pick by club/GPR among namesakes rather than inventing a fuller name to re-search.
- If still ambiguous, ask the user to clarify (name + club). Do not guess.

**The summary MUST be about the resolved player and their actual current club. Never substitute, blend in, or fall back to your general knowledge of a similarly-named player — use ONLY what the tools return.**

## Step 2: Get bio and performance data

Call `getPlayerBioDataByPlayerId` with the player's ID. This single call carries most of the profile:

- Identity: age, height, nationality, `currentClub`, `generalPosition`
- `gpr` — Gemini Player Rating (see framing rules in Step 6)
- `teamStyleFit` — how well the player fits **your team's** tactics (not their current team's)
- `fitScore` — how well the player matches **your defined archetype** for the role
- Valuation: `playerValuation` (public), `minGeminiPlayerValuation` / `maxGeminiPlayerValuation` (Gemini's fair-price range; fall back to `fairFee` / `expectedFee` if the Gemini bounds are null)
- `contractExpires` — express as months remaining (or "contract expired" when past)
- `isInjured` — mention only when true

Also call `getPlayer` if you need the full player record (positions list, team relation), and `getPlayerDataSummary` for the aggregated performance view.

## Step 3: Get seasonal scores and match history (general profile)

- `getPlayerSeasonCategoricalScores` — season-level categorical scores (attributes) showing how the profile trends across seasons and skill areas.
- `listPlayerMatches` — recent appearances: playing time, results, competition level.

## Step 4: Read scout reports (required for any scout-opinion phrasing)

The organization's scouts write their own reports. When the question is about **scout opinion** — "what do my scouts think of X", "how did we scout X" — those reports are the **primary source**, not bio data or categorical scores.

- Call `organizationScoutReports(filter: { search: "<player name>" })`. Use the reports' ratings (offensive/defensive/athleticism/game-intelligence), notes, and recommendations.
- If `totalCount` is 0, say plainly that your scouts have not written any reports on this player yet — then optionally offer the data-driven profile.
- **Never fabricate scout opinion.** Do not relabel bio data, GPR, or categorical scores as "Scout Assessment" / "our scouts rated…" when no report was read. Scout language is only warranted when backed by an actual report.

For a general profile request, reading scout reports is optional enrichment; include a brief scout note only if reports exist.

## Step 5: Read internal notes and pipeline status

**Notes** — call `listNotesByPlayerId` with the player's ID. Notes are the organization's second intelligence source alongside scout reports: transfer intelligence (availability, release-clause chatter, agent conversations) often lives ONLY in notes.

- Notes and scout reports are different sources — never present one as the other.
- An empty result means no visible notes for this player — say so when the user asked about notes; skip silently in a general profile.
- Results are permission-scoped to what this user may see. Never try to read notes any other way.

**Pipeline status** — optional enrichment for a general profile. Call `listMyKanbanBoards`, then `listKanbanCards` for the relevant board(s), and look for the player's card. If found, report the status as "<phase> (<board>)". If the org has no boards or the player has no card, omit status entirely — do not mention its absence. Treat both calls as best-effort: on timeout, permission error, or any backend failure, silently omit pipeline status and continue with the rest of the summary — never abort an otherwise valid player summary because of a kanban failure.

## Step 6: Present the general profile

Structure:

1. **Header**: name, age, nationality, current club, position(s).
2. **Squad-role projection (REQUIRED)**: state the player's projected squad role based on GPR: <40 = reserve depth, 40–<60 = backup, 60–<75 = competing for a starting spot, ≥75 = likely starter. Phrase it concretely ("would likely be a backup option at right wing"). This is non-negotiable — include it in every general profile.
3. **Performance overview**: GPR, Team Fit, Player Fit, and standout categorical scores.
4. **Valuation & contract**: public valuation plus Gemini's lower–upper fair-price range; contract as months remaining. Include pipeline status and injury status when present.
5. **Season trends & recent matches**: improvement/decline across seasons; recent appearances with competition and minutes.
6. **Insights**: career trajectory, strengths, or concerns the data highlights — including a brief scout/notes flavor when reports or notes exist.

### GPR framing rules (verbatim semantics)

- GPR stands for "Gemini Player Rating" — never "General Performance Rating" or "Gemini Performance Rating".
- Scale: 0–100 where 50 = average player in the player's league. Under 40 = below league average; 40–<75 = average to above average; ≥75 = good quality.
- Report the single GPR value only — never mention "Time Decay GPR", "TIME_DECAYED_GPR", or "Seasonal GPM".
- **Honesty rule**: when GPR is below 40, use honest language like "limited" or "below standards". Do NOT use "solid", "promising", or "intriguing" for a sub-40 player.

### Formatting rules

- Never mention database table names, column names, SQL queries, joins, or any data-retrieval mechanics.
- Human-friendly metric names only: "GPR" not "TIME_DECAYED_GPR", "Fit Score" not "FIT_SCORE", "Valuation" not "PLAYER_VALUATION".
- Team Fit measures fit to **your team's** tactics; Player Fit measures match to **your defined archetype**. Never describe either as fit to the player's current club.

## Step 7: Present the scouting verdict

When the user asks for a verdict / scout opinion / "what do we know about X" synthesis:

- Base it **ONLY** on the scout reports (Step 4) and notes (Step 5). Do not pad with bio data or categorical scores, and never use general knowledge about the player.
- Write a **single cohesive prose paragraph, 3–5 sentences** — no bullet points, numbered lists, or section headers. (Exception: the no-source fallback below is exempt from this length — use its exact two-sentence wording.)
- Cover: key strengths, weaknesses, concerns, and any transfer intelligence found in the notes (release clauses, availability, agent conversations). Any release-clause mention must be attributed as unverified note chatter (e.g. "a note mentions a possible release clause of...") — never present it as a confirmed contractual term; authoritative clause amounts come only from the lookup-contracts skill.
- If multiple scouts reported, weave in where they agree or disagree.
- If there are no scout reports but notes exist: summarize the notes and mention that no formal scouting reports have been submitted yet.
- If there are neither reports nor notes, say so plainly:
  > Your organization has no scout reports or notes on <player> yet, so I can't give you a scouting verdict. I can put together a data-driven profile instead if you'd like.
  Do NOT call further tools to fill the gap, and do NOT substitute bio-data conclusions dressed as scout opinion.

## Common pitfalls

1. Don't use `executeSqlQuery` for this intent — scout reports and notes are permission-scoped; raw SQL over-counts and leaks what the user can't see.
2. Don't skip player-name resolution or squad-role projection.
3. Don't soften a sub-40 GPR ("solid", "promising") — the honesty rule exists so verdicts stay trustworthy.
4. Don't collapse "no notes" and "no reports" into each other — they are separate sources with separate empty states.
5. Don't answer contract *clause* questions here (release clause amounts, buyback options) — that's the lookup-contracts skill; this skill only reports contract expiry timing. Note-sourced release-clause chatter may still be mentioned in a scouting verdict, but only as attributed, unverified note content — never as a confirmed contractual term.

Offer follow-up actions such as comparing the player with others (via `sql-analytics`) or adding them to a watchlist (via `watchlist-management`).
