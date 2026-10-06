---
name: squad-wages
description: >
  Rank the user's own first-team players by annual wage, or give one named
  player's wage, exactly as the Squad Planning table's Wage/yr column shows
  them, respecting the viewer's access level.
---

# Squad Wages

Use this skill when the user asks who the **highest wage earners** are, who is **paid the most**, or for the **top N earners** on their own first team, the wage of **one named player**, or the highest paid players in **one position** ("highest paid defenders"). The answer comes from the same squad data the app's Squad Planning table shows (its Wage/yr column), with the same access rules.

Use a different intent when:

- The user wants players ranked by GPR, stats or scout reports → `rank_players`.
- The user asks about a **contract clause** (release clause, buyback, sell-on) → `lookup_contracts`.
- The user asks about the club's **Squad Cost Ratio**, a squad cost limit or the budget → `lookup_compliance`.

## Step 1: One call, with no arguments

If `getSquadPlayersByPosition` is not among your tools, say wage rankings aren't available yet and call no other tool (never SQL).

Call `getSquadPlayersByPosition` **once, with no arguments at all**. Never pass `scenarioId` or `positionIds` — the server refuses them. If a call fails with an error naming one of them, retry once with no arguments.

Never answer from earlier messages in the conversation or from memory: every name and amount in the answer comes from this turn's result. One tool call, then the answer. There is no other source for wages — never fall back to SQL or another tool.

The result holds `firstTeam[]`; each player has `name`, `position`, `age` and `wage`. Rank only the `firstTeam` players. Some results also carry count fields; they are used only in the last section, "Counts from the tool". Until then, write the answer as if there are none: Steps 3 and 4 alone give a complete answer.

## Step 2: Access refused

If the call itself is refused for access (a permission or role error), say that squad financials, including player wages, are not available at the user's access level. Never quote the error or say which role the user would need.

## Step 3: Rank the wages

The tool returns players in no particular order: the list is not sorted by wage. Never describe the order the tool returned as a ranking.

- A player whose `wage` is null is not ranked. Keep only players whose `wage` is a number. Sort them by `wage`, highest first.
- If the user names a number ("top 3", "five highest paid"), list exactly that many. Otherwise default to the top 10. Never list players beyond the N requested, including lower earners. If fewer players have a visible wage than that, list them all and say these are all the first-team players with a visible wage.
- Number the list by rank. Tied players share a rank number ("=4."), and the next rank counts the players above it; never number tied players one after another as if they were ranked. For each player give the name, the position in words ("centre-back", "right-back"), or omit it, and the annual wage.
- A player's rank is 1 + the number of players paid more than them; tied players share that rank, marked "=". Count the players above, never the distinct wages above. For example, after a 7-way tie:

  1. Player A — centre-back — €8.5M a year
  =2. Player B — right-back — €4.2M a year
  =2. Player C — left-back — €4.2M a year
  =2. Player D — centre-back — €4.2M a year
  =2. Player E — right-back — €4.2M a year
  =2. Player F — left-back — €4.2M a year
  =2. Player G — centre-back — €4.2M a year
  =2. Player H — left-back — €4.2M a year
  9. Player I — right-back — €1.8M a year

  Player I is 9., not 3. or 8.: eight players are paid more (1 + 8 = 9).
- Every ranked answer (a default top 10, a top N the user named, or a position list) says the list comes from the user's first team, in its opening sentence: "Here are the top 3 earners in your first team:". Never open with only "Your top 3 highest paid players are:".
- Never present a top N as the whole squad: call it "the top N of your first team".
- Players with the same wage are tied: say so, and never imply one out-earns the other.
- No numbers in prose: outside the numbered list lines, write no number except wage amounts and the N the user asked for. Never state how many players are tied, even among the players listed ("seven defenders tied" is a count): the shared rank number shows the tie. Never state a squad size, how many players have a wage, how many share the cut-off wage, or how many have no visible wage, and do not restate rank numbers in prose. Never state a total or count of players beyond the N listed; counts are left out by design.
- If any player beyond the N listed shares the cut-off wage, end the list with this fixed sentence, word for word (translated only into the user's language): "At least one more player outside this list also earns €X a year." Put no number or quantity word in it — no "40", "47", "more than", "many", "several" or "others" — however many players share the wage, and never change "one" to another number. Never name them. Never imply that a listed player out-earns a tied player who was left out. For example, a top 3 where more than three players share the top wage:

  =1. Player A — centre-back — €8.5M a year
  =1. Player B — central midfielder — €8.5M a year
  =1. Player C — centre-forward — €8.5M a year
  At least one more player outside this list also earns €8.5M a year.
- A missing wage is a `wage` field that is literally null in the result. A low wage is not a missing wage. If the result has null wages, say "Some first-team players have no visible wage" and name those players ("No visible wage for A and B"). Never mention, name or comment on players below the cut-off or beyond the N listed — not even to say their wage is missing. (A player whose `wage` is null, and the fixed tie sentence above, are the only exceptions.) Never state a number of null wages, and never claim a missing wage the result does not show. (When every wage is null, Step 6 applies instead.)

## Step 4: One position

When the user asks about one position or position group ("highest paid defenders", "top 3 earning strikers"), filter `firstTeam` on the returned `position` first, then rank with the same rules as Step 3, including ties, missing wages and no numbers in prose. Call the list "the top N defenders in your first team" (or the group asked for) and give no position totals.

If fewer players in that group have a visible wage than N, list them all and say these are all the <group, e.g. defenders> in your first team with a visible wage; never call it a top N. The answer's first line is then exactly this sentence (with the group asked for), and the ranked list follows it:

  These are all the defenders in your first team with a visible wage:

For a short list, never write "top N", "top 9" or "the N highest" anywhere in the answer, and give no number of players in the lead sentence. When every player in the group is listed, no tie sentence applies. For example, when the group runs out before N:

  These are all the defenders in your first team with a visible wage:

  1. Player A — centre-back — €8.5M a year
  =2. Player B — right-back — €4.2M a year
  =2. Player C — left-back — €4.2M a year
  4. Player D — centre-back — €1.8M a year

`position` is a side-specific code (the live codes include LCB, RCB, LDM, RCM, CAM and RCF). Group the codes with this map:

<!-- Backend twin: backend-v2 `src/mcp-server/execution/squad-wage-counts.ts`, `SQUAD_POSITION_GROUPS`, groups positions the same way to compute `positionGroupCounts` (SE-8854). This map, its role-letter fallback and that constant must change in lockstep, or the quoted group counts stop matching the groups listed here. -->

- Goalkeepers: GK
- Defenders: LB, RB, LCB, CB, RCB, LWB, RWB
- Midfielders: LDM, CDM, DM, RDM, LCM, CM, RCM, LAM, CAM, AM, RAM
- Wingers: LW, RW, LM, RM
- Forwards: LF, CF, RF, LCF, RCF, ST, SS

Match any other code by its role letters: a code ending in B = defender, one ending in M = midfielder, one ending in F or ST = forward.

Say which positions were included: never print position codes; describe the group in words ("centre-backs and full-backs"). A player whose `position` is null is not counted in any position group. If no first-team player has a matching position, say so and give no figure.

## Step 5: One named player

When the user names a player ("What does Saka earn?"), find them in `firstTeam` by name, ignoring accents and case; a surname or first name alone matches when only one first-team player has it. For a named player, do not rank the squad: answer "<name>'s wage is €X a year", using the name as the tool returned it.

- If the name matches more than one first-team player, list each match's name, position and wage, and ask which one the user meant.
- If the named player's `wage` is null, say their wage is not visible to the user (it may not be recorded, or the user's access level may not include squad financials). Give no figure.
- If the named player is not in `firstTeam`, say they are not in the first-team list, and give no figure. Never look for them in another tool, the academy or transfer targets, and never give a wage from memory.

## Step 6: Wages not visible

If every first-team `wage` is null, do not rank anyone. Say wages are not visible to the user: they may not be recorded, or the user's access level may not include squad financials. Never say "no wages are recorded" or that the squad has no data.

In either case, name no player and give no amount — whether the call was refused (Step 2) or every wage is null.

## Step 7: Amounts and wording

- `wage` is the **annual** gross salary, in euros. Label every figure as annual: "€5.2M a year".
- Never label a wage weekly or monthly, and never convert it (no dividing by 52 or 12). If the user asks for weekly or monthly wages, say the app records annual gross wages only, and give the annual figures.
- Write amounts in euros as the Squad Planning Wage/yr column shows them: from a million up, millions to at most two decimals with trailing zeros dropped ("€5.2M", "€10.44M"); below that, thousands to at most one decimal ("€750K", "€62.5K"). The result carries no currency code; the app shows wages in euros, so always use euros and never another currency symbol.
- Never show a figure the tool did not return.
- Describe a player only with `name`, `position`, `age` and `wage`. Never add a club, rating, nationality, contract detail or valuation from memory.
- Never show IDs, field names or the tool name to the user.

## Counts from the tool (only when the result has them)

Only if the result contains `wageTiers`, `visibleWageCount` and `firstTeamCount`; otherwise skip this whole section. If `firstTeamCount`, `visibleWageCount` or `wageTiers` is missing from the result, skip this whole section: the count-free wording of Steps 3 and 4 is the whole answer, and you state no count.

When all three are there, the result carries exact counts: `firstTeamCount` (first-team players), `visibleWageCount` (those with a visible wage), `wageTiers` (each distinct wage with the `count` of visible wages equal to it, highest first) and `positionGroupCounts` (one entry per position group with its `count`, `visibleWageCount` and `wageTiers`). They change Steps 3 and 4 only as below; every other rule there still applies. Never count players in `firstTeam` yourself: every count you state is a field the tool returned, copied as is. The only other number you may state is <listed>, how many of the players you listed earn the cut-off wage: <listed> is the one number you may work out from your own list, and it never exceeds N. If `visibleWageCount` is 0, no wage is visible: Step 6 applies.

- Call a top N "the top N of the M first-team players with a visible wage", where M is `visibleWageCount`.
- Only when `visibleWageCount` differs from `firstTeamCount`, add "<visibleWageCount> of your <firstTeamCount> first-team players have a visible wage." Never subtract one field from the other.
- In place of the fixed tie sentence, find the `wageTiers` entry whose `wage` equals the cut-off wage (the wage of the last player listed). If its `count` is more than the number of listed players on that wage, end the list with "<count> players earn €X a year; <listed> of them are shown above." <count> is that entry's `count`, copied as is; <listed> is how many of the players you listed earn €X. Never imply that a listed player out-earns a tied player who was left out, and never name the players not shown.
- With the count fields, report missing wages only through the visible-share sentence above, and name no one.
- Quote `positionGroupCounts` only when the request matches exactly one of the five groups (goalkeepers, defenders, midfielders, wingers or forwards); a narrower, wider or mixed request ("centre-backs", "full-backs", "central midfielders", "attackers", "defenders and midfielders") uses the count-free position wording of Step 4. Never add groups together. For a request matching one group, take the counts from the `positionGroupCounts` entry for that group: call the list "the top N of the <group `visibleWageCount`> defenders in your first team with a visible wage" (or the group asked for), and quote the cut-off tie from the group's `wageTiers` exactly as above.
