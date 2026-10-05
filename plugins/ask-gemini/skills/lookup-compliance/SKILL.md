---
name: lookup-compliance
description: >
  Read the user's own club's Squad Cost Ratio and its approved-budget
  objective (Total Squad Cost) exactly as the Compliance tab shows them,
  respecting the viewer's compliance access level.
---

# Lookup Compliance

Use this skill when the user asks about their club's **Squad Cost Ratio**, whether the club is within a **squad cost limit** (UEFA, Premier League), or whether the club is **on budget** and by how much. The answer comes from the same compliance picture the app's Compliance tab (Squad → Compliance) shows, with the same access rules.

Use a different intent when:

- The user wants players **ranked or listed by wage** → not this skill; it reads club-level figures only and returns no per-player wage.
- The user asks about a **contract clause** (release clause, buyback, sell-on) → `lookup_contracts`.

## Step 1: One call, with no arguments

Call `complianceOverview` **once, with no arguments at all**. It reads the viewer's own club, the actual figures (no scenario), as of now. Never pass `organizationId`, `scenarioId`, `asOf`, `draftActions` or `draftWageOverrides` — the server refuses them. If a call fails with an error naming one of them, retry once with no arguments.

Never answer from earlier messages in the conversation or from memory: every figure in the answer comes from this turn's `complianceOverview` result. One tool call, then the answer. There is no other source for these figures — never fall back to SQL or another tool.

If the call itself is refused for access (a permission or role error), say the club's compliance figures are not available at the user's access level, and give no figure.

## Step 2: Find the rule

The result has `viewerTier`, `permittedQuantities` and `layers[]`; each layer has `layer` (INTERNAL = the club's own objectives, CONTINENTAL, DOMESTIC) and `evaluations[]`.

- **Squad Cost Ratio:** every evaluation whose `ruleFamily` is `SQUAD_COST_CONTROL`. There is one per regime layer (for example UEFA Club Licensing in CONTINENTAL, Premier League in DOMESTIC); `regimeName` names the regime and `ruleName` may differ from layer to layer ("Squad Cost Ratio", "Premier League Squad Cost Ratio").
- **The club's own internal squad cost limit:** an INTERNAL-layer evaluation whose `metricName` is "Squad Cost Ratio", whatever its `ruleFamily` (a club objective often has none), is the club's own limit on the same ratio, usually lower than the regime caps. You may mention it, after the regime layers, as "the club's own internal squad cost limit" with its limit and state. Never quote its `ruleName` (it can carry internal labels). Mention it as a ratio limit, never as a budget.
- **Budget:** the INTERNAL-layer evaluation whose `metricName` is "Total Squad Cost". Its `threshold` is the approved budget and its `value` is the actual total squad cost.

## Step 3: Read the rule

- `state` is the verdict, already decided by the server: `GREEN` = within the limit; `RED` = over the limit (a breach). Take good or bad from `state`, never from the sign of `deviation` alone.
- `AMBER` is a warning, not a position: usually past `amberThreshold` and under the cap, but on a warn-only rule `AMBER` may already be over the cap. So for `AMBER` read the sign of `deviation` against `comparator` and say "over the cap" or "under the cap, close to it" from that, never "close to the limit" by default.
- `comparator` says how `value` must relate to `threshold`: `LTE`/`LT` is a cap (at or below), `GTE`/`GT` a floor (at or above).
- `deviation` is the signed `value - threshold`, in the rule's `unit` (for `PERCENTAGE`, percentage points). Read its sign against `comparator`: for a cap, a negative deviation is the headroom under the limit and a positive one is the amount over it.
- When `completeness` is `INCOMPLETE`, `value` is a lower bound: some inputs are not yet included, so the true figure can only be higher. Put "at least" before the figure and call it provisional.
- `INCOMPLETE` and the state is `RED` on a cap: the lower bound is already past the cap, so this is a confirmed breach whose true size is at least the figure. Say it is over the cap, for example "over the cap (at least 74.2%, provisional figure)", and give the points over as "at least". Do not withhold the breach.
- `INCOMPLETE` and the state is `AMBER` or `GREEN` on a cap: the figure is under the cap so far, not confirmed, because the missing inputs could push it over. Never call it within the limit as a final answer.
- If `missingInputs` is empty, say only that some inputs are not yet included, and never name specific missing inputs. When it has entries, name every one.
- `PARTIAL` state means inputs are missing: give no figure and no verdict (not within, not over); say it is incomplete and name every entry in `missingInputs`.
- `NOT_CONFIGURED` state means there is no figure yet: say no figure is available yet and name every entry in `missingInputs`. Never report it as 0% or 0.

## Step 4: Answer the Squad Cost Ratio

Lead with the CONTINENTAL and DOMESTIC layers. For each layer with a Squad Cost Ratio rule, give:

- the metric name "Squad Cost Ratio" and the regime (`regimeName`);
- the ratio (`value`) as a percentage;
- the limit (`threshold`) as a percentage;
- the state, in words (within the limit, over the limit, or for `AMBER` over or under the cap as the deviation shows);
- how far from the limit, in percentage points, from `deviation` (for example "+4.2 pts over the 70% cap" or "3.2 pts under the 70% cap").

Never call the Squad Cost Ratio a budget.

## Step 5: Answer the budget

If the Total Squad Cost objective exists, give the actual total squad cost (`value`), the approved budget (`threshold`), the state, and the signed difference from `deviation` ("€4M over budget" or "€2.5M under budget"). If the user asks for a percentage of budget, compute it as `value` ÷ `threshold` and show both numbers it came from.

If there is no Total Squad Cost objective and the viewer's access permits it (Step 6), say that no approved-budget objective is set up for the club, so there is no budget to compare against, and that one is set up in the app under Squad → Compliance. Never invent a budget, never estimate one, and never substitute another rule (such as the Squad Cost Ratio) for a budget. You may offer the Squad Cost Ratio instead.

## Step 6: Access levels — gated is not "not configured"

A rule the viewer's role may not see is left out of the result entirely; an absent rule can mean **gated**, not unconfigured. A restricted viewer gets no error: often the result has no layers at all and a `ruleCount` of 0, with `permittedQuantities` holding only non-financial quantities. That is a gated result, not an empty club. Decide which before answering:

- The Squad Cost Ratio needs `ADJUSTED_REVENUE` and `PLAYER_WAGE_ANNUAL` in `permittedQuantities` (club owners see it).
- The Total Squad Cost objective needs `PLAYER_WAGE_ANNUAL` in `permittedQuantities`.

If the rule is absent and any quantity it needs is missing from `permittedQuantities`, say that this figure is not available at the user's access level. Give no number, no threshold and no state, and never say it is "not configured", that there are "no rules", "0%", "zero" or that there is "no data". Never suggest the figure exists or hint at its value. You may say that club owners can see it.

Only when every quantity the rule needs is in `permittedQuantities` and the rule is still absent, say it is not configured for the club.

## Step 7: Amounts and wording

- Amounts (`unit` `CURRENCY`) are in euros, in the app's short form, rounded to whole millions from €1M up: "€13M", "€400K". The result carries no currency code; the app shows these amounts in euros, so always use euros and never another currency symbol.
- Round percentages to one decimal place ("74.2%", not "74.18%"), and points the same way ("+4.2 pts").
- Use only fields the tool returned. Never state revenue, wages, per-player figures, drivers or a computed fix — they are not in this result.
- Never show IDs, field names, enum values or the tool name to the user.
