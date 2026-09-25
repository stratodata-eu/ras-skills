---
name: ras-skills-docs
description: >-
  Lists the RAS plugin's currently available skills — what each does, how to invoke it, its default
  scope/filters, and how to override those filters in a prompt. Use when asked "what skills do you
  have", "what can this plugin do", "list RAS skills", "how do I use ras-analysis / ras-oos-cost", or
  when a request is ambiguous about which RAS skill applies. This skill does not query or analyse any
  data itself — it only describes the other skills, so it does not need a live RAS/BigQuery connector.
---

# RAS skills docs — what's available and how to use it

A quick reference for what this plugin can do, so a user doesn't have to read source files to find
out. This skill only describes capabilities; it never touches data itself, so it does not follow
`/ras:ras-analysis`'s query-grounding protocol — there is nothing to verify.

## Currently available skills

| Skill | Invoke as | What it does | Default scope | Override by saying so in your prompt |
|---|---|---|---|---|
| `ras-analysis` | `/ras:ras-analysis` | Evidence-grounded analysis protocol and workflow for any RAS / BigQuery data question or client-facing deliverable — every number traced to a query, every claim labelled verified / inferred / unknown. | `sales_channel = 'E-COM'`; no country filter (all countries combined). | Name another channel (e.g. B2B) to include it; name a specific country to filter/break out by it. |
| `ras-oos-cost` | `/ras:ras-oos-cost` | Estimates revenue lost to out-of-stock products, per calendar month — baseline daily order value from in-stock days only, applied to the days a product was actually out of stock. | Target period = last fully closed calendar month. Baseline window = trailing ~12 months of in-stock days (or full history if shorter). Inherits `ras-analysis`'s channel/country defaults. | Name a different month or date range (one loss figure is still produced per month); name specific EANs/products; name other channels or a country. |
| `ras-oos-risk` | `/ras:ras-oos-risk` | Flags best-selling products at risk of an upcoming stockout — ranks by recent sales velocity (in-stock days only), compares to current stock, projects days of stock remaining. | Velocity window = trailing 30 days. Best-sellers = top 20 EANs by units sold. Reorder point = 14 days of stock remaining. Inherits `ras-analysis`'s channel/country defaults. | Name a different window, best-seller cut (e.g. top 50), or reorder point (e.g. "flag under 21 days"); name other channels or a country. |
| `ras-scorecard` | `/ras:ras-scorecard` | Builds a client-facing scorecard artifact evaluating a defined set of goals/metrics against RAS data, then keeps it refreshed on a recurring cadence. Takes a prompt or an uploaded onepager as input. | Cadence = weekly (last fully closed week). Trend window = trailing 12 periods. Status bands: on track / at risk (within 10%) / off track / track-only. Inherits `ras-analysis`'s channel/country defaults. | Name a different cadence (daily/monthly/quarterly), trend window, or status bands per metric; supply your own targets/thresholds in the prompt or onepager. |
| `ras-skills-docs` | `/ras:ras-skills-docs` | This skill — lists what's available and how to use it. | — | — |

All filters are just plain language in your prompt — there is no separate config or flag system.
Say what you want changed from the default and the skill applies it.

## Example prompts

- *"Why did revenue drop in June?"* → `ras-analysis`, default scope (E-COM, all countries).
- *"Same, but B2B only in Slovakia."* → `ras-analysis`, channel and country both overridden.
- *"How much did stockouts cost us last month?"* → `ras-oos-cost`, default target month.
- *"What was the OOS cost for March, just for these 3 EANs?"* → `ras-oos-cost`, month and product
  scope both overridden.
- *"Which of our top sellers are about to run out of stock?"* → `ras-oos-risk`, default window/cut/
  reorder point.
- *"Same, but top 50 and flag anything under 21 days."* → `ras-oos-risk`, best-seller cut and reorder
  point both overridden.
- *"Build us a weekly scorecard tracking net sales growth and repeat purchase rate against these
  targets [onepager attached]."* → `ras-scorecard`, default weekly cadence.
- *"What can this plugin do?"* → this skill.

## Keeping this list current

This skill lists only what's currently under `skills/` — actively distributed to clients. Skills
moved to `archive/` (see `plugins/ras/README.md`) are intentionally left out; they're withdrawn from
use, not deleted. Whoever adds, removes, or archives a skill must update the table above in the same
change, and bump the plugin version (`plugins/ras/README.md` → "Adding a new skill").
